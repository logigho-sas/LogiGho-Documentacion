## Autor:
Fecha creacion: 2026-09-02
Estado: aceptada

# ADR-005 — Confiabilidad de ClienteInter: token real de 30s, reintentos con backoff, pacing entre tandas y fallback cruzado de transportadora

**Autor:** Iker Acevedo
**Fecha:** 2026-09-02
**Estado:** Aceptada

---

## Contexto

En producción, cargas de guías de Interrapidísimo empezaron a rechazarse casi por completo, con un patrón intermitente: un lote chico (1 guía) a veces pasaba, y el siguiente lote —incluso corrido segundos después, mismas guías— salía 100% rechazado. Un lote de 92 guías sí había funcionado unos días antes. La sospecha inicial del equipo (correcta en parte): "estamos mandando muchas peticiones seguidas y algo nos está bloqueando".

El diagnóstico fue en capas, porque el código no dejaba rastro suficiente para ver la causa real:

1. **`ClienteInter.RastrearAsync` no logueaba nada cuando Inter respondía con un status HTTP distinto de 2xx.** El fallo quedaba encapsulado en el `record Rastreo` y solo se contaba en el resumen final (`conError: N`) — cero información de *por qué*. Se agregó logging del `statusCode` y el cuerpo de la respuesta antes de poder avanzar con cualquier otra hipótesis.
2. Con el logging puesto, empezaron a aparecer **401 "No está autorizado"**, no 429 — la sospecha original de rate limit no era la causa raíz (aunque sí era un riesgo real y separado, ver punto 4).
3. **Se decodificó el JWT que devuelve `GenerarTokenTemporal`** (payload en Base64, sin necesidad de herramientas especiales) y se confirmó, dos veces con tokens reales distintos, que `exp - iat = 30` — **el token de Inter vive 30 segundos reales**, no los 20 minutos que asumía `_tokenVence = DateTime.UtcNow.AddMinutes(20)`. Ese número de 20 minutos nunca había sido verificado contra Inter, era una estimación "conservadora" arbitraria.
4. Aparte del bug del token, **el diseño de concurrencia (`SemaphoreSlim`, 10 en paralelo, sin pausa entre tandas) sí podía gatillar 429 real** con volúmenes de 200+ guías, un riesgo distinto y adicional al del token.
5. Revisando una guía real reportada como "de Inter" que el sistema rechazaba (`700203753929`, 12 dígitos), se encontró un tercer problema, independiente de los dos anteriores: el sistema la clasificaba como **Envía** (coincide con la regla "12 dígitos"), Envía la rechazaba con 400 (confirmado contra la API real desde Postman), y ahí se quedaba — sin probar con Inter, porque el mecanismo de reintento cruzado de transportadora (`MereceReintentoConOtraTransportadora`) solo cubría un motivo de rechazo (`SinGuiaOriginalEnRespuesta`), no un 400 explícito de la transportadora ni una guía que no calzó con ningún formato conocido.

Los puntos 1-4 explican por qué guías genuinamente de Inter se rechazaban en producción. El punto 5 es un problema de clasificación aparte, expuesto por la misma investigación.

---

## Opciones consideradas

### Para el token (puntos 2-3)

**Opción A — Bajar la vigencia cacheada a un valor cercano al real, más invalidación reactiva en 401**

Cachear el token por ~20s (margen bajo los 30s reales) y, además, invalidar el caché y reintentar automáticamente si Inter responde 401 en cualquier momento — sin depender de acertarle al número exacto de vigencia real de Inter.

**Pros:** no necesita que Inter documente ni garantice su TTL real; se autocorrige si Inter cambia esa vigencia en el futuro. Sigue amortizando el costo de login contra pedir token por cada guía.
**Contras:** ninguno relevante — es estrictamente mejor que asumir un número fijo sin verificar.

### Para la concurrencia (punto 4)

**Opción A — Tandas de tamaño fijo con pausa entre tanda y tanda, en vez de solo un tope de concurrencia**

Reemplazar el `SemaphoreSlim` (topa cuántas van a la vez, pero la siguiente arranca apenas se libera un cupo, sin pausa) por un `for` que procesa en tandas de `_concurrencia` y espera `_pausaEntreTandas` entre cada una.

**Pros:** simple de razonar y de ajustar por variable de entorno sin tocar código; resuelve el tráfico sostenido sin cortes que un semáforo solo no evita.
**Contras:** ninguno significativo — el volumen total tarda un poco más (pausas explícitas), aceptable dado el margen de 15 minutos por invocación.

### Para la clasificación/fallback de transportadora (punto 5)

**Opción A — Ampliar cuándo vale la pena reintentar con otra transportadora, reusando el mecanismo ya existente**

`GuiaProcesada.MereceReintentoConOtraTransportadora` pasa de cubrir solo `SinGuiaOriginalEnRespuesta` a cubrir también `ErrorConsultaExterna` (la transportadora asignada rechazó explícitamente, ej. 400) y `GuiaNoExisteEnSistema` (ninguna transportadora reconoció el formato). `ProcesarJobUseCase.ReintentarAsync` ya sabía agrupar por transportadora ya intentada y pedirle al factory las alternativas — no hizo falta ningún mecanismo nuevo.

**Pros:** cero código nuevo de orquestación, reusa 100% la cañería existente (DRY). Cubre en un solo cambio tanto "la transportadora asignada dijo que no" como "no se pudo asignar ninguna".
**Contras:** durante una caída real de una transportadora (como el propio incidente de esta ADR), cada guía rechazada ahora dispara una llamada extra a la otra transportadora — más tráfico durante un incidente. Se acepta porque es una sola llamada extra por guía, de solo lectura, no en bucle.

### Opción B considerada y descartada — reintentar con otra transportadora en TODOS los rechazos, sin distinguir motivo

Se descartó: rechazos por reglas de negocio (`ReglaPermisoTienda`, `ReglaPedidoAnulado`) o por guía ya procesada no tienen sentido reintentados con otra transportadora — el problema no es de qué transportadora se consultó.

---

## Decisión

**Se eligió:** Opción A en los tres frentes.

**Razón:** en los tres casos la opción elegida es la que menos asume y más se apoya en evidencia real (JWT decodificado, prueba directa en Postman) en vez de suposiciones no verificadas — que es justamente lo que causó el bug original del token.

---

## Consecuencias

**Positivas:**
- El 401 masivo por token vencido queda resuelto — se refresca antes de que Inter lo invalide, y si aun así falla, se autocorrige en el siguiente intento.
- El riesgo de 429 por tráfico sostenido baja con el pacing entre tandas, ajustable por variable de entorno sin redeploy de código.
- Guías reales que antes se perdían silenciosamente (mal clasificadas, o sin ningún formato reconocido) ahora tienen una segunda oportunidad automática contra las demás transportadoras.
- Cualquier fallo HTTP de Inter ahora queda en el log con `statusCode` y cuerpo — la próxima vez que algo falle, hay evidencia real desde el primer minuto, no una investigación de varias horas como esta.

**Negativas:**
- Más tráfico hacia las APIs externas durante una caída real de una transportadora (cada guía rechazada intenta la alternativa una vez). Aceptado como costo menor frente al riesgo de perder guías reales.
- `ClienteEnvia` sigue sin reintentos HTTP propios (429/5xx) — quedó fuera de alcance de esta ADR, es una mejora pendiente simétrica a la que se le hizo a `ClienteInter`.
- La pregunta de fondo del punto 5 (¿cuál es el patrón real y completo de las guías Inter que no siguen "13 dígitos + empieza en 30"?) sigue sin resolverse — el fallback cruzado es una red de seguridad, no un fix de la regla de clasificación en sí. Queda pendiente acumular más ejemplos reales en producción antes de tocar `NormalizadorGuia.PareceInter`.

---

## Impacto en el código

| Módulo / Archivo | Cambio |
| ----------------- | ------ |
| `ClienteInter.cs` | `_tokenVence` de `AddMinutes(20)` a `AddSeconds(20)`. Nuevo `EnviarConReintentosAsync`: reintentos con backoff exponencial + jitter para 429/5xx, respeta `Retry-After`, loguea `statusCode` y cuerpo en cada fallo. `ConcurrenciaPorDefecto` de 10 a 5. Nueva `_pausaEntreTandas` (env `PAUSA_ENTRE_PETICIONES_INTER_MS`, default 300ms) — tandas explícitas con pausa en vez de solo `SemaphoreSlim`, tanto en rastreo como en lotes de estados. |
| `GuiaProcesada.cs` | `MereceReintentoConOtraTransportadora` ampliada de 1 a 3 motivos: `SinGuiaOriginalEnRespuesta`, `ErrorConsultaExterna`, `GuiaNoExisteEnSistema`. |
| `EstrategiaConsultaExterna.cs` | Método `Rechazo` deja de ser `static` (necesita `this.Transportadora`), marca `Transportadora` en cada `GuiaProcesada` rechazada — así `ReintentarAsync` sabe cuál ya se probó. |
| `EstrategiaSinResolucion.cs` | Marca `Transportadora = "SIN RESOLUCION"` en el rechazo (cosmético para logs; no afecta la lógica de `ObtenerAlternativas`, que ya devolvía todas las transportadoras reales para cualquier nombre no registrado). |
| `WorkerHandler.cs` | Orden del arreglo `estrategias` invertido: Inter antes que Envía — define la prioridad del fallback cruzado, no solo la clasificación por formato (que es indiferente al orden). |
| `aws-lambda-worker.json`, `aws-lambda-worker.preprod.json` | `CONCURRENCIA_INTER` de 10 a 5 (ambos entornos, ahora iguales). Nueva variable `PAUSA_ENTRE_PETICIONES_INTER_MS=300`. |
| `ConsultarJobUseCaseTests.cs` | Fix de un test preexistente y roto (`CrearAsync` sin `await` ni `CancellationToken`), no relacionado con esta ADR pero bloqueaba correr la suite completa. |

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-09-02 | Iker Acevedo | Decisión inicial: token TTL real descubierto (30s), reintentos con backoff, pacing entre tandas, fallback cruzado de transportadora ampliado. |

---

## Referencias

- [`ApiLambdaDevolucionesMasivo.md`](../ApiLambdaDevolucionesMasivo.md) — variables de entorno y flujo general.
- [`clientes-transportadora.md`](../infraestructura/clientes-transportadora.md) — detalle actualizado de `ClienteInter`, `ClienteEnvia` y el fallback cruzado.
- [`worker-handler.md`](../handlers/worker-handler.md) — orden de registro de estrategias.
