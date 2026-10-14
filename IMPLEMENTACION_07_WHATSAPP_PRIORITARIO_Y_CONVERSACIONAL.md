# Implementación 07: WhatsApp prioritario y ciclo conversacional

**Fecha:** 14 de octubre de 2026
**Base:** brechas de notificaciones, trazabilidad y seguimiento de la auditoría SmartService.
**Estado:** implementado y validado con pruebas simuladas; no se conectó Meta, NATS productivo ni bases de datos.

## Resumen

WhatsApp queda como canal predeterminado para notificaciones con teléfono (`NOTIFICATION_CHANNELS=whatsapp,log`). Las plantillas incluyen cinco parámetros para comunicar el código/estado y sugerir los comandos ciudadanos `CONFORME`, `DISCONFORME: motivo` y `REABRIR: motivo`.

MS-AI ahora intercepta mensajes de seguimiento con código antes de ejecutar el parser de alta. Comprueba el teléfono/correo canónico mediante `reclamos.public-tracking`; luego registra confirmación, disconformidad o reapertura por NATS y contesta en el mismo hilo de WhatsApp. La conversación permanece cerrada para que la respuesta no se interprete como un reclamo nuevo. Se conservó la consulta de estado por WhatsApp ya existente y su contrato de respuesta.

MS-Reclamos publica el evento de cambio de estado también para cierre, nueva visita, revisión rechazada, disconformidad y reapertura. El número se extrae correctamente de claves `whatsapp:phone:<número>` y `web:phone:<número>`. Eventos iniciados por el ciudadano se registran y notifican a los consumidores, pero MS-Notifications no envía una segunda plantilla: MS-AI ya confirma la acción en la conversación.

## Archivos modificados

### MS-Notifications

- `ms-notifications/src/notifications/notifications.service.ts`: prioriza WhatsApp, incluye `rechazado` entre estados notificables y evita segunda plantilla para actor `CITIZEN`.
- `ms-notifications/src/notifications/whatsapp-channel.strategy.ts`: adapta el cuerpo y parámetros del template para respuestas ciudadanas.
- `ms-notifications/src/notifications/notifications.service.spec.ts`: prueba prioridad WhatsApp y omisión de mensaje duplicado.
- `ms-notifications/src/notifications/whatsapp-channel.strategy.spec.ts`: verifica cinco parámetros en el template.
- `ms-notifications/.env.example`: deja `whatsapp,log` como orden y agrega `rechazado`.
- `ms-notifications/README.md`: documenta template y límites de la ventana WhatsApp.

### MS-Reclamos

- `ms-reclamos/src/reclamos/reclamos.service.ts`: publica eventos para transiciones directas y normaliza destinatario telefónico.
- `ms-reclamos/src/reclamos/dto/citizen-response.dto.ts`: permite identificar el reclamo por código más contacto sin exponer el ID al ciudadano.
- `ms-reclamos/src/reclamos/status-notifications.spec.ts`: verifica publicaciones para las principales transiciones y canales.

### MS-AI

- `ms-ai/src/conversation/conversation-state-support.service.ts`: coordina consulta, confirmación, disconformidad y reapertura con contacto validado; conserva statuses conversacionales existentes y normaliza respuestas públicas.
- `ms-ai/src/ai-process/ai-process.service.ts`: intercepta el seguimiento antes de los handlers de creación y guarda la conversación en estado cerrado.
- `ms-ai/src/conversation/conversation-state-support.service.spec.ts`: cubre comandos, contacto inválido, consulta de estado y disconformidad sobre reclamo cerrado.
- `ms-ai/src/ai-process/ai-process.service.spec.ts`: comprueba que el parser/handler normal de reclamo no procese la respuesta ciudadana.

## Configuración de producción

### Servicio `ms-notifications`

- `NOTIFICATION_CHANNELS=whatsapp,log`
- `NOTIFICATION_STATUSES=en_proceso,resuelto,cerrado,rechazado`
- `META_API_VERSION=v25.0`
- `META_WHATSAPP_ACCESS_TOKEN`: secreto del entorno, no versionar ni enviar por chat.
- `META_WHATSAPP_PHONE_NUMBER_ID`: identificador de número Business.
- `META_WHATSAPP_TEMPLATE_NAME=smartservice_claim_status` o el nombre aprobado en Meta.
- `META_WHATSAPP_TEMPLATE_LANGUAGE=es_AR`

La plantilla debe aprobarse en Meta Business Manager con cinco parámetros en el orden documentado en el archivo `ms-notifications/README.md`. Sin aprobación/credenciales no se probó el envío real.

### Servicio `ms-reclamos`

- `NATS_SERVERS`: conectar al mismo NATS que MS-AI, Gateway y Notifications.
- `DATABASE_URL`: su PostgreSQL de Notifications/Reclamos según corresponda al servicio.

### Servicio `ms-ai`

- `NATS_SERVERS`: conectar a los topics de Reclamos.
- `REDIS_URL`: persistencia temporal del estado conversacional.
- `AI_CONVERSATION_TTL_SECONDS`: TTL del contexto, si se requiere conservar el hilo por más de su valor por defecto.

Guardar variables por servicio en los `.env` locales no versionados durante desarrollo y en el gestor de secretos/variables de cada servicio desplegado en producción. No poner tokens en el frontend.

## Validación ejecutada

- MS-AI: 11 suites y 58 pruebas aprobadas; `npm run build` aprobado.
- MS-Reclamos: 4 suites y 16 pruebas aprobadas; 1 prueba de integración con DB omitida; build aprobado.
- MS-Notifications: 2 suites y 7 pruebas aprobadas; 1 integración NATS/DB omitida; build aprobado.

La salida de WhatsApp fue simulada mediante pruebas; no se enviaron mensajes reales.

## Próximo paso

Configurar y aprobar la plantilla en Meta y ejecutar una prueba end-to-end en un entorno aislado con NATS y PostgreSQL. Verificar recepción del mensaje de estado, delivery por WhatsApp, respuesta del ciudadano, validación código/contacto, registro de historial y ausencia de una segunda notificación automática.
