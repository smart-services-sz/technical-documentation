# Implementación 09: posición de variables del template Meta

**Fecha:** 14 de octubre de 2026
**Motivo:** Meta requiere que las variables de template no queden al principio ni al final del cuerpo.
**Estado:** código, parser, prueba y documentación alineados; build y tests aprobados.

## Cuerpo aprobado para crear en Meta

Template `smartservice_claim_status_v2`, categoría Utilidad, idioma Spanish (ARG) / `es_AR`, sin encabezado ni pie:

```text
Actualización SmartService: el reclamo {{1}} se encuentra {{2}}. Usa los botones para responder.
```

- `{{1}}` aparece después de texto fijo.
- `{{2}}` va seguido de texto fijo, por lo que no queda al final.

Muestras ficticias para revisión: `{{1}}` = `REC-TEST01`; `{{2}}` = `resuelto`.

Botones Quick Reply, en orden: `Confirmar resolución`, `No se resolvió`, `Solicitar reapertura`. Los payloads se adjuntan dinámicamente al enviar el template desde MS-Notifications.

## Código alineado

- `ms-notifications/src/notifications/notifications.service.ts`: genera el nuevo cuerpo compacto.
- `ms-notifications/src/notifications/whatsapp-channel.strategy.ts`: extrae código/estado solo con este formato y mantiene los tres payloads rápidos.
- `ms-notifications/src/notifications/whatsapp-channel.strategy.spec.ts` y `notifications.service.spec.ts`: cubren el mensaje actualizado.
- `ms-notifications/README.md`: documenta el cuerpo requerido.

## Validación

- Tests enfocados de WhatsApp/Notifications: 7 aprobados.
- `npm run build` de MS-Notifications: aprobado.

Una vez que Meta apruebe la plantilla, configurar `META_WHATSAPP_TEMPLATE_NAME=smartservice_claim_status_v2` y desplegar MS-Notifications.
