# Implementación 10: sugerencia asistida de posibles duplicados

**Fecha:** 15 de octubre de 2026  
**Hallazgo de origen:** no existía detección de reclamos repetidos, capacidad diferencial mencionada por la tesis.  
**Estado:** implementado y validado; no se modificó el esquema ni se consultó una base real.

## Resumen

Antes de crear un reclamo por el flujo conversacional, MS-AI busca hasta 100 reclamos `pendiente` y 100 `en_proceso` de la misma categoría y municipio. Compara distancia geográfica mediante Haversine y similitud de tokens normalizados; conserva como máximo tres candidatos dentro de 120 metros cuando coincide la dirección o hay coincidencia textual mínima.

Los candidatos se guardan en `metadata.posiblesDuplicados` con código de seguimiento, descripción, dirección, estado, distancia y similitud. MS-Reclamos solo expone campos explícitamente permitidos; no se incluyen contacto ni datos personales. El dashboard los presenta como **“Posibles duplicados: revisar, no se fusionaron”** y permite abrir el detalle del candidato. La creación continúa siempre: no se bloquea ni fusiona automáticamente.

Si el servicio de reclamos no responde durante la consulta de candidatos, MS-AI registra una advertencia y continúa con el alta normal.

## Archivos modificados

- `ms-ai/src/conversation/conversation-state-support.service.ts`: consulta candidatos, limita por categoría/municipio/estado y calcula distancia/similitud.
- `ms-ai/src/conversation/conversation-state-support.service.spec.ts`: cubre sugerencia cercana, exclusión por categoría/distancia y continuidad si falla la búsqueda.
- `ms-reclamos/src/reclamos/dto/reclamo-response.dto.ts`: define el tipo administrativo de candidatos sugeridos.
- `ms-reclamos/src/reclamos/reclamos.service.ts`: valida y proyecta únicamente campos permitidos desde metadata.
- `ms-reclamos/src/reclamos/reclamos.service.spec.ts`: verifica campos retornados y ausencia de datos privados.
- `admin-dashboard/src/services/reclamos.service.ts`: incorpora el tipo `PosibleDuplicado`.
- `admin-dashboard/src/components/reclamos-panel.tsx`: muestra código, distancia y similitud con acción de revisión.
- `admin-dashboard/src/components/admin-dashboard.module.css`: estilos del aviso de duplicidad.
- `admin-dashboard/src/components/reclamos-panel.test.tsx`: comprueba el aviso y la apertura del candidato.
- `technical-doc/REPORTE_AUDITORIA_TESIS_VS_CODIGO_2026-10.md`: actualiza el estado del hallazgo de duplicados.

## Validación ejecutada

- MS-AI: 11 suites y 60 pruebas aprobadas; build aprobado.
- MS-Reclamos: 4 suites y 17 pruebas aprobadas; 1 integración con PostgreSQL omitida; build aprobado.
- Admin Dashboard: 5 archivos y 27 pruebas aprobadas; typecheck y build aprobados.

## Límite conocido

Es una heurística inicial, no un modelo semántico ni una decisión de negocio. Los umbrales (120 m y 18% de similitud textual) deben calibrarse con ejemplos reales y revisión humana. No hay nuevas variables de entorno ni migración requerida.