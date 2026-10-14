# Implementación 01: aislamiento territorial de reclamos

**Fecha:** 14 de octubre de 2026
**Hallazgo de origen:** acceso potencial de usuarios a reclamos de otros municipios/áreas.
**Estado:** implementado y validado con pruebas unitarias y builds; no se usó una base de datos.

## Resumen

El Gateway ahora deriva el ámbito de municipio y área desde el JWT autenticado, sobrescribe los filtros territoriales de la query y lo propaga en las operaciones administrativas por reclamo. MS-Reclamos aplica el ámbito recibido en sus consultas y antes de cambios o eliminaciones. Un registro fuera del ámbito se responde como no encontrado, sin revelar si existe en otro municipio.

Los usuarios `AGENT` deben tener municipio y área configurados; si falta alguno, el Gateway rechaza la operación. Los usuarios `ADMIN` con municipio quedan limitados a ese municipio; el administrador de plataforma sin ámbito territorial puede continuar operando con los filtros explícitos que envíe.

## Archivos modificados

### Gateway

- `gateway/src/reclamos/reclamos.controller.ts`: aplica el ámbito del JWT a listado, detalle, historial, revisión, nueva visita, cierre, actualización y eliminación.
- `gateway/src/reclamos/reclamos.service.ts`: propaga municipio/área en los mensajes NATS para las operaciones por reclamo.
- `gateway/src/reclamos/reclamos.controller.spec.ts`: añade cobertura para filtros falsificados, ámbito faltante, consultas por ID, mutaciones y administrador de plataforma.

### MS-Reclamos

- `ms-reclamos/src/common/dto/territorial-scope.dto.ts`: define los campos opcionales y validados del ámbito territorial en mensajes internos.
- `ms-reclamos/src/reclamos/dto/reclamo-id.dto.ts`: agrega ámbito al contrato de lectura, historial y eliminación por ID.
- `ms-reclamos/src/reclamos/dto/review-resolution.dto.ts`: agrega ámbito a la revisión de resolución.
- `ms-reclamos/src/reclamos/dto/request-new-visit.dto.ts`: agrega ámbito a la solicitud de nueva visita.
- `ms-reclamos/src/reclamos/dto/close-reclamo.dto.ts`: agrega ámbito al cierre.
- `ms-reclamos/src/reclamos/dto/update-reclamo.dto.ts`: agrega ámbito al comando de actualización.
- `ms-reclamos/src/reclamos/reclamos.controller.ts`: pasa el ámbito de los DTO a las operaciones correspondientes.
- `ms-reclamos/src/reclamos/reclamos.service.ts`: filtra el listado, conteo, detalle e historial; valida alcance antes de revisar, solicitar visitas, cerrar, actualizar o eliminar.
- `ms-reclamos/src/reclamos/reclamos.service.spec.ts`: verifica filtros de listado/conteo, ocultamiento de detalle e historial y bloqueo de cambios fuera de ámbito.

## Validación ejecutada

- `gateway`: `npm test -- --runInBand reclamos.controller.spec.ts` — 5 pruebas aprobadas.
- `ms-reclamos`: `npm test -- --runInBand reclamos.service.spec.ts` — 8 pruebas aprobadas.
- `gateway`: `npm run build` — aprobado.
- `ms-reclamos`: `npm run build` — aprobado.

## Límite del alcance

El ámbito protege las operaciones administrativas HTTP que pasan por Gateway. Los mensajes internos NATS que omiten el ámbito conservan el comportamiento anterior para mantener compatibilidad con consumidores de servicio confiables; NATS debe permanecer en una red privada y los consumidores internos que actúen por cuenta de un usuario deben propagar su contexto territorial. Las rutas públicas de creación y seguimiento no se modificaron en esta entrega.
