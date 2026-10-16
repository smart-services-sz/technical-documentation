# Implementacion 16: auditoria transversal de logging

**Fecha:** 16 de octubre de 2026  
**Estado:** el analisis se implemento parcialmente en [la entrega 18](IMPLEMENTACION_18_AUDITORIA_TRANSVERSAL_Y_DASHBOARD.md): interceptores globales, saneamiento de logs sensibles y consulta ADMIN. Falta confirmar el despliegue/ingesta NATS real y resolver durabilidad.

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

- `ms-auth/src/main.ts` imprimia `DATABASE_URL` completo; se retiro en la entrega 18.
- `gateway/src/main.ts` imprimia `NATS_SERVERS` y `REDIS_URL` completos; ahora informa solo si estan configurados.
- Los bootstraps de MS-AI, Geo, Notifications, Reclamos y Routing ya no imprimen URLs NATS completas.
- No se debe solucionar esto ocultando parcialmente passwords con regex improvisada. El log de arranque debe registrar nombres de servicio, entorno, puerto/host no sensible y versiones, nunca connection strings, query strings ni tokens.

### Datos personales y contenido ciudadano

- Esos puntos se sanearon en la entrega 18: payload completo, texto/contactKey, direccion/coordenadas, destinatario y cuerpo de error de proveedor ya no se escriben en logs de consola.

### Correlacion y formato

- La convencion mas consistente es logger Nest con contexto de clase y prefijo manual `[correlationId]`, seguido por evento y campos `key=value`.
- Hay operaciones sin `correlationId`, otras que registran solo `id`, otras que no dejan evento de exito/fallo y varios catch que reenvian el error sin registrar contexto.
- La correlacion se pierde o cambia entre mensajes NATS/HTTP. No todos los DTO/eventos tienen `correlationId`, `municipalityId`, actor ni identificador de operacion.
- El logging esta distribuido entre `Logger` de Nest, `console.error` del proxy y una estrategia de notificacion que imprime contenido. No se encontro configuracion compartida de Winston/Pino ni un formato/enlace global de redaccion.

### Auditoria de negocio no centralizada

- MS-Auth publica eventos `audit.log` mediante `AuditPublisher`, usado en Auth, usuarios, roles, permisos y configuracion municipal. El envio es fire-and-forget (`emit`) y no espera confirmacion.
- MS-Audit ya implementa consumidor NATS, persistencia PostgreSQL y consulta ADMIN; la migracion inicial fue aplicada y el esquema se verifico sin drift. La base no tiene aun eventos guardados y no hay validacion end-to-end con NATS productivo. NATS Core/fire-and-forget no retiene eventos si el consumidor esta offline.
- La entrega 18 agrega interceptores operativos NATS en Gateway y los seis microservicios y una vista ADMIN en Dashboard. Estos productores estan en codigo local; falta confirmar que sus versiones desplegadas publiquen y que MS-Audit reciba los eventos.
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

1. **Contencion de secretos/PII:** implementada en codigo en la entrega 18 con allowlist y pruebas negativas. Falta verificar los logs del entorno desplegado y conservar pruebas de regresion.
2. **Contrato de correlacion compatible:** interceptores HTTP/NATS implementados con prefijo y campos allowlist; municipio/actor se incluyen cuando vienen del contexto autenticado.
3. **Matriz de eventos por servicio:** Gateway HTTP y handlers NATS de los seis microservicios instrumentados; faltan pruebas integradas y revisar procesos batch/background si se incorporan.
4. **Auditoria durable:** MS-Audit ya persiste `audit.log` y el Dashboard permite consulta ADMIN. Definir consumer durable/ack JetStream u outbox, reintentos, retencion, permisos y alarma por desconexion; NATS Core `emit` no garantiza entrega.
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

La fase de contencion y cobertura en codigo se implemento en la entrega 18. Siguiente: desplegar productores, confirmar ingesta real con datos sinteticos y decidir JetStream/outbox; mantener el formato legible existente.
