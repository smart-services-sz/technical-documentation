# Implementación 03: alta ciudadana desde el sitio web

**Fecha:** 14 de octubre de 2026
**Hallazgo de origen:** la tesis incluye formulario web en el MVP; el sitio no enviaba reclamos.
**Estado:** implementado y validado; no se usó una base de datos.

## Resumen

Se agregó `POST /api/reclamos/public` en Gateway. El endpoint limita envíos a 5 por IP cada hora, obtiene la localidad configurada del servidor, geocodifica la dirección mediante `geo.geocode-address` y rechaza direcciones no resueltas o que no correspondan a esa localidad. Solo entonces envía `reclamos.create`, con identificador de contacto normalizado, prioridad inicial `media`, municipio y coordenadas generadas en backend.

El sitio público ahora ofrece un formulario con problema, dirección, categoría y teléfono/correo. Al registrarse, presenta el código de seguimiento y un mensaje para conservarlo. La categoría es seleccionada por el ciudadano; esta entrega no aplica clasificación IA al canal web.

## Archivos modificados

- `gateway/src/reclamos/reclamos.controller.ts`: expone el alta pública y pasa la IP.
- `gateway/src/reclamos/reclamos.service.ts`: rate limit, configuración territorial, geocodificación, verificación de localidad y persistencia por NATS.
- `gateway/src/reclamos/dto/public-claim-submission.dto.ts`: valida campos, longitudes, categoría y contacto.
- `gateway/src/reclamos/reclamos.controller.spec.ts`: verifica el paso del formulario y la IP.
- `gateway/src/reclamos/public-claim-submission.spec.ts`: cubre alta válida, geocodificación fallida, localidad distinta y rate limit.
- `gateway/.env.example`: documenta la configuración del municipio/country público sin secretos.
- `public-site/index.html`: agrega navegación, CTA y formulario de reclamo.
- `public-site/app.js`: conecta el envío y muestra el código de forma segura.
- `public-site/styles.css`: adapta controles de textarea y selector al estilo del sitio.

## Configuración

Definir en el entorno de Gateway:

- `PUBLIC_MUNICIPALITY_ID`: identificador del municipio, formato slug.
- `PUBLIC_MUNICIPALITY_NAME`: localidad que Google Geocoding debe devolver.
- `PUBLIC_COUNTRY`: país de geocodificación; por defecto `Argentina`.

El formulario no acepta latitud, longitud, municipio ni prioridad desde el navegador.

## Validación ejecutada

- Gateway: pruebas enfocadas — 11 aprobadas.
- Gateway: `npm run build` — aprobado.
- Sitio: `node --check` y smoke test de inicialización — aprobados.
- Sitio: carga estática revisada en navegador; formulario y navegación aparecen correctamente.

## Límite de esta entrega

El formulario todavía no admite archivos adjuntos. La carga privada S3/Wasabi y su presentación autenticada se registrarán en la siguiente entrega. La integración real con Google y PostgreSQL no se ejecutó porque no se configuraron servicios ni se conectó una base durante esta validación.
