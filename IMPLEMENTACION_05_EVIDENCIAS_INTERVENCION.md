# Implementación 05: evidencias de intervenciones operativas

**Fecha:** 14 de octubre de 2026
**Hallazgo de origen:** el endpoint de intervención aceptaba cualquier URL y no controlaba el acceso al archivo.
**Estado:** implementado y validado con S3 simulado; no se conectó un bucket real ni una base de datos.

## Resumen

El agente ahora adjunta un archivo a una intervención desde su parada de ruta. Gateway valida primero que la intervención aparezca en una ruta asignada al usuario; MS-Routing conserva una segunda comprobación de pertenencia al persistir. Solo después de esa autorización se sube el archivo a S3 privado.

MS-Routing recibe y persiste una clave `objectKey` validada, ya no una URL arbitraria. Gateway firma lecturas de rutas asignadas, ruta individual, planes y la respuesta de adjunto; los enlaces caducan a los 5 minutos. Las observaciones del reclamo registran nombre, tipo y fecha, pero no incluyen claves S3.

El proxy Next reenvía el cuerpo `multipart/form-data` como bytes y conserva su boundary. El cliente API ya no impone `Content-Type: application/json` cuando envía `FormData`.

## Archivos modificados

### Gateway

- `gateway/src/storage/evidence-storage.module.ts`: módulo compartido de almacenamiento.
- `gateway/src/reclamos/reclamos.module.ts`: importa el módulo compartido.
- `gateway/src/reclamos/evidence-storage.service.ts`: añade detección MP4 y permite firmar evidencia operativa; el cliente S3 se cierra al apagar Gateway.
- `gateway/src/routing/routing.module.ts`: importa el almacenamiento compartido.
- `gateway/src/routing/routing.controller.ts`: recibe multipart y deriva actor/municipio del JWT.
- `gateway/src/routing/dto/attach-intervention-evidence.dto.ts`: limita el body a ID y descripción; no acepta URL, actor ni rol desde el cliente.
- `gateway/src/routing/routing.service.ts`: comprueba asignación antes de subir, limpia objetos si MS-Routing falla y firma URLs en respuestas autorizadas.
- `gateway/src/routing/routing.controller.spec.ts`: valida contexto autenticado y archivos en el endpoint.
- `gateway/src/routing/routing.service.spec.ts`: cubre rechazo previo a S3, upload, cleanup y firma de lecturas.
- `gateway/src/reclamos/evidence-storage.service.spec.ts`: añade cobertura para contenido MP4.

### MS-Routing

- `ms-routing/src/routing/dto/attach-intervention-evidence.dto.ts`: reemplaza `urlArchivo` de entrada por `objectKey` con patrón restringido.
- `ms-routing/src/routing/routing.service.ts`: persiste la clave en la columna existente y evita copiarla a observaciones del reclamo. No se creó una migración.

### Admin Dashboard y transporte multipart

- `admin-dashboard/src/lib/api-client.ts`: no fuerza `Content-Type` JSON para `FormData`.
- `admin-dashboard/src/lib/gateway-auth-proxy.ts`: reenvía multipart como `ArrayBuffer`, sin decodificar ni corromper contenido binario.
- `admin-dashboard/src/app/api/routing/interventions/evidence/route.ts`: crea el proxy autenticado hacia Gateway.
- `admin-dashboard/src/services/routing.service.ts`: agrega tipos de evidencia y método multipart.
- `admin-dashboard/src/components/agent-routes-panel.tsx`: permite adjuntar y descargar evidencias en paradas con intervención registrada.
- `admin-dashboard/src/lib/gateway-auth-proxy.test.ts`: prueba preservación de bytes y boundary.
- `public-site/index.html`: permite también seleccionar videos MP4 en evidencia ciudadana.

## Validación ejecutada

- Gateway: 3 suites de Routing/Storage — 16 pruebas aprobadas.
- Gateway: `npm run build` — aprobado.
- MS-Routing: 5 suites — 31 pruebas aprobadas.
- MS-Routing: `npm run build` — aprobado.
- Admin Dashboard: 5 archivos — 26 pruebas aprobadas.
- Admin Dashboard: typecheck y `npm run build` — aprobados.
- Proxy multipart: 5 pruebas aprobadas, incluyendo preservación byte a byte.

Las operaciones de S3 se probaron con cliente simulado. No se validó conectividad, permisos ni URLs contra Wasabi real.

## Datos de Wasabi requeridos

Configurar estas variables **en Gateway**, en `gateway/.env` para desarrollo local o en el gestor de variables/secretos del servicio Gateway desplegado. No colocarlas en el dashboard ni en `public-site`:

- `S3_ENDPOINT`: endpoint HTTPS de Wasabi correspondiente a la región del bucket.
- `S3_REGION`: región del bucket.
- `S3_BUCKET`: nombre exacto del bucket privado.
- `S3_ACCESS_KEY_ID`: access key creada en Wasabi.
- `S3_SECRET_ACCESS_KEY`: secret key asociada; mantenerla solo en el entorno Gateway.
- `S3_FORCE_PATH_STYLE=false`: valor recomendado para endpoint Wasabi estándar.

La política del access key debe permitir `s3:PutObject`, `s3:GetObject` y `s3:DeleteObject` únicamente en el bucket/prefijo usado por SmartService. El bucket debe permanecer privado; no requiere ACL pública ni CORS para el flujo actual, porque las cargas pasan por Gateway y las descargas usan navegación a una URL firmada.

Las variables `PUBLIC_MUNICIPALITY_ID`, `PUBLIC_MUNICIPALITY_NAME` y `PUBLIC_COUNTRY` existentes también deben estar definidas en Gateway para el alta pública.

## Límite conocido

Los registros históricos que ya contengan URLs externas en `urlArchivo` se siguen devolviendo con compatibilidad anterior; las nuevas operaciones solo aceptan y guardan claves S3. No se migraron registros existentes. El endpoint actual admite hasta un archivo de 5 MB por intervención.
