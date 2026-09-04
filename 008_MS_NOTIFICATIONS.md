# MS Notifications

Microservicio correspondiente al Incremento 5 de CORE 4.

## Responsabilidades

- Consumir el evento versionado `reclamos.v1.status.changed`.
- Notificar solamente estados configurados como comunicables.
- Evitar duplicados mediante `idempotencyKey`.
- Registrar cada intento de entrega en `NotificationAttempt`.
- Aplicar reintentos limitados mediante `RetryPolicy`.
- Seleccionar el canal mediante `NotificationChannelStrategy`.
- Mantener separado el resultado de entrega del cambio principal del reclamo.

## Persistencia

La persistencia es propia del microservicio:

- `Notification`: estado de la notificacion e idempotencia.
- `NotificationAttempt`: resultado de cada intento.

No se ejecutan migraciones al iniciar el proceso. Usar `npx prisma migrate deploy` con el `DATABASE_URL` correspondiente cuando exista una base provisionada.

## Evento consumido

Patron NATS: `reclamos.v1.status.changed`

Payload minimo:

```json
{
  "version": "1",
  "correlationId": "uuid",
  "idempotencyKey": "claim-id-status",
  "occurredAt": "2026-09-04T00:00:00.000Z",
  "producer": "ms-reclamos",
  "actor": {},
  "claimId": "uuid",
  "trackingCode": "REC-XXXX",
  "newStatus": "resuelto",
  "recipientPhone": "+549..."
}
```

## Canales

- `log`: canal seguro para desarrollo y pruebas.
- `whatsapp`: adaptador Meta Graph reutilizable con las variables de Gateway.

Configurar `NOTIFICATION_CHANNELS=log,whatsapp` para habilitar la estrategia WhatsApp cuando exista destinatario y credencial.

## Variables

Ver `ms-notifications/.env.example`.

- `NATS_SERVERS`
- `DATABASE_URL`
- `NOTIFICATION_CHANNELS`
- `NOTIFICATION_STATUSES`
- `NOTIFICATION_MAX_ATTEMPTS`
- `NOTIFICATION_RETRY_BASE_MS`
- `META_API_VERSION`
- `META_WHATSAPP_ACCESS_TOKEN`
- `META_WHATSAPP_PHONE_NUMBER_ID`