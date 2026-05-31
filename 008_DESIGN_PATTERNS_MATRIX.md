# 008 - Patrones de Diseno Tradicionales (GoF)

Fecha: 2026-05-31
Alcance: gateway, ms-auth, ms-ai, ms-geo, ms-reclamos, admin-dashboard

Este documento clasifica los patrones detectados en tres grupos clasicos:
1. Creacionales
2. Estructurales
3. De comportamiento

## 1) Creacionales

| Patron | Que significa (simple) | Donde se implementa |
|---|---|---|
| Singleton | Una sola instancia compartida durante la vida de la app. | [ms-auth/src/prisma/prisma.service.ts](../ms-auth/src/prisma/prisma.service.ts#L6), [ms-reclamos/src/prisma/prisma.service.ts](../ms-reclamos/src/prisma/prisma.service.ts#L9), [gateway/src/redis/redis.service.ts](../gateway/src/redis/redis.service.ts#L6), [ms-ai/src/redis/redis.service.ts](../ms-ai/src/redis/redis.service.ts#L6) |
| Factory Method | Se delega la creacion de objetos complejos al framework/fabrica. | [gateway/src/main.ts](../gateway/src/main.ts#L11), [ms-auth/src/main.ts](../ms-auth/src/main.ts#L12), [ms-ai/src/main.ts](../ms-ai/src/main.ts#L9), [ms-reclamos/src/main.ts](../ms-reclamos/src/main.ts#L10) |
| Builder | Construccion paso a paso de un objeto de configuracion. | [gateway/src/main.ts](../gateway/src/main.ts#L28) |

## 2) Estructurales

| Patron | Que significa (simple) | Donde se implementa |
|---|---|---|
| Facade | Una capa simplifica el acceso a subsistemas complejos. | [gateway/src/auth/auth.service.ts](../gateway/src/auth/auth.service.ts#L13), [gateway/src/reclamos/reclamos.service.ts](../gateway/src/reclamos/reclamos.service.ts#L17) |
| Adapter | Se adapta un formato externo al modelo interno del sistema. | [gateway/src/webhooks/webhooks.service.ts](../gateway/src/webhooks/webhooks.service.ts#L153), [ms-geo/src/geo/geo.service.ts](../ms-geo/src/geo/geo.service.ts#L36) |
| Proxy | Un intermediario controla y reenvia llamadas a otro servicio. | [admin-dashboard/src/lib/gateway-auth-proxy.ts](../admin-dashboard/src/lib/gateway-auth-proxy.ts#L17), [admin-dashboard/src/app/api/auth/login/route.ts](../admin-dashboard/src/app/api/auth/login/route.ts#L4) |
| Decorator (via anotaciones) | Se agrega comportamiento usando decoradores/metadata. | [gateway/src/app.module.ts](../gateway/src/app.module.ts#L1), [ms-auth/src/auth/auth.controller.ts](../ms-auth/src/auth/auth.controller.ts#L16), [ms-reclamos/src/reclamos/reclamos.controller.ts](../ms-reclamos/src/reclamos/reclamos.controller.ts#L18) |

## 3) De Comportamiento

| Patron | Que significa (simple) | Donde se implementa |
|---|---|---|
| Strategy | Se intercambia la forma de resolver una validacion/autenticacion. | [gateway/src/auth/jwt.strategy.ts](../gateway/src/auth/jwt.strategy.ts#L7) |
| Chain of Responsibility | Varias validaciones/guards se ejecutan en cadena. | [gateway/src/app.module.ts](../gateway/src/app.module.ts#L39), [gateway/src/common/guards/roles.guard.ts](../gateway/src/common/guards/roles.guard.ts#L7), [gateway/src/common/guards/permissions.guard.ts](../gateway/src/common/guards/permissions.guard.ts#L11) |
| Observer (Pub/Sub) | Un emisor publica eventos y otros servicios reaccionan. | [ms-auth/src/utils/audit.publisher.ts](../ms-auth/src/utils/audit.publisher.ts#L37), [ms-auth/src/auth/auth.controller.ts](../ms-auth/src/auth/auth.controller.ts#L16), [ms-geo/src/geo/geo.controller.ts](../ms-geo/src/geo/geo.controller.ts#L15), [ms-ai/src/ai-process/ai-process.controller.ts](../ms-ai/src/ai-process/ai-process.controller.ts#L15) |
| State | El comportamiento cambia segun el estado actual. | [ms-ai/src/ai-process/ai-process.service.ts](../ms-ai/src/ai-process/ai-process.service.ts#L109), [ms-ai/src/ai-process/ai-process.service.ts](../ms-ai/src/ai-process/ai-process.service.ts#L338) |

## 4) Resumen rapido por carpeta

| Carpeta | Patrones mas visibles |
|---|---|
| gateway | Facade, Adapter, Chain of Responsibility, Strategy |
| ms-auth | Observer (eventos NATS), Singleton (PrismaService), Factory Method (bootstrap) |
| ms-ai | State, Observer (NATS), Singleton (RedisService), Factory Method |
| ms-geo | Adapter (Google), Observer (MessagePattern), Factory Method |
| ms-reclamos | Observer (MessagePattern), Singleton (PrismaService), Factory Method |
| admin-dashboard | Proxy (BFF), Decorator (route handlers con metadata de framework) |

## 5) Nota de lectura

- Algunos patrones estan implementados de forma explicita en el codigo de negocio (por ejemplo State).
- Otros aparecen a traves del framework Nest/Next (por ejemplo Decorator, Factory Method en bootstrap).
- Este documento prioriza simplicidad: patron detectado y ubicacion concreta de implementacion.

## 6) Matriz academica para tesis

Escala usada:
- Nivel de evidencia: Alta (implementacion explicita), Media (implementacion parcialmente explicita), Baja (inferido por framework/convencion).
- Impacto arquitectonico: Alto, Medio, Bajo.

| Categoria | Patron | Evidencia principal | Nivel de evidencia | Impacto arquitectonico | Recomendacion de refactor |
|---|---|---|---|---|---|
| Creacional | Singleton | [ms-auth/src/prisma/prisma.service.ts](../ms-auth/src/prisma/prisma.service.ts#L6), [ms-reclamos/src/prisma/prisma.service.ts](../ms-reclamos/src/prisma/prisma.service.ts#L9), [gateway/src/redis/redis.service.ts](../gateway/src/redis/redis.service.ts#L6), [ms-ai/src/redis/redis.service.ts](../ms-ai/src/redis/redis.service.ts#L6) | Alta | Alto | Mantener. Agregar pruebas de ciclo de vida (onModuleInit/onModuleDestroy) y health checks para conexiones. |
| Creacional | Factory Method | [gateway/src/main.ts](../gateway/src/main.ts#L11), [ms-auth/src/main.ts](../ms-auth/src/main.ts#L12), [ms-ai/src/main.ts](../ms-ai/src/main.ts#L9), [ms-reclamos/src/main.ts](../ms-reclamos/src/main.ts#L10) | Media | Medio | Extraer configuraciones comunes de bootstrap a util compartido para reducir duplicacion entre microservicios. |
| Creacional | Builder | [gateway/src/main.ts](../gateway/src/main.ts#L28) | Alta | Medio | Mantener. Versionar el contrato Swagger y estandarizar metadatos (title/version/tags) en todos los servicios HTTP. |
| Estructural | Facade | [gateway/src/auth/auth.service.ts](../gateway/src/auth/auth.service.ts#L13), [gateway/src/reclamos/reclamos.service.ts](../gateway/src/reclamos/reclamos.service.ts#L17) | Alta | Alto | Mantener. Unificar manejo de errores RPC->HTTP en un mapper comun para todas las fachadas del gateway. |
| Estructural | Adapter | [gateway/src/webhooks/webhooks.service.ts](../gateway/src/webhooks/webhooks.service.ts#L153), [ms-geo/src/geo/geo.service.ts](../ms-geo/src/geo/geo.service.ts#L36) | Alta | Alto | Mantener. Formalizar contratos de entrada/salida por version para reducir riesgo ante cambios de APIs externas. |
| Estructural | Proxy | [admin-dashboard/src/lib/gateway-auth-proxy.ts](../admin-dashboard/src/lib/gateway-auth-proxy.ts#L17), [admin-dashboard/src/app/api/auth/login/route.ts](../admin-dashboard/src/app/api/auth/login/route.ts#L4) | Alta | Alto | Mantener. Incorporar trazabilidad (correlation-id) y politicas de timeout/retry por endpoint. |
| Estructural | Decorator | [gateway/src/app.module.ts](../gateway/src/app.module.ts#L1), [ms-auth/src/auth/auth.controller.ts](../ms-auth/src/auth/auth.controller.ts#L16), [ms-reclamos/src/reclamos/reclamos.controller.ts](../ms-reclamos/src/reclamos/reclamos.controller.ts#L18) | Media | Medio | Mantener. Documentar convenciones de decoradores (roles, permisos, message topics) en guia interna. |
| Comportamiento | Strategy | [gateway/src/auth/jwt.strategy.ts](../gateway/src/auth/jwt.strategy.ts#L7) | Alta | Alto | Mantener. Separar validaciones de payload en un validador dedicado para facilitar pruebas unitarias. |
| Comportamiento | Chain of Responsibility | [gateway/src/app.module.ts](../gateway/src/app.module.ts#L39), [gateway/src/common/guards/roles.guard.ts](../gateway/src/common/guards/roles.guard.ts#L7), [gateway/src/common/guards/permissions.guard.ts](../gateway/src/common/guards/permissions.guard.ts#L11) | Alta | Alto | Mantener. Definir orden formal de guards y casos borde en tests e2e de autorizacion. |
| Comportamiento | Observer (Pub/Sub) | [ms-auth/src/utils/audit.publisher.ts](../ms-auth/src/utils/audit.publisher.ts#L37), [ms-auth/src/auth/auth.controller.ts](../ms-auth/src/auth/auth.controller.ts#L16), [ms-geo/src/geo/geo.controller.ts](../ms-geo/src/geo/geo.controller.ts#L15), [ms-ai/src/ai-process/ai-process.controller.ts](../ms-ai/src/ai-process/ai-process.controller.ts#L15) | Alta | Alto | Mantener. Versionar topics criticos y estandarizar schema de eventos para evolucion segura. |
| Comportamiento | State | [ms-ai/src/ai-process/ai-process.service.ts](../ms-ai/src/ai-process/ai-process.service.ts#L109), [ms-ai/src/ai-process/ai-process.service.ts](../ms-ai/src/ai-process/ai-process.service.ts#L338) | Alta | Alto | Evolucionar a modelo de estado explicito (tabla de transiciones) para reducir complejidad ciclomatica y facilitar auditoria. |

## 7) Lectura de resultados

1. Los patrones con mayor impacto en la arquitectura actual son Facade, Adapter, Proxy, Observer, Strategy, Chain of Responsibility y State.
2. El sistema esta fuertemente apoyado en patrones estructurales y de comportamiento, coherente con una arquitectura de microservicios orientada a eventos.
3. En creacionales, predomina el uso pragmatico provisto por framework (Factory Method) y servicios singleton para infraestructura (Prisma/Redis).
4. El principal frente de mejora para tesis y evolucion tecnica es formalizar contratos y transiciones (eventos NATS y maquina de estados en ms-ai).
