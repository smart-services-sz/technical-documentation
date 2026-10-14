# Implementación 04: evidencias privadas S3/Wasabi

**Fecha:** 14 de octubre de 2026
**Hallazgo de origen:** el sistema no tenía carga real ni almacenamiento externo de evidencias.
**Estado:** implementado y validado; no se conectó un bucket ni una base de datos.

## Resumen

El alta web ahora admite hasta 3 archivos de 5 MB cada uno. Gateway verifica las firmas de contenido y solo acepta JPEG, PNG, WebP y PDF, sin confiar en el MIME o la extensión informados por el navegador. Los archivos se almacenan en un bucket S3 privado, con nombres de objeto aleatorios bajo `evidencias/<municipio>/<correlationId>/`.

MS-Reclamos ya admite `metadata` JSON, así que se guardan allí las claves privadas y metadatos de cada objeto, sin migración. Si falla la persistencia del reclamo o no se recibe su código de seguimiento, Gateway intenta borrar los archivos recién subidos. También limpia cargas parciales cuando S3 falla.

Las APIs de listado eliminan las claves S3 de las respuestas. El detalle autenticado, después de aplicar el ámbito municipal existente, entrega enlaces firmados con vigencia de 5 minutos. El dashboard incorpora una acción “Detalle” y permite descargar evidencias usando esos enlaces. El bucket nunca se hace público.

## Archivos modificados

### Gateway y almacenamiento

- `gateway/package.json` y `gateway/package-lock.json`: agregan AWS SDK S3, presign y tipos de Multer.
- `gateway/src/reclamos/evidence-storage.service.ts`: valida formato por bytes, límites, upload, limpieza, firma de descarga y cierre del cliente S3.
- `gateway/src/reclamos/evidence-storage.service.spec.ts`: cubre upload, detección de MIME falso, limpieza parcial y URL firmada.
- `gateway/src/reclamos/reclamos.controller.ts`: configura multipart para un máximo de 3 evidencias de 5 MB.
- `gateway/src/reclamos/reclamos.service.ts`: sube y asocia evidencia con el reclamo, hace cleanup compensatorio, firma descargas del detalle y sanitiza claves en listados.
- `gateway/src/reclamos/reclamos.module.ts`: registra el servicio de almacenamiento.
- `gateway/src/reclamos/reclamos.controller.spec.ts`: verifica el paso de archivos desde el endpoint.
- `gateway/src/reclamos/reclamos.service.spec.ts`: verifica links firmados, ocultamiento de claves y cleanup si falta el código.
- `gateway/src/reclamos/public-claim-submission.spec.ts`: cubre asociación y rollback compensatorio de evidencias.
- `gateway/.env.example`: lista configuración S3 de ejemplo sin credenciales.

### Sitio y dashboard

- `public-site/index.html`: agrega selector múltiple de evidencia al formulario.
- `public-site/app.js`: envía `FormData`, valida cantidad/tamaño en la interfaz y mantiene la respuesta segura.
- `admin-dashboard/src/services/reclamos.service.ts`: agrega el tipo de evidencia del detalle.
- `admin-dashboard/src/components/reclamos-panel.tsx`: incorpora detalle del reclamo y enlaces de descarga.
- `admin-dashboard/src/components/reclamos-panel.test.tsx`: verifica que el detalle muestre el enlace firmado.

## Configuración requerida

Configurar en Gateway, usando credenciales reales solo en el gestor de secretos del entorno:

- `S3_ENDPOINT`
- `S3_REGION`
- `S3_BUCKET`
- `S3_ACCESS_KEY_ID`
- `S3_SECRET_ACCESS_KEY`
- `S3_FORCE_PATH_STYLE` según el proveedor

El bucket debe ser privado. La política del proveedor debe permitir al Gateway `PutObject`, `GetObject` para firmar descargas y `DeleteObject` para cleanup; no requiere acceso público.

## Validación ejecutada

- Gateway: 4 suites enfocadas — 23 pruebas aprobadas.
- Gateway: `npm run build` — aprobado.
- Admin Dashboard: 5 archivos — 25 pruebas aprobadas.
- Admin Dashboard: typecheck — aprobado.
- Admin Dashboard: `npm run build` — aprobado.
- Sitio público: sintaxis e inicialización — aprobadas; formulario y selector revisados en navegador.

No se ejecutó upload real contra Wasabi/S3; las pruebas usan cliente S3 simulado. El alta pública sí requiere `PUBLIC_MUNICIPALITY_ID`, `PUBLIC_MUNICIPALITY_NAME` y configuración válida de S3 cuando se adjuntan archivos.
