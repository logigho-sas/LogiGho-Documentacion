## Autor: Iker Acevedo

Fecha creacion: 2026-08-26

Estado: produccion

## Lambda: ApiLambdaDevolucionesMasivo

**Accionador:** 3 funciones dentro del mismo proyecto — 2 detrás de API Gateway, 1 invocada solo internamente

**AOT:** No (runtime `dotnet10` managed)

---

## ¿Qué hace?

Reemplaza al módulo legacy de devoluciones masivas, que causaba picos de 99% de CPU en DocumentDB al procesar archivos grandes desde el navegador con lógica dispersa y sin control de lote. Este módulo hace lo mismo — pasar guías de devolución a "Devolución Completada" en `PedidosInter` — pero **de forma asíncrona, en el servidor, por lotes**, con reintentos, idempotencia y un estado consultable en todo momento.

El operario sube un archivo (o pistolea guías una por una) desde el front [`ingreso-devoluciones`](../../../../../frontend/views/logistica/ingreso-devoluciones/ingreso-devoluciones.md). Por cada guía, el sistema intenta en orden: idempotencia (¿ya se procesó?) → match directo en `PedidosInter` → consulta a la API de la transportadora (Inter/Envía) para extraer la guía original de un texto libre → validaciones de negocio (tienda, pedido anulado). Ver [`ProcesarJobUseCase`](casos-uso/procesar-job-usecase.md) para el detalle de las 4 fases.

---

## Por qué 3 funciones en un solo proyecto, y no 3 lambdas separadas

Las tres funciones (`IniciarHandler`, `WorkerHandler`, `EstadoHandler`) comparten el mismo dominio, los mismos modelos y los mismos repositorios — dividirlas en proyectos separados obligaría a triplicar (o publicar como paquete NuGet interno) cada clase de `Dominio/` e `Infraestructura/`. Un mismo proyecto con 3 `function-handler` distintos en cada `aws-lambda-*.json` da 3 despliegues independientes (memoria, timeout y rol propios) sin pagar el costo de mantenimiento de 3 repos o 3 proyectos sincronizados a mano.

Cada función igual se despliega, escala y factura **por separado** — no es un monolito en ejecución, es un monorepo de handlers.

## Por qué el Worker no es una API HTTP síncrona

Procesar 3.000 guías contra Mongo y contra 2 APIs de transportadoras externas puede tardar varios minutos. API Gateway corta cualquier integración a los **30 segundos**. Si el Worker fuera un endpoint HTTP, el operario tendría que mantener la conexión abierta ese tiempo (imposible desde un navegador) o el gateway cortaría la respuesta a mitad de proceso sin forma de saber si terminó.

La solución: `IniciarHandler` crea el job y devuelve en ~200ms (dentro del límite del gateway), y dispara al `WorkerHandler` con `InvocationType.Event` — una invocación asíncrona de Lambda a Lambda, que **no** pasa por API Gateway y por lo tanto no tiene el techo de 30 segundos. El Worker puede correr hasta 15 minutos (el máximo de una Lambda), y el front se entera del avance consultando `EstadoHandler` cada 2.5 segundos (polling), no esperando una respuesta HTTP.

---

## Las 3 funciones

| Función | Ruta | Expuesta en API Gateway | Timeout | Memoria |
| ------- | ---- | ------------------------ | ------- | ------- |
| [`IniciarHandler`](handlers/iniciar-handler.md) | `POST /devolucionesMasivo/iniciar` | Sí | 30s | 512 MB |
| [`WorkerHandler`](handlers/worker-handler.md) | — (invocación interna `Lambda.InvokeAsync`) | **No** | 900s (15 min) | 1024 MB |
| [`EstadoHandler`](handlers/estado-handler.md) | `GET /devolucionesMasivo/estado` | Sí | 30s | 512 MB |

El Worker no tiene ruta propia a propósito: si tuviera un endpoint HTTP, cualquiera con el nombre de la función podría dispararlo directo, saltándose la validación de usuario y la creación del job que hace el Iniciador.

---

## Flujo completo, punta a punta

```
Front: POST /iniciar { Guias: [...] }
  IniciarHandler
    -> LectorTokenCognito.Leer()           decodifica el JWT del header "Token" (no "Authorization")
    -> IniciarJobUseCase.EjecutarAsync()
         valida usuario + tiendas asignadas
         limpia/deduplica guias (NormalizadorGuia)
         valida tope de 3000 (JobDevolucion.MaximoGuiasPorJob)
         crea el documento en JobsDevolucion (Estado=Pendiente)
    -> Lambda.InvokeAsync(WorkerHandler, InvocationType.Event)   dispara y NO espera
    <- responde 200 { JobId, GuiasAceptadas, GuiasDescartadas }   en ~200ms

WorkerHandler (corre en el fondo, el front ya no lo espera)
  -> ProcesarJobUseCase.EjecutarAsync(jobId)
       lee GuiasPendientes del job (una Queue<string>)
       mientras haya pendientes y quede tiempo:
         toma un lote de 200
         lo resuelve por fases (ver mas abajo)
         escribe el progreso del lote (JobRepository.ActualizarProgresoAsync)
       si se acaba el tiempo con guias pendientes -> PedirContinuacionAsync
         si Continuaciones < 3 -> se re-invoca a si mismo CON EL MISMO JobId
         si no -> MarcarFallidoAsync
       si termina -> MarcarCompletadoAsync

Front: GET /estado?jobId=X   (polling cada 2.5s)
  EstadoHandler -> ConsultarJobUseCase -> findOne por _id, barato
    cuando Estado es Completado/Fallido, el front pide una vez mas con
    &detalle=true para traer el array Rechazadas completo
```

---

## Qué pasa cuando el Worker muere y se reactiva — ¿cambia el JobId?

**No. El `JobId` es el mismo en todas las continuaciones de un mismo lote.** Es una pregunta que vale la pena responder explícito porque es la pieza que hace que todo el diseño de idempotencia y de correlación (`JobId` en `InventarioDevolucion`, ver [ADR-003](adr/ADR-003-jobid-en-inventariodevolucion.md)) funcione sin casos borde.

Mecánica exacta (`WorkerHandler.cs` + `ProcesarJobUseCase.cs`):

1. El Worker recibe `{ "jobId": "abc123" }` como payload de invocación.
2. Dentro de `ProcesarJobUseCase.EjecutarAsync`, hay un margen de seguridad: se cancela 10 segundos antes del timeout duro de la función (`contexto.RemainingTime - 10s`), para alcanzar a guardar el progreso del lote en curso antes de que AWS corte en seco.
3. Si al terminar un lote de 200 guías queda menos del margen configurado (`MARGEN_CONTINUACION_SEGUNDOS`, por defecto 120s) para arrancar otro lote completo, **no arranca uno nuevo a medias** — llama a `PedirContinuacionAsync`.
4. `PedirContinuacionAsync` revisa `job.PuedeContinuar` (`Continuaciones < MaximoContinuaciones`, tope de **3**). Si puede continuar: `JobRepository.IncrementarContinuacionesAsync(jobId)` — incrementa el contador **sobre el mismo documento**, no crea uno nuevo.
5. `WorkerHandler` ve `resultado.RequiereContinuacion == true` y llama `ReinvocarAsync(jobId, contexto)`, que vuelve a invocar **la misma función Worker** con **el mismo `jobId`** en el payload.
6. La nueva invocación arranca de cero en `EjecutarAsync(jobId)`, lee el job de Mongo, y retoma `job.GuiasPendientes` exactamente donde quedó — porque `ActualizarProgresoAsync` ya había persistido esa lista antes de que la invocación anterior terminara.

Si se alcanza el tope de 3 continuaciones con guías todavía pendientes, el job se marca `Fallido` con un mensaje explícito (`"Se alcanzo el maximo de 3 continuaciones con N guias sin procesar."`) — el techo existe para que un job que no avanza (por ejemplo, un problema sistémico con una API externa) no se reinvoque indefinidamente facturando Lambda sin que nadie lo note.

**Por qué esto es seguro y no duplica trabajo:** cada continuación retoma con `job.GuiasPendientes`, que es la cola real de lo que falta — no reprocesa lo que el lote anterior ya escribió. Y aunque algo saliera mal y una guía se reprocesara, `ObtenerGuiasYaProcesadasAsync` (fase de idempotencia, ver abajo) la detectaría como `YaProcesada` y no volvería a tocar ninguna API externa ni a escribir dos veces en `InventarioDevolucion`.

---

## Las 4 fases de `ProcesarJobUseCase` (por lote de 200 guías)

Cada fase reduce el conjunto que pasa a la siguiente — lo caro (llamadas a APIs de transportadoras) solo lo paga lo que no se pudo resolver con lo barato (Mongo).

| Fase | Qué hace | Costo |
| ---- | -------- | ----- |
| **A — Idempotencia** | `ObtenerGuiasYaProcesadasAsync`: ¿esta guía ya tiene una fila con `Validacion="OK"` en `InventarioDevolucion`? | 1 query por trozo de 500 |
| **A — Match directo** | Busca por `GuiaDevolucion` y por `Numeropreenvio` en `PedidosInter`, en un solo `$in` por lote (nunca una consulta por guía) | 2 queries por lote |
| **B — Estrategias por formato** | Solo lo que Mongo no resolvió. `FactoryEstrategiaDevolucion` reparte cada guía a `EstrategiaConsultaExterna` (Inter o Envía) según el formato — ver [clientes de transportadora](infraestructura/clientes-transportadora.md) | 1 llamada HTTP por guía a la API externa, en paralelo acotado |
| **C — Reintento con otra transportadora** | Guías rechazadas por `SinGuiaOriginalEnRespuesta`, `ErrorConsultaExterna` (la transportadora asignada rechazó explícitamente) o `GuiaNoExisteEnSistema` (ninguna reconoció el formato) — ver [ADR-005](adr/ADR-005-confiabilidad-inter.md) | Proporcional a los fallos, no al lote completo |
| **D — Reglas transversales** | `ReglaPermisoTienda`, `ReglaPedidoAnulado` — se aplican DESPUÉS de resolver, sobre el pedido ya encontrado | En memoria, sin I/O |
| **Escritura** | `MarcarDevolucionCompletadaAsync` + `RegistrarEnInventarioDevolucionAsync` + `ActualizarLoteAsync` (inventario), en ese orden obligatorio | 3 escrituras en bulk por lote, nunca una por guía |

La idempotencia filtra por `Validacion == "OK"` específicamente — una guía **rechazada** no bloquea el reintento. Si el operario corrige el motivo (por ejemplo, le asignan la tienda que le faltaba) y vuelve a escanear la misma guía, tiene que poder reprocesarse.

---

## Endpoints

| Método | Ruta | Handler | Body / Query | Autenticación |
| ------ | ---- | ------- | ------------- | --------------|
| `POST` | `/devolucionesMasivo/iniciar` | `IniciarHandler` | `{ "Guias": ["...", ...] }` | Bearer (header `Token`, Cognito) |
| `GET` | `/devolucionesMasivo/estado` | `EstadoHandler` | sin `jobId` → jobs abiertos del usuario | Bearer (header `Token`, Cognito) |
| `GET` | `/devolucionesMasivo/estado?jobId=X` | `EstadoHandler` | detalle de un job puntual | Bearer (header `Token`, Cognito) |
| `GET` | `/devolucionesMasivo/estado?jobId=X&detalle=true` | `EstadoHandler` | agrega el array `Rechazadas` completo | Bearer (header `Token`, Cognito) |

**Header no estándar:** el JWT viaja en un header llamado `Token`, no `Authorization`. `LectorTokenCognito.NombreHeader = "Token"` — es el mismo esquema que usa el resto de la plataforma (`consumo-generico.service.ts:22` en el front). El código decodifica el payload del JWT pero **no verifica la firma** — confía en que el Authorizer de Cognito en API Gateway ya lo validó antes de invocar la Lambda; si el token fuera falso o hubiera vencido, la petición nunca habría llegado.

---

## Seguridad

- **Identidad**: `LectorTokenCognito.Leer()` decodifica el JWT (base64url del payload) sin verificar firma — la firma la verifica el Authorizer de API Gateway antes. Busca `email` en el token; si no está, cae a `sub` contra el campo `cognitoId` (arreglo) en la colección `Users`.
- **Permisos por tienda**: `ContextoUsuario.PuedeOperarTienda(tienda)` — `true` si el usuario tiene `"Todas"` en su lista de tiendas asignadas, o si la tienda puntual está en esa lista. `ReglaPermisoTienda` la aplica sobre cada guía resuelta, DESPUÉS de encontrar el pedido — nunca antes, porque hasta ese punto no se sabe de qué tienda es.
- **Aislamiento por dueño**: `ConsultarJobUseCase` valida que `job.Usuario == emailUsuario` antes de devolver cualquier dato. Si no coincide, responde `NoEncontrado` (no `NoAutorizado`) — decir "existe pero no es tuyo" ya confirma que ese `jobId` existe, información que el solicitante no debería tener (mismo criterio que usa GitHub con repos privados).
- **Secretos**: `CADENA_CONEXION`, `ID_CLIENTE`, `USER_AUTH`, `TOKEN_AUTH` viajan cifrados con AES-256-ECB (`EncripcionAES.DecryptAES256ECB`) en las variables de entorno, nunca en texto plano.

---

## Variables de entorno

| Variable | Descripción | Función que la usa |
| -------- | ----------- | ------------------- |
| `CADENA_CONEXION` | Cadena de conexión MongoDB, cifrada AES | Las 3 |
| `DATABASE_NAME` | Nombre de la base de datos | Las 3 |
| `NOMBRE_LAMBDA_WORKER` | Nombre de la función Worker a invocar | `IniciarHandler` (dispara), `WorkerHandler` (se reinvoca a sí mismo) |
| `HEADER_SECURITY` | Header de seguridad hacia `ApiLambdaActualizacionInventario` (vacío en PreProd — no lo valida) | `WorkerHandler` (vía `ServicioInventario`) |
| `URL_SERVICIO_LOGIGHO` | Base para `POST /actualizaInventarioLotes` | `WorkerHandler` |
| `INVENTARIO_SIMULADO` | `"true"` registra en log sin llamar al servicio real — para probar el flujo sin mover inventario | `WorkerHandler` |
| `URL_SERVICIO_ENVIA` | URL base de la API de Envía (guía en la ruta) | `WorkerHandler` |
| `CONCURRENCIA_ENVIA` | Guías en paralelo contra Envía (por defecto 5) | `WorkerHandler` |
| `URL_SERVICIO_LOGIN`, `URL_SERVICIO_INTER`, `URL_SERVICIO_GUIA` | 3 URLs base distintas de Interrapidísimo (login, rastreo, estados) | `WorkerHandler` |
| `ID_CLIENTE`, `USER_AUTH`, `TOKEN_AUTH` | Credenciales Inter, cifradas AES | `WorkerHandler` |
| `CONCURRENCIA_INTER` | Guías en paralelo contra Inter (por defecto 5, antes 10 — ver [ADR-005](adr/ADR-005-confiabilidad-inter.md)) | `WorkerHandler` |
| `PAUSA_ENTRE_PETICIONES_INTER_MS` | Pausa entre tandas de peticiones a Inter, en ms (por defecto 300) — ver [ADR-005](adr/ADR-005-confiabilidad-inter.md) | `WorkerHandler` |
| `MAXIMO_INTENTOS_CONSUMO` | (ver config PreProd) | `WorkerHandler` |
| `MARGEN_CONTINUACION_SEGUNDOS` | Segundos de margen antes de pedir continuación. **`"0"` desactiva el chequeo de tiempo por completo** — SOLO para pruebas locales con el Mock Lambda Test Tool, cuyo `RemainingTime` nunca simula el timeout real. En real nunca debe estar en 0 | `WorkerHandler` |

---

## Dependencias externas

| Servicio | Uso |
| -------- | --- |
| `API Interrapidísimo` (Login + ApiServInter + ApiVentaCredito) | `ClienteInter`: rastreo por guía de devolución (extrae la guía original de `DiceContener`) + estados de la guía original (fecha de devolución) |
| `API Envía` | `ClienteEnvia`: consulta unitaria por guía, extrae el número original de 12 dígitos del texto libre `anotaciones` |
| `MongoDB — PedidosInter` | Match directo, `MarcarDevolucionCompletadaAsync` |
| `MongoDB — InventarioDevolucion` | Idempotencia, registro histórico de cada guía procesada |
| `MongoDB — JobsDevolucion` | Estado del job, único canal entre Iniciador/Worker/front |
| `MongoDB — Users` | Tiendas asignadas al usuario |
| `ApiLambdaActualizacionInventario` (`/actualizaInventarioLotes`) | `ServicioInventario`: actualización de stock en lote, un solo POST por hasta 500 guías |

---

## Historial de cambios

| Fecha | Autor | Cambio |
| ----- | ----- | ------ |
| 2026-08-20 | Iker Acevedo | Diseño y construcción inicial: 3 funciones, dominio, estrategias, reglas de validación, repositorios, tests unitarios e integración. |
| 2026-08-24 | Iker Acevedo | Rediseño de `JobDevolucion`: de guardar el detalle completo de cada guía a solo contadores + `Rechazadas` (ver [ADR-002](adr/ADR-002-jobdevolucion-solo-contadores.md)). Fix del bug de idempotencia que bloqueaba el reintento de guías rechazadas. |
| 2026-08-25 | Iker Acevedo | `JsonStringEnumConverter` en `RespuestaHttp` para que los enums viajen como texto (antes como número), consistente con lo que ya guarda Mongo. `FechaInicio` expuesta en `EstadoJobResponse`. Despliegue a PreProd y validación end-to-end. |
| 2026-08-26 | Iker Acevedo | `JobId` agregado a cada fila de `InventarioDevolucion` (ver [ADR-003](adr/ADR-003-jobid-en-inventariodevolucion.md)). Fechas propias del módulo migradas a UTC real, dejando la excepción deliberada de `PedidosInter."Fecha Dev Completada"` en hora Colombia (ver [ADR-004](adr/ADR-004-fechas-utc.md)). Fix de un `CS0844` (propiedad duplicada en inicializador) en `ConsultarJobUseCase`. |
| 2026-09-02 | Iker Acevedo | Confiabilidad de `ClienteInter`: token real de 30s (antes se asumían 20 min), reintentos con backoff ante 429/5xx, pacing entre tandas, logging de fallos HTTP. Fallback cruzado de transportadora ampliado a 3 motivos de rechazo. Ver [ADR-005](adr/ADR-005-confiabilidad-inter.md). |

---

## Observaciones

- El Worker corre con **1024 MB** de memoria y **900s** de timeout — el doble de memoria que Iniciar/Estado, porque procesa lotes completos en memoria y hace llamadas HTTP concurrentes.
- `Iniciar` y `Estado` corren en la VPC de la plataforma (con `function-subnets`/`function-security-groups`); en PreProd esos campos están vacíos a propósito — no hace falta VPC en ese entorno.
- Ningún handler usa AOT (`PublishAot`) — a diferencia de otras lambdas del repo (`ApiLambdaCrudGenericoAOT`), este proyecto usa el runtime managed `dotnet10` por la cantidad de dependencias externas (HttpClient, MongoDB driver, System.Text.Json con reflexión) que complicarían la compilación nativa.
- Ver [`ApiLambdaDevolucionesMasivo.Tests`](../../../../../../LambdasLogiGho.Aplicacion/Logigho/ApiLambdaDevolucionesMasivo/ApiLambdaDevolucionesMasivo.Tests) para la suite completa: unitarios, integración contra Mongo real, y contract tests opcionales contra las APIs reales de Inter/Envía (`[Trait("Categoria","Externo")]`, activados por variable de entorno).
