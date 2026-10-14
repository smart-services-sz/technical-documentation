# Implementación 08: respuestas rápidas WhatsApp e historial visible

**Fecha:** 14 de octubre de 2026
**Origen:** solicitud para evitar que el ciudadano repita un código largo y poder ver reaperturas/respuestas en el dashboard.
**Estado:** cambios preparados en el workspace y validados; Meta debe tener aprobado el template v2 para producción.

## Resumen

La notificación WhatsApp mantiene un cuerpo breve con código y estado y agrega tres botones quick reply: confirmar resolución, indicar disconformidad y solicitar reapertura. Cada botón lleva un payload controlado por la aplicación con código y acción, por ejemplo `smartservice|REC-ABC123|reopen`.

El webhook acepta mensajes Meta `type: button`, valida estrictamente ese payload y lo convierte en un comando interno (`REC-ABC123 REABRIR`); payloads no reconocidos se descartan. MS-AI valida código y teléfono de contacto, registra la respuesta/reapertura y contesta en la conversación existente. La disconformidad de un reclamo cerrado conserva el comportamiento previo de reabrirlo.

El detalle del dashboard obtiene el reclamo y su historial de acciones. Muestra etiquetas legibles, fecha, actor, estados anterior/nuevo, tipo de respuesta y motivo; incluye en particular las acciones `claim.reopened` y `citizen.response`.

## Archivos modificados en esta entrega

### WhatsApp

- `ms-notifications/src/notifications/whatsapp-channel.strategy.ts`: cuerpo corto y tres botones quick reply con payload dinámico.
- `ms-notifications/src/notifications/whatsapp-channel.strategy.spec.ts`: valida los payloads de los tres botones.
- `ms-notifications/src/notifications/notifications.service.spec.ts`: alinea la prueba de WhatsApp con el cuerpo breve.
- `ms-notifications/.env.example`: usa el nombre `smartservice_claim_status_v2`.
- `ms-notifications/README.md`: documenta las variables y los tres botones que debe aprobar Meta.
- `gateway/src/webhooks/handlers/dispatch.handler.ts`: convierte callbacks de botones a comandos internos y rechaza payloads desconocidos.
- `gateway/src/webhooks/handlers/dispatch.handler.spec.ts`: prueba confirmación, disconformidad, reapertura y payload inválido.

### Dashboard

- `admin-dashboard/src/components/reclamos-panel.tsx`: carga detalle e historial juntos y traduce acciones técnicas a etiquetas legibles.
- `admin-dashboard/src/components/reclamos-panel.test.tsx`: comprueba que el detalle presenta reapertura, respuesta, motivo y descarga de evidencia.

MS-AI reutiliza el interceptor de respuestas y reaperturas previamente implementado; no hubo cambios de código nuevos en ese servicio para esta entrega.

## Configuración en Meta

El template a configurar es `smartservice_claim_status_v2`, idioma `Spanish (ARG)` / `es_AR`, categoría Utility, con dos variables de cuerpo:

```text
Actualización SmartService: el reclamo {{1}} se encuentra {{2}}. Usa los botones para responder.
```

Agregar tres botones Quick Reply en este orden:

1. `Confirmar resolución`
2. `No se resolvió`
3. `Solicitar reapertura`

El adaptador envía los payloads `smartservice|<trackingCode>|confirm`, `smartservice|<trackingCode>|disagree` y `smartservice|<trackingCode>|reopen`, respectivamente. Configurar `META_WHATSAPP_TEMPLATE_NAME=smartservice_claim_status_v2` en MS-Notifications después de que Meta apruebe la plantilla. No se debe modificar el template anterior de dos variables mientras el nuevo esté pendiente.

## Validación ejecutada

- Gateway: 4 pruebas de webhook aprobadas; build aprobado.
- MS-Notifications: 7 pruebas de servicio/adaptador aprobadas; build aprobado.
- Admin Dashboard: 26 pruebas aprobadas; typecheck y build aprobados.
- El flujo conversacional de MS-AI ya había pasado 9 pruebas enfocadas y build durante la implementación previa.

No se envió una plantilla real a Meta; el envío de producción depende de la aprobación de la plantilla v2 y de sus credenciales.
