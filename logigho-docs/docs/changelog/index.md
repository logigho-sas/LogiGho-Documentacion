# Hisotial de cambios

Historial de cambios, nuevas funcionalidades y correcciones del sistema LogiGho.

---

## Formato de entrada

```
## [version o fecha] — Descripción corta

### Nuevas funcionalidades
- Descripción de qué se agregó

### Cambios
- Descripción de qué cambió

### Correcciones
- Descripción de bugs corregidos

### Documentación
- Qué se documentó en esta entrega
```

---

## [2026-09-02] — DevolucionesMasivo: confiabilidad de Inter (token real, reintentos, fallback cruzado)

### Correcciones

- **Token de Inter vencía en 20 minutos según el código, en 30 segundos según Inter real** (confirmado decodificando el JWT que devuelve `GenerarTokenTemporal`) — causaba 401 masivos en cargas de más de 30s de duración. `_tokenVence` bajado a 20s.
- Fallos HTTP de `ClienteInter` (rastreo y estados) no dejaban ningún rastro en logs cuando Inter respondía distinto de 2xx — ahora se loguea `statusCode` y cuerpo de la respuesta en cada intento fallido.
- Sin reintentos ante `429`/`5xx` de Inter: ahora hasta 4 intentos con backoff exponencial + jitter, respetando `Retry-After`.
- Tráfico sostenido sin pausas entre tandas (`SemaphoreSlim` sin cortes) podía seguir gatillando 429 con lotes de 200+ guías — ahora hay pausa configurable entre tanda y tanda (`PAUSA_ENTRE_PETICIONES_INTER_MS`).
- Guías rechazadas por la transportadora asignada (`ErrorConsultaExterna`, ej. 400 explícito) o sin ningún formato reconocido (`GuiaNoExisteEnSistema`) no se reintentaban contra otra transportadora — solo cubría `SinGuiaOriginalEnRespuesta`. Ahora los 3 motivos disparan el fallback cruzado, con Inter primero por volumen.

### Cambios

- `CONCURRENCIA_INTER` por defecto bajada de 10 a 5 en código y en ambos entornos (prod/preprod).

### Documentación

- [ADR-005](../backend/lambdas-dotnet/aplicacion/Logigho/ApiLambdaDevolucionesMasivo/adr/ADR-005-confiabilidad-inter.md) con el diagnóstico completo. Actualizados `clientes-transportadora.md`, `worker-handler.md` y la tabla de variables de entorno del módulo.

---

## [2026-07-27] — Pancake: doble escritura de páginas, fix de ventana y endpoint on-demand

### Nuevas funcionalidades

- **Endpoint on-demand** `ApiLambdaConsultarEstadisticasPagina` (API Gateway `POST`): trae estadísticas frescas de una página (hoy / ayer / rango personalizado) **sin persistir**, con fallback de token de página.
- `ApiLambdaListarPaginasPancake` ahora hace **doble escritura** Mongo + Aurora MySQL.

### Correcciones

- Ventana `cierre_dia_anterior`: `until` corregido de `00:00:00` del día siguiente a `**23:59:59`** del mismo día — Pancake ya no arrastra gasto del otro día.

### Documentación

- Nueva página **Endpoint on-demand (API)** en Backend → Integración Pancake; actualizadas las páginas de `ListarPaginasPancake` (SQL) y `CalcularVentanaTiempo` (fix `until`).

---

## [2026-07-13] — Integración Pancake (estadísticas de campañas)

### Nuevas funcionalidades

- Pipeline serverless de **5 lambdas .NET 8** que recolecta las estadísticas de campañas de Pancake (`pages.fm`) 4 veces al día (7am/9am/2pm/5pm hora Colombia), orquestado con **AWS Step Functions** y agendado con **EventBridge Scheduler**.
- **Doble escritura** de estadísticas por campaña: MongoDB + **RDS Aurora MySQL** (patrón Composite, best-effort, UPSERT idempotente, modo inerte plug-and-play).

### Documentación

- Nueva sección **Backend → Integración Pancake**: visión general con diagrama de arquitectura, una página por lambda, orquestación (Step Functions + ASL + EventBridge) y guía de operación (ejecución manual, agregar franjas, monitoreo y troubleshooting).

