# SmartService — guía breve para el examen final

> Base: inspección del código fuente actual de `C:\smartservice-backend` (04-09-2026). Los documentos existentes se usaron solo como orientación y se contrastaron con el código. **Hecho** indica evidencia directa; **interpretación** indica intención arquitectónica probable.

## 1. Mapa general de arquitectura

SmartService es un monorepo con arquitectura de microservicios NestJS. El `gateway` expone HTTP, aplica autenticación/autorización y transforma entradas externas; NATS conecta los servicios mediante request/reply (`@MessagePattern`) y eventos (`@EventPattern`/`emit`); Redis mantiene buffers, idempotencia y estado conversacional; Prisma/PostgreSQL persiste usuarios, reclamos, rutas y notificaciones.

```text
public-site / admin-dashboard (Next.js BFF)
                 |
                 v HTTP
gateway: controllers -> facades/proxies NATS -> ms-auth / ms-reclamos / ms-routing
   | webhook Meta -> Chain -> inbound Chain -> NATS request/reply -> ms-ai
   |                                                        |-> ms-geo
   |                                                        |-> ms-reclamos
   v
Redis (deduplicación, debounce, contexto)

ms-reclamos -- evento reclamos.v1.status.changed --> ms-notifications
Prisma/PostgreSQL: ms-auth, ms-reclamos, ms-routing, ms-notifications
```

Responsabilidades principales:

| Componente | Responsabilidad comprobada | Archivos de entrada |
|---|---|---|
| `gateway` | API HTTP, JWT/RBAC, webhooks, normalización y fachadas NATS | `src/app.module.ts`, controllers y services por módulo |
| `ms-auth` | usuarios, login/refresh/logout, roles y permisos | `src/auth`, `src/roles`, `src/permissions`, `prisma/schema.prisma` |
| `ms-ai` | seguridad de input, interpretación y máquina conversacional | `src/ai-process`, `src/ai-security`, `src/conversation` |
| `ms-geo` | adaptar Google Geocoding al contrato interno | `src/geo/geo.service.ts`, DTO/controller |
| `ms-reclamos` | agregado Reclamo, reglas de ciclo de resolución, historial y eventos | `src/reclamos`, `prisma/schema.prisma` |
| `ms-routing` | reglas, generación/confirmación de planes, optimización, rutas e intervenciones | `src/routing`, `prisma/schema.prisma` |
| `ms-notifications` | consumir cambios de estado, elegir canal, reintentar y auditar envíos | `src/notifications`, `prisma/schema.prisma` |
| `admin-dashboard` | UI y BFF/proxy hacia gateway | `src/app/api`, `src/services`, `src/lib/gateway-auth-proxy.ts` |

Flujo principal: Meta llama `WebhooksController` → `WebhooksService` ejecuta firma/deduplicación/rate limit/dispatch → `InboundService.receiveMessage` ejecuta sanitización/idempotencia/persistencia/debounce → `consolidateAndDispatch` llama `ai.process-inbound-message` → `AiProcessService` recupera estado Redis y elige un `ConversationStateHandler` → el handler usa `ConversationStateSupportService` para interpretar, geocodificar y crear el reclamo → `ReclamosService` persiste. Los cambios posteriores de estado publican `reclamos.v1.status.changed`, consumido por `NotificationsController`.

## 2. Strategy (implementación explícita)

### 2.1 Optimización de rutas

**Propósito.** Intercambiar algoritmos de optimización sin cambiar al cliente.

**Participantes.** `RouteOptimizationStrategy` define `optimize`; `LocalOptimizationStrategy` y `GoogleMapsOptimizationStrategy` son estrategias; `FallbackOptimizationStrategy` compone primaria/secundaria; `RoutingPlannerContext` selecciona; `RoutingService.buildSimulation` es cliente. Archivos: `ms-routing/src/routing/patterns/strategy/*`, `routing.module.ts`, `routing.service.ts` y `route-optimization.types.ts`.

**Flujo.** `RoutingService` construye borradores → `RoutingPlannerContext.optimize` consulta `useGoogleOptimization` → usa local o Google envuelto por fallback → devuelve rutas y contadores. Google reordena paradas vía Directions API; local conserva el orden; fallback captura el fallo global de la primaria.

**Pregunta probable: “agregar optimización por proveedor X”.** Crear una clase que implemente `RouteOptimizationStrategy`; añadirla a `RoutingModule.providers`; inyectarla en `RoutingPlannerContext`; extender la regla/opción que la selecciona y, si necesita configuración, `.env`/compose; añadir tests de la estrategia, selección y fallback. Si cambia el contrato, actualizar `route-optimization.types.ts`, DTO gateway y dashboard. **No modificar:** las estrategias actuales ni Prisma si no se persisten datos nuevos.

**Error frecuente.** Crear la clase pero no registrarla/seleccionarla; acoplar `RoutingService` al SDK; cambiar el orden de paradas sin recalcular `sequence`; confundir fallo parcial (Google conserva una ruta concreta) con excepción global (activa fallback).

### 2.2 Validaciones de seguridad de IA

**Propósito.** Encapsular reglas de validación independientes y ejecutarlas de manera extensible.

**Participantes.** `InputSecurityStrategy.validate`, token `INPUT_SECURITY_STRATEGIES`; `InputLengthStrategy`, `PromptInjectionStrategy`, `SensitiveAccessStrategy`, `ClaimKeywordStrategy`; `AiSecurityService.validateAndSanitizeInput` es contexto; `AiProcessModule` registra el arreglo. Archivos: `ms-ai/src/ai-security/strategies/*`, `ai-process/ai-security.service.ts`, `ai-process/ai-process.module.ts`.

**Flujo.** Nest inyecta el arreglo en el orden declarado → `validateAndSanitizeInput` recorta el texto → llama todas las estrategias hasta que una lanza `BadRequestException`.

**Pregunta: “incorporar una validación de lenguaje prohibido”.** Crear `ForbiddenLanguageStrategy implements InputSecurityStrategy`; declararla `@Injectable`; registrarla como provider; agregarla al `useFactory` y `inject` de `INPUT_SECURITY_STRATEGIES`; crear su spec y ampliar la prueba de orquestación. Definir conscientemente el orden, porque la primera excepción decide la respuesta. **No modificar:** `AiProcessService`, Redis ni otras estrategias salvo que cambie el contrato común.

**Error frecuente.** Registrar el provider pero omitirlo del arreglo; devolver boolean cuando el contrato es `void`/excepción; usar regex global con estado (`/g` + `test`) sin reiniciarla; duplicar en `AiSecurityService` una regla que debe vivir en una estrategia.

### 2.3 Canales de notificación

**Propósito.** Variar el proveedor de envío conservando un contrato único.

**Participantes.** `NotificationChannelStrategy`; `LogChannelStrategy`, `WhatsappChannelStrategy`; `NotificationsService.selectChannel` y `handleClaimStatusChanged`; `NotificationsModule`; `RetryPolicy` es una política separada, parecida a Strategy pero sin interfaz/intercambio dinámico. Archivos: `ms-notifications/src/notifications/*`.

**Flujo.** llega evento → filtra estado/destinatario → verifica idempotencia → `selectChannel` elige WhatsApp o log → persiste intento → `RetryPolicy` calcula intentos/backoff → actualiza estado.

**Pregunta: “agregar correo”.** Crear `EmailChannelStrategy`; registrarla en `NotificationsModule`; inyectarla y ampliar `selectChannel`; añadir configuración/secretos y tests; revisar DTO del evento, que ya admite `recipientEmail`; si se agrega metadata del proveedor, revisar Prisma/migración. **No modificar:** emisor `ms-reclamos` si el contrato existente alcanza; `WhatsappChannelStrategy` y `LogChannelStrategy`.

**Error frecuente.** Solo crear la estrategia; olvidar idempotencia/retry; seleccionar email aunque falte correo; persistir el intento con claves incompatibles; cambiar el texto y romper `WhatsappChannelStrategy.extractTemplateParameters`, que depende del formato exacto.

## 3. State (implementación explícita)

### 3.1 Conversación de IA

**Propósito.** Variar la respuesta según la fase persistida de la conversación.

**Participantes.** `ConversationPhase`/`ConversationState`; `ConversationStateHandler` (`canHandle`, `handle`) y token `CONVERSATION_STATE_HANDLERS`; estados `AwaitingIdentityAndClaimState`, `CollectingClaimDataState`, `ClosedConversationState`; contexto `AiProcessService`; colaborador compartido `ConversationStateSupportService`; composición en `AiProcessModule`.

**Flujo.** `AiProcessService` lee Redis o crea estado inicial → atiende como casos especiales tracking/respuesta ciudadana → busca handler por `canHandle(phase)` → ejecuta → guarda el estado resultante con TTL. Transiciones comprobadas: `awaiting_identity_and_claim` → `collecting_claim_data`; esta última permanece mientras faltan datos y pasa a `closed` al crear; `closed` puede reabrir a recopilación.

**Pregunta: “añadir fase `confirming_claim` antes de crear”.** Añadir literal a `ConversationPhase`; crear handler; modificar `CollectingClaimDataState` para transicionar a la fase nueva sin crear aún; registrar/injectar handler en `AiProcessModule`; implementar confirmación/cancelación; actualizar serialización/compatibilidad de estados Redis y tests de selección/transiciones. Probablemente mover la creación del reclamo al nuevo estado. **No modificar:** Prisma de reclamos si el agregado persistido no cambia; gateway y NATS si el payload/respuesta conservan contrato.

**Error frecuente.** Añadir el literal sin handler (produce “No hay handler…”); registrar la clase pero no el token; crear el reclamo antes de confirmar; olvidar estados ya guardados en Redis; poner lógica de fase en un gran `if` del contexto.

### 3.2 Ciclo de vida de ruta

**Propósito.** Delegar cambios y efectos temporales al estado actual.

**Participantes.** `RouteLifecycleState`, `RouteStatusTransitionContext/Update`; `BaseRouteState`, `AssignedState`, `InProgressState`, `CompletedState`, `CancelledState`; `RouteStateFactory`; `RouteLifecycleContext`; cliente `RoutingService.updateRouteStatus`; tipos `RoutingRouteStatus`; DTO gateway/microservicio y columna Prisma `RoutingRoute.status`.

**Flujo.** se carga ruta y autoriza actor → factory traduce `currentStatus` a objeto State → `transitionTo(target)` calcula `startedAt/completedAt` → transacción actualiza ruta y audita.

**Pregunta: “agregar estado `paused`”.** Extender `RoutingRouteStatus`; implementar `PausedState`; incorporarlo explícitamente en `RouteStateFactory`; definir transiciones/fechas en `BaseRouteState` o por estado; ampliar `UpdateRouteStatusDto` en `ms-routing` y su espejo en `gateway`; revisar UI/service del dashboard, persistencia y pruebas de contexto/service. Prisma usa `String`, por lo que no exige enum/migración salvo que se añadan columnas. **No modificar:** optimización Strategy, reclamos o notificaciones salvo que el nuevo estado de ruta deba provocar efectos allí.

**Advertencia comprobada.** El State actual calcula efectos, pero no contiene una matriz de transiciones prohibidas: estados completado/cancelado pueden volver a progreso y cualquier estado puede ir a asignado/cancelado/completado. Si el requisito dice “solo transiciones válidas”, la regla pertenece a los estados/contexto y sus tests, no solo al DTO.

**Error frecuente.** Hacer que la factory caiga silenciosamente en `CancelledState` para un nuevo valor; validar solo el valor objetivo; olvidar limpiar/conservar timestamps; actualizar estado fuera de la transacción de auditoría.

## 4. Chain of Responsibility (dos cadenas explícitas)

**Propósito.** Ejecutar pasos ordenados, permitiendo cortar el procesamiento.

### Cadena de webhook Meta

`WebhookHandler` y `WebhookContext` → `SignatureValidationHandler` → `DuplicateMessageHandler` → `RateLimitHandler` → `DispatchHandler`; ensamblada en `WebhooksService`, iniciada por `receiveMetaWhatsappWebhook`. Archivos: `gateway/src/webhooks/handlers/*`, `webhooks.service.ts`, controller/module.

### Cadena inbound normalizada

`AbstractWebhookHandler` y `WebhookProcessingContext` → `SanitizeWebhookHandler` → `IdempotencyWebhookHandler` → `PersistContextWebhookHandler` → `DebounceDispatchWebhookHandler`; ensamblada en `InboundService`, iniciada por `receiveMessage`. Archivos: `gateway/src/inbound/handlers/*`, `inbound.service.ts`.

**Pregunta: “bloquear tipos de contenido antes de consumir Redis”.** Si se refiere al payload crudo de Meta, crear un `WebhookHandler` y colocarlo antes de deduplicación/rate limit en `WebhooksService`; si se refiere al DTO interno multicanal, crear/extender handler inbound después de sanitizar pero antes de persistir. Actualizar contexto solo si transporta un dato nuevo y probar corte, orden y ausencia de side effects. **No modificar:** `ms-ai` para una regla puramente de borde.

**Error frecuente.** Insertar el handler en la cadena equivocada; olvidar llamar al siguiente; continuar después de `stopped/stopProcessing`; ubicar una validación cara antes de firma; crear timers y no liberarlos en `onModuleDestroy`.

**Distinción.** Los tres guards globales (`JwtAuthGuard`, `RolesGuard`, `PermissionsGuard`) registrados como `APP_GUARD` también forman un pipeline ordenado de autorización, pero es una cadena provista por Nest, no la implementación GoF manual anterior.

## 5. Observer / Publish–Subscribe

**Propósito.** Desacoplar al productor de los consumidores de un evento.

**Implementación comprobada.** `ReclamosService.publishStatusChanged` emite `reclamos.v1.status.changed` con `ClaimStatusChangedEvent`; `NotificationsController.handleClaimStatusChanged` observa con `@EventPattern` y delega al servicio. NATS también se usa en RPC con `send` + `@MessagePattern`; eso es request/reply, no Observer puro.

**Participantes/archivos.** `ms-reclamos/src/reclamos/reclamos.service.ts`, `events/claim-status-changed.event.ts`, `reclamos.module.ts`/`nats.module.ts`; `ms-notifications/src/notifications/dto/claim-status-changed.event.ts`, controller/service/module; `docker-compose.yml` y variables NATS.

**Pregunta: “auditar cambios de estado en otro consumidor”.** Crear un nuevo servicio/controlador con `@EventPattern` sin cambiar el productor si el evento trae todo lo necesario. Si falta un campo, evolucionar de forma compatible el DTO en productor y consumidores (o versionar tópico), construirlo en `publishStatusChanged`, actualizar pruebas de emisión/consumo. **No modificar:** la lógica transaccional del reclamo salvo que el momento de publicación cambie.

**Riesgo clave.** La publicación ocurre después de la transacción de actualización: puede persistirse el cambio y fallar el emit; no hay outbox comprobado. NATS/event handlers pueden reentregar: `ms-notifications` se protege con `idempotencyKey`. Error frecuente: reutilizar una clave inestable, romper el schema sin versionar o tratar un evento fire-and-forget como RPC.

## 6. Patrones estructurales y de framework

### Facade / API Gateway

Los services del gateway (`auth.service.ts`, `reclamos.service.ts`, `routing.service.ts`, roles/permisos) ocultan NATS, timeouts y traducción de errores a los controllers HTTP. Para agregar una operación: contrato/DTO en gateway → método facade `natsClient.send(topic, payload)` → endpoint controller → `@MessagePattern` y método service en el microservicio → tests de ambos bordes. **No** acceder a Prisma desde gateway.

### Adapter

`ms-geo/src/geo/geo.service.ts` adapta la respuesta de Google Geocoding al resultado interno; `DispatchHandler.mapMetaMessageToInbound` adapta Meta a `CreateInboundMessageDto`; `WhatsappChannelStrategy` adapta el mensaje interno al template de Graph API. Para cambiar proveedor, aislar formato/SDK en el adaptador y conservar el contrato interno. Actualizar variables, mocks y pruebas de errores. **No** filtrar objetos externos crudos hacia dominio/Prisma.

### Proxy/BFF

`admin-dashboard/src/lib/gateway-auth-proxy.ts` y rutas `src/app/api/**/route.ts` reenvían al gateway y centralizan cookies/autenticación del servidor. Para un endpoint administrativo nuevo: route handler BFF + service/UI + endpoint gateway; mantener secretos/cookies en servidor. **No** llamar microservicios/DB directamente desde componentes React. El `AGENTS.md` del dashboard exige consultar sus docs Next locales antes de editar esa aplicación; esta guía no la modifica.

### Decorator + Guards

Decoradores `@Public`, `@Roles`, `@Permissions`, `@RequestContext`, `@CurrentUser`, `@ClientIp`, `@MessagePattern` y `@EventPattern` adjuntan metadata o adaptan parámetros; guards la interpretan mediante `Reflector`. Para un permiso nuevo: seed/datos de autorización, decorador en endpoint, roles/permisos de gateway y ms-auth, tests de acceso. **No** codificar roles manualmente dentro del service de negocio si basta metadata + guard.

### Singleton e inyección de dependencias

`PrismaService` y `RedisService` son providers de módulo con una instancia compartida por contenedor Nest y hooks de conexión/cierre. Es más preciso decir “scope singleton de Nest” que “Singleton GoF clásico”: no tienen `getInstance` privado/global. Para otra conexión, crear provider/token/módulo claros y hooks; no instanciar Prisma/Redis por request.

### Factory Method / Builder (evidencia de framework)

`NestFactory.create`/`createMicroservice` construye aplicaciones y `DocumentBuilder` (gateway `main.ts`) construye configuración Swagger. `RouteStateFactory`, en cambio, es una factory de negocio explícita que mapea status a State. Cambios de bootstrap/configuración pertenecen a cada `main.ts`, módulos y `.env`; no a services de dominio. No presentar `NestFactory` como una fábrica escrita por el equipo: es uso de API del framework.

### Retry Policy (Policy Object)

`ms-notifications/src/notifications/retry-policy.ts` concentra máximo de intentos y backoff exponencial usado por `NotificationsService`. Para jitter o límites por canal, cambiar/extender esta política y sus tests; si se vuelve intercambiable, conviene introducir interfaz/token. **No** duplicar sleeps dentro de cada canal.

## 7. Tabla “si te piden X, revisar/modificar Y”

| Requerimiento | Modificar/revisar | Impacto y archivos que normalmente no se tocan |
|---|---|---|
| Atributo nuevo de `Reclamo` | `ms-reclamos/prisma/schema.prisma` + migración; create/update/response DTO y `ReclamosService`; DTO espejo/facade/controller gateway; IA si lo genera; dashboard si lo muestra; tests | Regenerar Prisma. No tocar routing si no selecciona/persiste ese dato. |
| Atributo de usuario | `ms-auth` schema+migración, DTO/service; gateway DTO/facade/controller; dashboard | Revisar JWT solo si el atributo entra al token; no tocar reclamos salvo scope/contrato. |
| Campo persistido de ruta | schema+migración `ms-routing`, DTO/types/service, gateway y UI, specs | No tocar State/Strategy si el campo no altera esas reglas. |
| Validación de DTO | DTO propietario con `class-validator`; espejo gateway; controller/pipes y tests | No ponerla en Prisma ni frontend como única defensa. |
| Regla de seguridad IA extensible | nueva `InputSecurityStrategy` + `AiProcessModule` + specs | No tocar `AiProcessService` si el contrato no cambia. |
| Nueva estrategia de optimización | strategy + `RoutingModule` + `RoutingPlannerContext`, opciones/DTO/config/tests | No reescribir las estrategias existentes. |
| Nuevo canal | strategy + `NotificationsModule` + selección/config/tests | No cambiar productor si evento actual basta. |
| Nuevo estado de conversación | phase/type + handler + registro + estado predecesor + specs/compatibilidad Redis | No tocar reclamos si payload final es igual. |
| Nuevo estado/transición de ruta | type, State/factory/context, DTOs gateway/routing, UI, tests; Prisma solo si cambia schema | No validar únicamente en controller. |
| Validación de webhook | handler y ensamblado de la cadena correcta; contexto/config/spec | No enviar payload inseguro a IA para que lo rechace tarde. |
| Regla de cierre de reclamo | `ReclamosService.close`/review/new visit, DTO y specs de integración; gateway/UI | No mezclar con Route State salvo requerimiento explícito. |
| Nueva categoría/prioridad | enums Prisma reclamos+migración, DTOs/tipos en reclamos/gateway/routing/IA, prompt/intérprete y validación de respuesta, reglas/UI/tests | Evitar listas divergentes; no cambiar solo un enum. |
| Nuevo evento | DTO versionado, productor `emit`, consumidor `@EventPattern`, NATS/config, tests e idempotencia | No usar `send` si no requiere respuesta. |
| Nuevo endpoint | controller+DTO+service gateway; patrón NATS y controller/service destino; dashboard si aplica | No acceder a DB desde gateway/BFF. |

## 8. Preguntas técnicas rápidas y respuestas

1. **¿Dónde vive una regla de negocio?** En el microservicio dueño del agregado: reclamos en `ReclamosService`, rutas en `RoutingService`/State/Strategy. Gateway y dashboard validan/adaptan acceso, pero no deben ser la única fuente de la regla.
2. **¿Por qué hay DTO duplicado en gateway y microservicio?** Son límites de transporte distintos (HTTP y NATS). Si cambia el contrato deben evolucionar juntos; la duplicación evita importar internals, aunque aumenta riesgo de desalineación.
3. **¿Cómo se agrega comportamiento sin romper lo existente?** Implementar la interfaz apropiada, registrar por DI, seleccionar explícitamente y mantener contratos; tests del componente y del punto de composición garantizan Open/Closed en la práctica.
4. **¿State y Strategy se parecen?** Ambos delegan. Strategy elige algoritmo/política; State elige comportamiento en función del estado persistido y modela transiciones.
5. **¿Dónde se controla autorización territorial?** En controllers/guards del gateway (por ejemplo pruebas de `routing.controller.spec.ts`) y datos de usuario `municipalityId/areaId`; el gateway sobrescribe filtros del cliente. Las consultas downstream también deben respetar el scope recibido.
6. **¿Qué asegura consistencia?** Transacciones Prisma en operaciones locales; no existe transacción distribuida comprobada. NATS, Redis y APIs externas requieren idempotencia, timeout, retry o compensación.
7. **¿Qué patrón protege reintentos?** Idempotencia con Redis en cadenas del gateway, `RoutingGenerationRequest.idempotencyKey` en routing y `Notification.idempotencyKey` en notificaciones. No es un GoF, sino un patrón de resiliencia.
8. **¿Dónde cambiar categorías aceptadas?** En todos los contratos cerrados: enum Prisma reclamos, DTO/tipos de gateway/routing, tipos y prompt/intérprete de IA, `AiSecurityService.validCategories`, reglas/UI y pruebas.
9. **¿Cuál es la intención incierta?** La documentación llama Observer a todos los `MessagePattern`, pero solo los eventos `emit`/`EventPattern` prueban pub/sub; request/reply es mensajería RPC. También Singleton/Factory/Builder dependen principalmente del framework.

## 9. Riesgos, dependencias y pruebas

- Cambios Prisma: crear migración, ejecutar `prisma generate`, probar compatibilidad/despliegue y datos existentes. Nunca editar migraciones aplicadas como sustituto de una nueva.
- Contratos: buscar tópico NATS y DTO en productor, consumidor y gateway. Mantener versionado e idempotency/correlation IDs.
- APIs externas: probar errores, timeout, credenciales ausentes y formato; no registrar tokens ni datos sensibles.
- Redis: considerar TTL, claves antiguas, carrera/idempotencia y limpieza de timers.
- Pruebas clave: `ms-ai/src/ai-security/**/*.spec.ts`, `ms-ai/src/conversation/states/*.spec.ts`, `ai-process.service.spec.ts`; `ms-routing/.../patterns/**/*.spec.ts` y `routing.service.spec.ts`; `ms-notifications/.../*.spec.ts` e integración NATS; `ms-reclamos/reclamos.integration.spec.ts`; specs de controllers gateway y tests del dashboard.
- Cobertura débil comprobada: varios tests de auth/roles/permisos solo verifican que el componente exista; una regla nueva exige casos de comportamiento y autorización, no solo actualizar el fixture.
- Verificación mínima por paquete afectado: `npm test -- --runInBand` y `npm run build`; integración cuando haya PostgreSQL/NATS/Redis. `ms-geo` no define script de tests.

## 10. Resumen para memorizar

1. **Composición:** `app.module.ts`/módulos registran providers, guards, strategies y state handlers.
2. **Contratos:** interfaces de patrones, DTOs, `routing.types.ts`, `conversation-state.type.ts`, eventos y tópicos NATS.
3. **Dominio:** `ReclamosService`, `RoutingService`, estados y estrategias; controllers son adaptadores de transporte.
4. **Persistencia:** cuatro `prisma/schema.prisma` independientes y sus migraciones.
5. **Extensión segura:** clase nueva + registro DI + selección/transición + contrato/config + tests.
6. **No tocar por defecto:** base de datos desde gateway, internals de otra estrategia/estado, o frontends cuando el contrato visible no cambia.
7. **Patrones más defendibles en examen:** Strategy, State y Chain (explícitos); Observer/pub-sub, Facade y Adapter (arquitectónicos); Proxy/Decorator/Singleton/Factory/Builder (principalmente framework); Retry/Idempotency (resiliencia).

