## Autor: Iker Acevedo

Fecha creacion: 2026-08-26

Estado: produccion

## Handler: WorkerHandler

**Accionador:** Invocación interna de Lambda a Lambda (`InvocationType.Event`), disparada por [`IniciarHandler`](iniciar-handler.md). **No tiene ruta en API Gateway.**
**Ubicación:** `Handlers/WorkerHandler.cs`
**Timeout:** 900s (15 min) — el máximo permitido por Lambda
**Memoria:** 1024 MB

---

## ¿Qué hace?

El motor del módulo. Procesa las guías pendientes de un job en lotes de 200, y si se queda sin tiempo, **se reinvoca a sí mismo con el mismo `JobId`** para continuar exactamente donde quedó. Ver [ApiLambdaDevolucionesMasivo.md — "Qué pasa cuando el Worker muere"](../ApiLambdaDevolucionesMasivo.md#qué-pasa-cuando-el-worker-muere-y-se-reactiva--cambia-el-jobid) para la mecánica completa de continuaciones.

---

## Por qué no tiene ruta propia

Si el Worker tuviera un endpoint HTTP, cualquiera con el nombre de la función podría invocarlo directo, saltándose toda la validación de usuario, la creación del job y la deduplicación de guías que hace `IniciarHandler`. Al no exponerse, la única forma de arrancar un procesamiento es pasar primero por `/iniciar` — el Worker confía en que el `jobId` que recibe corresponde a un documento válido ya creado.

---

## Payload de invocación

```json
{ "jobId": "a3f9c1e2d4b5467a9c1e2d4b5467a9c1" }
```

Sin `jobId`, el handler logea `worker-sin-jobid` y termina sin hacer nada — no hay respuesta HTTP que devolver, es una invocación asíncrona.

---

## Flujo interno

```
EjecutarAsync(evento, contexto)
  jobId = evento["jobId"]
  margen = contexto.RemainingTime - 10s     (minimo 30s)
  cts = CancellationTokenSource(margen)      cancela ANTES del timeout duro, para
                                              alcanzar a guardar el progreso del lote en curso

  Construir(contexto)                        cableado manual de dependencias (sin contenedor DI)
    DevolucionRepository, JobRepository, UsuarioRepository
    estrategias = [ EstrategiaConsultaExterna(ClienteInter), EstrategiaConsultaExterna(ClienteEnvia) ]
    reglas = [ ReglaPermisoTienda, ReglaPedidoAnulado ]
    FactoryEstrategiaDevolucion(estrategias, EstrategiaSinResolucion)
    ServicioInventario

  ProcesarJobUseCase.EjecutarAsync(jobId, cts.Token)     ver casos-uso/procesar-job-usecase.md

  si resultado.RequiereContinuacion:
    ReinvocarAsync(jobId, contexto)
      Lambda.InvokeAsync({ FunctionName: NOMBRE_LAMBDA_WORKER, InvocationType: Event, Payload: {jobId} })
      MISMO jobId — no se crea un job nuevo
```

**Sin `try/catch` alrededor de `casoUso.EjecutarAsync`, a propósito**: si algo explota sin control, la invocación queda marcada como fallida en las métricas de AWS y el stack trace completo llega a CloudWatch. Tragar la excepción acá la volvería invisible para monitoreo.

---

## Cableado manual de dependencias

`Construir(contexto)` arma todas las dependencias a mano, sin un contenedor de inyección de dependencias. Con pocas piezas es más explícito y evita el riesgo real de registrar como `Transient` algo que debería ser `Singleton` — un error de configuración de DI que ya costó caro en otro proyecto del ecosistema (SigueTuEnvio).

Registrar una transportadora nueva es agregar una línea al arreglo `estrategias` — ni el `FactoryEstrategiaDevolucion` ni `ProcesarJobUseCase` cambian (Open/Closed).

**El orden del arreglo importa desde [ADR-005](../adr/ADR-005-confiabilidad-inter.md):** además de decidir qué formato se prueba primero en la clasificación (irrelevante entre Inter/Envía, porque 12 y 13 dígitos son excluyentes), ahora define en qué orden `ObtenerAlternativas` prueba las transportadoras cuando una guía rechazada se reintenta con otra. Inter va primero porque es la transportadora con más volumen en la plataforma.

---

## Variables de entorno propias

Ver la tabla completa en [ApiLambdaDevolucionesMasivo.md](../ApiLambdaDevolucionesMasivo.md#variables-de-entorno) — el Worker es el único de los 3 handlers que usa casi todas: credenciales de Inter, URL de Envía, URL del servicio de inventario, y `MARGEN_CONTINUACION_SEGUNDOS`.

---

## Observaciones

- 1024 MB de memoria (el doble que Iniciar/Estado): procesa lotes de 200 guías en memoria y mantiene HTTP clients estáticos con conexiones concurrentes abiertas.
- El margen de cancelación (`RemainingTime - 10s`) es distinto del margen de continuación (`MARGEN_CONTINUACION_SEGUNDOS`, por defecto 120s): el primero protege contra que AWS corte en seco; el segundo decide si vale la pena arrancar otro lote completo o mejor pedir continuación ya.
- `MARGEN_CONTINUACION_SEGUNDOS="0"` desactiva el chequeo de tiempo — solo para pruebas locales con el Mock Lambda Test Tool, cuyo `RemainingTime` nunca simula el timeout real (siempre da ~0). En real nunca debe estar en 0.
