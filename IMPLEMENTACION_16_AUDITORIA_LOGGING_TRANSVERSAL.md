# Implementacion 16: auditoria transversal de logging

**Fecha:** 16 de octubre de 2026  
**Estado:** analisis estatico y plan de trabajo; no se modifico codigo ni se desplegaron cambios.

## Objetivo

Relevar el logging del monorepo antes de aplicar una correccion aislada. Se busca que cada operacion importante pueda reconstruirse entre Gateway y microservicios, conservando el formato legible usado hoy (`[correlationId] evento | clave=valor`) y sin registrar secretos, datos personales ni contenido ciudadano.

Este trabajo distingue:

- **Log operativo:** observabilidad de arranque, flujo, tiempos, resultados y errores tecnicos.
- **Auditoria de negocio:** quien hizo que cambio sobre que entidad/municipio y cuando; requiere retencion y destino durables.
- **Historial funcional:** eventos visibles en el producto, por ejemplo historial de reclamos/rutas. No sustituye ni el log operativo ni la auditoria.

## Metodo y alcance

Se hicieron busquedas de codigo en `src` de los servicios Nest, Gateway y Dashboard: 129 coincidencias de logger/console en 36 archivos, 105 referencias a `AuditPublisher`/auditoria en 11 archivos y 192 apariciones de `catch`/excepciones en 38 archivos. Se revisaron en detalle entradas de arranque, alta de reclamos, entrada conversacional, geocodificacion, WhatsApp, notificaciones y el publicador de auditoria.

Es un inventario estatico de la version local al 16/10/2026. Los conteos de busqueda no representan cobertura de eventos. No se inspeccionaron historiales de logs de Railway ni configuraciones de retencion/alertas, y no se ejecuto trafico real.

## Hallazgos

### Secretos expuestos en logs de arranque

- `ms-auth/src/main.ts` imprime `DATABASE_URL` completo con `console.log`. Si el valor contiene usuario/clave, termina en stdout.
- `gateway/src/main.ts` imprime `NATS_SERVERS` y `REDIS_URL` completos.
- `ms-ai/src/main.ts`, `ms-geo/src/main.ts`, `ms-notifications/src/main.ts`, `ms-reclamos/src/main.ts` y `ms-routing/src/main.ts` imprimen `NATS_SERVERS`. Una URL NATS puede contener credenciales.
- No se debe solucionar esto ocultando parcialmente passwords con regex improvisada. El log de arranque debe registrar nombres de servicio, entorno, puerto/host no sensible y versiones, nunca connection strings, query strings ni tokens.

### Datos personales y contenido ciudadano

- `ms-reclamos/src/reclamos/reclamos.controller.ts` escribe el payload completo de `reclamos.create` con `JSON.stringify`, que puede incluir telefono, correo, DNI, direccion, texto libre e historial de mensajes.
- `ms-ai/src/ai-process/ai-process.service.ts` registra `contactKey` y `consolidatedText` sin redaccion. El texto puede incluir DNI, telefonos, correos, domicilios y detalles sensibles.
- `gateway/src/inbound/inbound.service.ts` registra `contactKey`; ademas tiene una respuesta de MS-AI en nivel `debug`, que puede contener estado/contenido de conversacion.
- `ms-reclamos/src/reclamos/reclamos.service.ts` y el controlador tambien incluyen `contactKey` al registrar la creacion.
- `ms-geo/src/geo/geo.service.ts` registra direccion completa y coordenadas exactas. Direccion/ubicacion son datos personales cuando se vinculan al solicitante.
- `ms-notifications/src/notifications/log-channel.strategy.ts` registra telefono destinatario y mensaje completo. Esta estrategia es el fallback configurado junto a WhatsApp, por lo que no es solo codigo de prueba.
- `gateway/src/whatsapp/whatsapp.service.ts` registra el destinatario y errores de Meta sin una politica de saneamiento visible. Los errores externos deben limitarse a codigo, estado y referencia; no volcar cuerpos, headers ni URLs que puedan incluir tokens.

### Correlacion y formato

- La convencion mas consistente es logger Nest con contexto de clase y prefijo manual `[correlationId]`, seguido por evento y campos `key=value`.
- Hay operaciones sin `correlationId`, otras que registran solo `id`, otras que no dejan evento de exito/fallo y varios catch que reenvian el error sin registrar contexto.
- La correlacion se pierde o cambia entre mensajes NATS/HTTP. No todos los DTO/eventos tienen `correlationId`, `municipalityId`, actor ni identificador de operacion.
- El logging esta distribuido entre `Logger` de Nest, `console.error` del proxy y una estrategia de notificacion que imprime contenido. No se encontro configuracion compartida de Winston/Pino ni un formato/enlace global de redaccion.

### Auditoria de negocio no centralizada

- MS-Auth publica eventos `audit.log` mediante `AuditPublisher`, usado en Auth, usuarios, roles, permisos y configuracion municipal. El envio es fire-and-forget (`emit`) y no espera confirmacion.
- MS-Audit ya implementa consumidor NATS, persistencia PostgreSQL y consulta ADMIN; la migracion inicial fue aplicada y el esquema se verifico sin drift. La base no tiene aun eventos guardados y no hay validacion end-to-end con NATS productivo. NATS Core/fire-and-forget no retiene eventos si el consumidor esta offline.
- El consumidor persiste solo `audit.log`: los logs operativos de otros servicios siguen en stdout y todavia no se conectan a esa base. La consulta esta expuesta en Gateway, pero no hay pantalla React.
- Reclamos y Routing guardan historiales funcionales propios. No hay una correlacion central evidente entre cambio de usuario, reclamo, plan, ruta y entrega de notificacion.

### Cobertura de eventos incompleta

No existe una matriz de eventos obligatorios por servicio. En particular, no se asegura de forma uniforme:

- entrada/fin/resultado de operaciones y duracion;
- fallos tecnicos y dependencias degradadas (NATS, Redis, PostgreSQL, Meta, Google, S3);
- cambios de estado y decisiones de negocio con actor/municipio;
- reintentos, idempotencia, timeout y resultado final;
- shutdown ordenado y disponibilidad de dependencias.

El objetivo no es registrar cada lectura de base o cada helper; eso genera ruido y aumenta riesgo de PII. Se deben cubrir limites de procesos, dependencias y cambios de negocio auditables.

## Formato recomendado, compatible con el actual

Mantener logger Nest y texto legible. Para cada evento relevante, agregar siempre:

`[correlationId] evento | service=... | municipalityId=... | actorId=... | entidadId=... | resultado=... | durationMs=...`

Campos no aplicables se omiten. No imprimir objetos completos ni texto libre. Para identificar un ciudadano en diagnosticos repetidos, evaluar un HMAC con secreto rotatorio y alcance limitado; nunca registrar el telefono/correo/contactKey en claro ni usar un hash sin clave como anonimizado. Los errores externos se reducen a `provider`, `status`, `code`, `retry` y `reference`; stack y causa tecnica se conservan solo si estan saneados.

Niveles propuestos:

- `log`: hitos de operacion completada y cambios esperados relevantes.
- `warn`: rechazo esperado, fallback o dependencia degradada recuperable.
- `error`: operacion fallida, con `correlationId`, codigo de error y stack saneado.
- `debug`: metadatos tecnicos no sensibles; nunca request/response completo ni credenciales.

## Plan por fases

1. **Contencion inmediata de secretos/PII:** eliminar `DATABASE_URL raw`, dejar de imprimir connection strings, quitar payload completo, texto conversacional, contacto, direccion/coordenadas y mensaje de notificacion. Sustituir por campos allowlist. Anadir pruebas que demuestren que secretos/PII no aparecen.
2. **Contrato de correlacion compatible:** mantener `[correlationId]` y `key=value`; propagar un ID por HTTP/NATS, registrar inicio/final y duracion en limites Gateway/consumidor, incluir municipio y actor solo donde el contexto sea confiable.
3. **Matriz de eventos por servicio:** Auth/usuarios, Reclamos, AI, Geo, Routing, Notifications, Gateway/webhooks y almacenamiento. Para cada operacion declarar evento de exito, fallo, campos permitidos y nivel.
4. **Auditoria durable:** configurar y desplegar MS-Audit/NATS; definir consumer durable/ack o transactional outbox, reintentos, retencion, permisos y alarma por desconexion. No tratar un `emit` Core sin confirmacion como auditoria durable.
5. **Sanitizacion y defensa:** centralizar saneamiento de excepciones; cubrir tokens, passwords, connection strings, headers Authorization, query strings, cuerpos Meta/Google/S3 y datos ciudadanos. No depender solo de ocultar a nivel de plataforma.
6. **Verificacion:** pruebas unitarias de redaccion y campos; test de correlacion HTTP→NATS; revision de logs en entorno de staging con datos sinteticos; validar retencion, acceso, busqueda y alertas. No enviar PII real para probar logs.

## Criterios de aceptacion

- Ningun log imprime `DATABASE_URL`, `REDIS_URL`, `NATS_SERVERS` completo, tokens ni headers de autorizacion.
- Ningun log operativo imprime telefono, correo, DNI, direccion/coordenadas asociadas, texto de reclamo, conversacion o mensaje de WhatsApp.
- Una operacion puede seguirse con `correlationId` desde entrada Gateway hasta respuesta/error del servicio consumidor.
- Los eventos de auditoria de cambios identifican actor, municipio, entidad, accion y timestamp; el destino confirma persistencia y tiene retencion/alertas definidas.
- Los errores conservan informacion accionable sin request/response ni secretos.
- Los tests fallan si reaparece payload completo o PII en logs.

## Siguiente paso recomendado

Empezar por la fase 1 en `ms-auth/src/main.ts`, `gateway/src/main.ts`, los demas bootstraps, `ms-reclamos`, `ms-ai`, `ms-geo` y `ms-notifications`; despues configurar NATS y probar el consumidor MS-Audit, fortalecer durabilidad y migrar productores por servicio. El alcance es transversal, por lo que conviene implementar y desplegar por fases, no reemplazar todos los loggers de una vez.
