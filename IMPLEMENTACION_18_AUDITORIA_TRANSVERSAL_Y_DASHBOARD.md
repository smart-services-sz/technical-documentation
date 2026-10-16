# Implementacion 18: auditoria transversal y consulta en dashboard

**Fecha:** 16 de octubre de 2026  
**Estado:** instrumentacion y vista implementadas en workspace; suites/builds locales aprobados. Falta desplegar estos cambios y verificar la ingesta NATS en el entorno.

## Resultado

Gateway registra resultado HTTP, codigo, ruta normalizada y duracion. Genera un `correlationId` UUID si no llega uno valido, lo propaga al request y devuelve `x-correlation-id`. Actor y municipio se toman del JWT. No se incluyen query string, body, respuesta ni headers.

MS-AI, MS-Auth, MS-Geo, MS-Notifications, MS-Reclamos y MS-Routing registran operaciones NATS entrantes mediante un interceptor global. Cada evento incluye `eventId`, servicio, subject, entidad/ID allowlist, actor/municipio cuando existen, correlacion, resultado, duracion y tipo de error. No se copia el payload ni la respuesta. MS-Audit no se auto-instrumenta para evitar un bucle recursivo; sus fallos de persistencia conservan un log operativo saneado.

Se retiraron de logs de consola URLs de conexion, payload completo de reclamos, contacto/texto conversacional, direcciones/coordenadas, destinatario/mensaje WhatsApp, cuerpos de error del proveedor y fragmentos de prompt. El canal fallback `log` conserva el hito con recipient/content redactados. MS-Audit redacciona metadata con claves de secretos/PII y limita el tamano.

El Dashboard agrega la seccion **Auditoria**, solo para ADMIN: filtros por servicio, actor, accion, nivel, entidad/ID, municipio, correlacion y rango de fechas; tabla paginada de 25 registros, maximo 100 por consulta. Metadata, IP y User-Agent aparecen solo al desplegar el detalle. Gateway fuerza el municipio del JWT para un ADMIN municipal; el ADMIN de plataforma sin municipio conserva filtros globales.

## Formato y cobertura

Los logs operativos mantienen Nest `Logger` y el prefijo `[correlationId] evento | clave=valor`. Los eventos estructurados enviados a MS-Audit agregan `eventId`, `municipalityId` y `correlationId` al contrato existente; la metadata de interceptores contiene transporte, operacion, resultado, status/duracion y tipo de error.

Gateway cubre handlers HTTP; los seis microservicios cubren handlers NATS. No se registran lecturas internas/funciones auxiliares individualmente. MS-Audit se excluye de su propio interceptor para no generar recursividad.

## Archivos modificados

- Gateway: `src/audit/audit-logging.interceptor.ts`, `src/audit/audit-logging.module.ts`, prueba del interceptor, registro global en `src/app.module.ts` y correlacion en `src/common/decorators/request-context.decorator.ts`.
- MS-AI, MS-Auth, MS-Geo, MS-Notifications, MS-Reclamos y MS-Routing: interceptores/modulos globales bajo `src/audit/` y registro en `AppModule`; MS-AI incluye prueba de redaccion y MS-Auth conserva prueba del publisher.
- MS-Auth: se retiro la impresion de `DATABASE_URL`; `AuditPublisher` mantiene eventId UUID.
- MS-Reclamos, MS-AI, MS-Geo y Notifications: se sanearon los logs de payload/contacto/texto/direccion/coordenadas/destinatario.
- Dashboard: `/api/audit/logs`, servicio de filtros, `audit-logs-panel.tsx`, estilos, pruebas del panel y navegacion ADMIN.
- MS-Audit: redaccion central de claves de secretos y PII.

## Validacion ejecutada

- Gateway: 4 pruebas enfocadas de endpoint/interceptor; build aprobado.
- MS-AI: 11 pruebas de interceptor/estrategias; build aprobado.
- MS-Auth: prueba del publisher; build aprobado.
- MS-Geo: build aprobado; no define suite de pruebas.
- MS-Notifications: prueba de redaccion del canal `log`; build aprobado.
- MS-Reclamos: 4 suites, 18 pruebas aprobadas y 1 integracion omitida; build aprobado.
- MS-Routing: 5 suites y 41 pruebas aprobadas; build aprobado.
- MS-Audit: 2 suites y 7 pruebas aprobadas; build aprobado.
- Dashboard: 9 archivos y 35 pruebas aprobadas; typecheck y build de produccion aprobados.
- Busqueda estatica: sin coincidencias en logs productivos para los patrones revisados de connection strings, contacto, contenido, direccion o coordenadas. Las coincidencias restantes son DTOs, fixtures o valores explicitamente redactados.

Las pruebas usan NATS y servicios simulados. No se ejecuto una operacion real a traves de NATS durante esta entrega.

## Pendiente de despliegue y durabilidad

Desplegar Gateway, Dashboard y los seis productores instrumentados con `NATS_SERVERS` apuntando al mismo NATS privado de MS-Audit. Ejecutar una accion no sensible por servicio y verificar conteo, entidad, correlacion y municipio en PostgreSQL. No utilizar datos ciudadanos reales como prueba.

NATS Core `emit` fire-and-forget puede perder eventos si MS-Audit esta desconectado. `eventId` permite deduplicar un reenvio, pero no proporciona replay. Si se requiere entrega durable, evaluar JetStream con consumer durable/ack o transactional outbox. Revisar tambien retencion y acceso de IP/User-Agent y metadata antes de conservar logs de produccion.
