# Implementación 06: notificaciones WhatsApp y respuestas conversacionales

**Fecha:** 14 de octubre de 2026
**Hallazgos de origen:** WhatsApp no era el canal predeterminado; varias transiciones directas no publicaban eventos; el estado conversacional tenía helpers de seguimiento/respuesta sin conectar al flujo entrante.
**Estado:** implementado y validado con servicios simulados; no se conectó Meta, NATS productivo ni una base de datos.

## Resumen

- WhatsApp queda como canal prioritario por defecto para eventos con teléfono (`NOTIFICATION_CHANNELS=whatsapp,log`). Las notificaciones contemplan `en_proceso`, `resuelto`, `cerrado` y `rechazado`.
- El teléfono se extrae correctamente de claves `whatsapp:phone:<número>` y `web:phone:<número>`, por lo que los reclamos creados desde el formulario web también pueden recibir novedades por WhatsApp si el ciudadano proporcionó un teléfono.
- Las transiciones que antes omitían evento ahora publican `reclamos.v1.status.changed`: cierre, nueva visita, rechazo de revisión, reapertura y disconformidad ciudadana, además de la actualización genérica que ya lo hacía.
- El template incluye instrucciones de respuesta. El ciudadano puede responder con el código y `CONFORME`, `DISCONFORME: motivo` o `REABRIR: motivo`.
- MS-AI intercepta esos comandos antes del parser de reclamos, valida código más teléfono/correo por MS-Reclamos, registra confirmación/disconformidad o reapertura y responde por la conversación de WhatsApp existente. El estado Redis vuelve a `closed`; un mensaje ajeno a ese formato continúa por el flujo normal.
- Los eventos iniciados por el propio ciudadano se conservan para trazabilidad, pero Notifications omite una segunda plantilla; la confirmación de la acción se devuelve dentro de la conversación entrante.

## Archivos modificados

### MS-Reclamos

- `ms-reclamos/src/reclamos/reclamos.service.ts`: publica los eventos de transiciones directas, usa destinatario telefónico de cualquier canal y etiqueta cambios de respuesta/reapertura como iniciados por `CITIZEN`.
- `ms-reclamos/src/reclamos/dto/citizen-response.dto.ts`: permite asociar la respuesta por `trackingCode` más `contactKey`, sin devolver el ID interno al ciudadano.
- `ms-reclamos/src/reclamos/status-notifications.spec.ts`: prueba eventos para disconformidad, reapertura, revisión rechazada, nueva visita y cierre.

### MS-Notifications

- `ms-notifications/src/notifications/notifications.service.ts`: prioriza WhatsApp, notifica estados configurados y omite una segunda notificación para eventos `CITIZEN`.
- `ms-notifications/src/notifications/whatsapp-channel.strategy.ts`: envía los parámetros del template, incluido el código requerido para responder.
- `ms-notifications/src/notifications/notifications.service.spec.ts`: verifica canal WhatsApp por defecto y ausencia de notificación duplicada.
- `ms-notifications/src/notifications/whatsapp-channel.strategy.spec.ts`: verifica los cinco parámetros del template.
- `ms-notifications/.env.example`: establece WhatsApp primero y agrega estado `rechazado`.
- `ms-notifications/README.md`: documenta el cuerpo y aprobación necesaria del template.

### MS-AI

- `ms-ai/src/conversation/conversation-state-support.service.ts`: valida consultas y comandos ciudadanos, extrae contacto de la clave canónica y llama a respuesta/reapertura por NATS.
- `ms-ai/src/ai-process/ai-process.service.ts`: intercepta la interacción antes de ejecutar handlers de creación de reclamos y persiste conversación cerrada.
- `ms-ai/src/conversation/conversation-state-support.service.spec.ts`: prueba confirmación, disconformidad, reapertura, dato de contacto incorrecto y mensajes no relacionados.
- `ms-ai/src/ai-process/ai-process.service.spec.ts`: prueba que una respuesta ciudadana no entre al parser/handler de alta.

## Configuración del template WhatsApp

Mantener `NOTIFICATION_CHANNELS=whatsapp,log` y `NOTIFICATION_STATUSES=en_proceso,resuelto,cerrado,rechazado`. En Meta Business Manager debe aprobarse `smartservice_claim_status` (o el valor de `META_WHATSAPP_TEMPLATE_NAME`) con idioma `es_AR` y cinco parámetros, según el cuerpo documentado en `ms-notifications/README.md`. `META_WHATSAPP_ACCESS_TOKEN`, `META_WHATSAPP_PHONE_NUMBER_ID` y el nombre del template se configuran solo en el entorno de `ms-notifications`; no compartir tokens por chat ni versionarlos.

## Validación ejecutada

- MS-AI: 11 suites, 56 pruebas aprobadas.
- MS-AI: `npm run build` — aprobado.
- MS-Reclamos: 4 suites aprobadas, 16 pruebas; integración con PostgreSQL omitida.
- MS-Reclamos: `npm run build` — aprobado.
- MS-Notifications: 2 suites aprobadas, 7 pruebas; integración NATS/PostgreSQL omitida.
- MS-Notifications: `npm run build` — aprobado.

Las pruebas simulan NATS y Meta; no se envió ningún mensaje real.

## Límite conocido

El ciudadano debe incluir el código de seguimiento en su respuesta para que el flujo valide su identidad por contacto y reclamo. El template nuevo debe estar aprobado por Meta y configurado en producción antes de activar envíos; sin ello, el adaptador registrará intentos fallidos y aplicará reintentos.
