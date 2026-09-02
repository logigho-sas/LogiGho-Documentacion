## Autor: Iker Acevedo

Fecha creacion: 2026-08-26

Estado: produccion

## Infraestructura: Estrategias y clientes de transportadora

**Ubicación:** `Infraestructura/Estrategias/`, `Infraestructura/Servicios/`

---

## Arquitectura: Strategy + Factory, sin switch por transportadora

```
IEstrategiaDevolucion (interfaz)
  Transportadora: string
  PuedeResolver(guia): bool
  ResolverLoteAsync(guias): ResultadoResolucion

EstrategiaConsultaExterna implements IEstrategiaDevolucion
  — UNA sola clase para cualquier transportadora que exponga consulta por guia.
  — recibe un IClienteTransportadora inyectado (ClienteInter o ClienteEnvia)
  — la orquestacion (preguntar -> extraer original -> buscar en PedidosInter -> armar resultado)
    es identica sin importar la transportadora

FactoryEstrategiaDevolucion
  — reparte el lote preguntandole a cada estrategia si "PuedeResolver" el formato
  — sin switch: agregar una transportadora es registrar una clase mas (Open/Closed)
  — EstrategiaSinResolucion es el respaldo: nunca se ofrece voluntaria, solo la
    usa el factory cuando NINGUNA estrategia reconocio el formato
```

Registrar una transportadora nueva mañana: implementar `IClienteTransportadora`, agregar una línea al arreglo `estrategias` en `WorkerHandler.Construir()`. Ni `EstrategiaConsultaExterna`, ni `FactoryEstrategiaDevolucion`, ni `ProcesarJobUseCase` cambian.

---

## `EstrategiaConsultaExterna` — la orquestación compartida

```
ResolverLoteAsync(guias)
  1. cliente.ConsultarLoteAsync(guias)          UNA tanda contra la API de la transportadora
  2. originales = consultas con GuiaOriginal, sin repetir
  3. repositorio.BuscarPorNumeropreenvioAsync(originales)   UNA consulta a Mongo, no una por guia
  4. Interpretar(consulta, pedidos) por cada una:

       consulta.Fallo?                    -> Rechazada, ErrorConsultaExterna
       !consulta.TieneGuiaOriginal?        -> Rechazada, SinGuiaOriginalEnRespuesta
       pedidos no tiene GuiaOriginal?       -> Rechazada, OriginalNoExisteEnSistema  (HUERFANA)
       si no                                -> ResueltaInter, con todos los datos del pedido
```

La guía huérfana (`OriginalNoExisteEnSistema`) se logea con `Log.Warn(_ctx, "guia-huerfana", ...)` y conserva `GuiaOriginalExtraida` — es el número que hay que reclamarle a la transportadora, porque nos devolvió un paquete que nunca despachamos.

`Rechazo(...)` marca `Transportadora = Transportadora` (la de la propia estrategia) en cada guía rechazada — necesario desde [ADR-005](../adr/ADR-005-confiabilidad-inter.md) para que `ProcesarJobUseCase.ReintentarAsync` sepa cuál transportadora ya se probó y no la vuelva a intentar al buscar alternativas.

---

## `ClienteInter` — Interrapidísimo

Necesita **dos llamadas por guía**, y el orden importa:

```
1. Rastreo(guia de devolucion)   -> "DiceContener" trae el numero original en texto libre
2. Estados(guia ORIGINAL)         -> la fecha del estado de devolucion
```

El paso 2 se consulta **sobre la guía original**, no sobre la escaneada — verificado contra datos reales, la guía de devolución nunca tiene ningún estado con "Devoluci" (Inter la marca "Entregada", porque efectivamente se entregó en nuestra bodega). El estado de devolución vive en la guía inicial, como `"Devolución ratificada"`.

```
"Devolución ratificada"  cuenta como devolucion
"Devolucion Regional"    NO cuenta — es una devolucion entre sedes de Inter,
                          no significa que la mercancia volvio a nuestra bodega
```

**Formato de guía Inter:** 13 dígitos, empieza en `"30"`.

**Token cacheado por contenedor** (`_token` estático, `_tokenVence`): el módulo legacy pedía token en **cada** invocación — con 1.000 guías eran 1.000 autenticaciones innecesarias. Acá se pide una vez y se reusa, con un `SemaphoreSlim` que evita que varias tareas concurrentes pidan tokens a la vez en el primer uso tras un cold start (double-check locking).

**Vigencia real del token: 20 segundos, no 20 minutos** (ver [ADR-005](../adr/ADR-005-confiabilidad-inter.md)). El token que devuelve `GenerarTokenTemporal` es un JWT — decodificando su payload (`exp - iat`) se confirmó que Inter lo vence a los **30 segundos reales**, no a los 20 minutos que asumía el código original. `_tokenVence = DateTime.UtcNow.AddSeconds(20)` deja 10s de colchón por latencia. Si aun así Inter lo invalida antes, el fallo llega como HTTP 401 y el mecanismo de reintentos (ver abajo) lo detecta y pide uno nuevo.

**Concurrencia y pacing:** `CONCURRENCIA_INTER` (por defecto **5**, antes 10 — se bajó porque 10 en paralelo disparaba 429 de Inter) — el rastreo es una llamada por guía, en **tandas** de este tamaño. Entre tanda y tanda hay una pausa (`PAUSA_ENTRE_PETICIONES_INTER_MS`, por defecto 300ms): antes solo se topaba cuántas iban a la vez con un `SemaphoreSlim` sin pausa, y con 200+ guías eso era tráfico sostenido sin cortes que seguía gatillando el 429 aunque el pico simultáneo fuera chico. Los estados de las originales sí aceptan varias por llamada, en lotes de 15 (`TamanoLoteMaximo`), con la misma pausa entre lote y lote.

**Reintentos con backoff (`EnviarConReintentosAsync`):** hasta `MaxIntentos = 4` por petición HTTP. Reintenta ante `429 Too Many Requests` y `5xx`, respetando el header `Retry-After` de Inter si lo manda; si no, backoff exponencial con jitter (~500ms → 1s → 2s, tope 4s) para que las tareas en paralelo no reintenten todas en el mismo instante. Cada intento fallido se loguea con `statusCode`, número de intento y hasta 500 caracteres del cuerpo de la respuesta (`inter-rastreo-http-no-ok` / `inter-estados-http-no-ok`) — antes de este fix, una respuesta no-2xx se descartaba en silencio, sin ningún rastro en logs, lo que hacía imposible diagnosticar por qué un lote entero terminaba rechazado.

---

## `ClienteEnvia`

Su API expone la guía en la **URL** (`GET .../ConsultaGuia/{guia}`), no admite lote — una llamada por guía, compensado con concurrencia controlada (`CONCURRENCIA_ENVIA`, por defecto 5).

**Formato de guía Envía:** 12 dígitos (`ConFormatoEnvia` rellena con ceros a la izquierda si hace falta).

**La guía original viene embebida en texto libre**, campo `anotaciones`:

```csharp
private static readonly Regex GuiaOriginal = new(@"\b\d{12}\b", RegexOptions.Compiled);
```

Un único número de 12 dígitos por anotación, verificado sobre 124 muestras reales. El `\b` (límite de palabra) evita capturar los dígitos de un número más largo por accidente.

Si no hay match, se devuelve `ConsultaGuiaResultado.SinDevolucion(guia)` — **no** es un fallo técnico (`Fallo=false`), es la transportadora respondiendo correctamente pero sin dato útil, y eso es lo que en `EstrategiaConsultaExterna.Interpretar` cae en `SinGuiaOriginalEnRespuesta` (reintentable con otra transportadora), no en `ErrorConsultaExterna`.

**Fecha normalizada al mismo formato que Inter**, para que `"Fecha Devolucion"` quede homogéneo en `PedidosInter` sin importar la transportadora — Envía entrega fecha (`dd/MM/yyyy`) y hora por separado, se combinan y parsean a `yyyy-MM-ddTHH:mm:ss.fff`.

---

## Tabla comparativa

| | ClienteInter | ClienteEnvia |
| - | ------------ | ------------ |
| Formato de guía | 13 dígitos, empieza en `30` | 12 dígitos |
| Llamadas por guía | 2 (rastreo + estados) | 1 |
| Soporta lote | Estados sí (15), rastreo no | No |
| Concurrencia por defecto | 5 (antes 10) | 5 |
| Pacing entre tandas | 300ms (`PAUSA_ENTRE_PETICIONES_INTER_MS`) | No tiene — nunca mostró el problema de 429 |
| Reintentos HTTP (429/5xx) | Sí, 4 intentos con backoff | No tiene todavía |
| Origen de la guía original | Regex sobre `DiceContener` | Regex sobre `anotaciones` |
| Autenticación | Token temporal cacheado 20s (vence real a los 30s) | Ninguna (URL pública con la guía) |

---

## Reintento cruzado de transportadora — cuándo y cómo

Desde [ADR-005](../adr/ADR-005-confiabilidad-inter.md), `GuiaProcesada.MereceReintentoConOtraTransportadora` dispara el reintento en `ProcesarJobUseCase.ReintentarAsync` para **tres** motivos, no solo uno:

```
SinGuiaOriginalEnRespuesta   -- la transportadora respondio 200 pero sin dato util en el texto
ErrorConsultaExterna         -- la transportadora rechazo la peticion (ej. 400 "no la tengo")
GuiaNoExisteEnSistema        -- ninguna transportadora reconocio el FORMATO de la guia
```

El tercer caso es el que resuelve `EstrategiaSinResolucion`: antes, una guía que no calzaba con el formato de ninguna transportadora (13+"30" para Inter, 12 dígitos para Envía) se rechazaba sin llamar a ninguna API. Ahora, como `EstrategiaSinResolucion` también marca este motivo como reintentable, esa guía entra al mismo mecanismo de `ReintentarAsync` y se prueba contra **todas** las transportadoras registradas (empezando por Inter, por ser la de mayor volumen) antes de darla por perdida — cubre guías reales cuyo formato no coincide con la heurística de dígitos que asumimos (ej. una guía Inter de 12 dígitos, en vez de los 13 usuales).

`ReintentarAsync` prueba cada alternativa **una sola vez**, sin backoff ni reintento múltiple — eso es aparte, ya lo cubre `MaxIntentos` dentro de cada cliente.

---

## `ServicioInventario`

`POST /actualizaInventarioLotes` — **un solo POST con todo el lote**, en vez de una llamada por guía. El trabajo pesado del lado de esa lambda es fijo por petición, no proporcional al lote: agrupar más guías no lo encarece.

`MaximoGuiasPorPeticion = 500` — se parte en trozos para no acercarse al límite de 15 minutos de esa lambda.

`INVENTARIO_SIMULADO=true` registra en log lo que habría enviado sin llamar al servicio real — la única forma segura de probar el flujo completo sin mover inventario real.

Un fallo acá **no tumba el job** — la devolución ya quedó registrada en `InventarioDevolucion` y el pedido sigue siendo elegible en una corrida posterior de `ApiLambdaActualizacionInventario`, que filtra por su propia marca de procesado.
