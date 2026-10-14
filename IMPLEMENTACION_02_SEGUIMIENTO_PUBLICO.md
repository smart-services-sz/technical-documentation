# Implementación 02: seguimiento ciudadano seguro

**Fecha:** 14 de octubre de 2026
**Hallazgo de origen:** la consulta pública enviaba solo el código, mientras MS-Reclamos exigía contacto; el sitio no gestionaba el resultado `not_found` y el código anterior diferenciaba errores por existencia.
**Estado:** implementado y validado; no se usó una base de datos.

## Resumen

El seguimiento ahora usa `POST /api/reclamos/seguimiento` con código y teléfono o correo. MS-Reclamos normaliza el contacto y solo devuelve estado, categoría y fecha de actualización si coincide con el reclamo. Código/contacto inexistentes o no coincidentes producen el mismo resultado público. Se retiró la consulta interna que recuperaba estado usando solo el código.

Gateway aplica un límite de 10 consultas por IP en 15 minutos mediante Redis. La clave de rate limit contiene el hash SHA-256 de la IP, no su valor en claro. Si Redis no está disponible, la consulta falla cerrada con `503`; al superar el límite responde `429`.

El sitio público pide ambos datos, maneja respuestas inválidas y rate limit, y renderiza los valores con `textContent`. La URL del Gateway se configura mediante `<meta name="smartservice-gateway-url">`; si está vacía, usa el mismo origen, salvo en localhost, donde conserva `http://localhost:3000` para desarrollo.

## Archivos modificados

### Gateway

- `gateway/src/reclamos/reclamos.controller.ts`: cambia el endpoint público a `POST`, valida el DTO y pasa la IP al servicio.
- `gateway/src/reclamos/reclamos.service.ts`: aplica rate limit con Redis antes de consultar NATS y falla de forma cerrada si Redis falla.
- `gateway/src/reclamos/dto/public-tracking.dto.ts`: valida el formato del código y acepta teléfono o correo como dato de contacto.
- `gateway/src/reclamos/reclamos.controller.spec.ts`: cubre el paso del par de credenciales y la IP solicitante.
- `gateway/src/reclamos/reclamos.service.spec.ts`: verifica NATS, contador Redis, límite 429 y fallo cerrado 503.

### MS-Reclamos

- `ms-reclamos/src/reclamos/dto/public-tracking.dto.ts`: alinea el mensaje NATS con el contrato de código y contacto.
- `ms-reclamos/src/reclamos/reclamos.service.ts`: normaliza teléfono/correo, compara ambos factores, minimiza la respuesta pública y elimina la consulta antigua por código únicamente.
- `ms-reclamos/src/reclamos/public-tracking.spec.ts`: prueba coincidencia por teléfono normalizado y búsqueda por correo normalizado.

### Sitio público

- `public-site/index.html`: agrega campo obligatorio de teléfono/correo y configuración del Gateway por metadato.
- `public-site/app.js`: envía el nuevo `POST`, muestra mensajes genéricos para datos no coincidentes, trata `429`, configura el origen API y evita insertar contenido con `innerHTML`.

## Validación ejecutada

- Gateway: pruebas de controlador y servicio de reclamos — 9 aprobadas.
- MS-Reclamos: suite completa — 11 aprobadas; 1 prueba de integración omitida porque requiere base de datos.
- Gateway: `npm run build` — aprobado.
- MS-Reclamos: `npm run build` — aprobado.
- Sitio público: `node --check` y smoke test de inicialización con DOM simulado — aprobados.

## Límites conocidos

La comprobación usa conocimiento de un teléfono o correo previamente asociado al reclamo; no prueba posesión actual del canal y no es un OTP. El rate limit es por IP, por lo que producción debe mantener una configuración confiable de proxy inverso para interpretar correctamente la IP del cliente. El endpoint anterior `GET /api/reclamos/seguimiento/:codigo` deja de ser compatible; el sitio del proyecto ya utiliza el nuevo contrato.
