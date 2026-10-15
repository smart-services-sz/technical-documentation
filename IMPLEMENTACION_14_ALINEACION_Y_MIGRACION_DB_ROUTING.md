# Implementacion 14: alineacion y migracion de la base de MS-Routing

**Fecha:** 15 de octubre de 2026

**Estado:** correccion de deriva e aislamiento municipal aplicados y verificados en la base autorizada de MS-Routing. Sin migraciones pendientes ni diferencias fisicas respecto de Prisma.

## Diagnostico previo

Se verifico mediante consultas de solo lectura que la conexion correspondia a MS-Routing (PostgreSQL 18.6). Las 13 tablas operativas estaban vacias: no habia reglas, planes, rutas, intervenciones, historial, reservas ni solicitudes de generacion que requirieran asignacion municipal.

El historial contenia `0_baseline` y `20260903112000_add_routing_action_history` completadas. El checksum registrado de `0_baseline` no coincide con el archivo local; tampoco se encontro coincidencia en las versiones locales revisadas ni normalizando saltos de linea/BOM. No se conoce el SQL original exacto aplicado con ese checksum.

La comparacion del esquema real con Prisma detecto, ademas del cambio municipal previsto, diferencias en dos tablas:

- `RoutingClaimAllocation`: fechas `timestamp with time zone` de precision 6 y defaults de UUID y `updatedAt` que no corresponden al modelo actual.
- `RoutingGenerationRequest`: las mismas diferencias para `planningDate`, fechas de creacion/actualizacion y defaults de UUID/`updatedAt`.

Se consulto al usuario ante esta deriva adicional y eligio revisar y alinear primero, en lugar de aplicar solamente el cambio municipal.

## Correccion aplicada

Se agrego una migracion independiente anterior a la municipal: `20261015130000_align_routing_allocation_generation`.

La migracion convierte las fechas afectadas a `timestamp(3)` con zona de sesion UTC explicita, elimina los defaults de UUID y `updatedAt` administrados por Prisma, conserva el default de creacion y ejecuta los cambios en una transaccion con timeout de locks. No cambia el esquema Prisma para adaptarlo a una deriva accidental ni modifica el SQL del baseline.

Antes del despliegue se volvio a comprobar el destino, la ausencia de filas operativas, el historial sin migraciones fallidas activas y que solo estuvieran pendientes las dos migraciones esperadas. Se ejecuto `prisma migrate deploy` con la conexion autorizada solo en el entorno del proceso, sin guardarla en archivos.

## Resultado verificado

- `20261015130000_align_routing_allocation_generation`: aplicada el `2026-10-15T13:07:00.939Z`.
- `20261015140000_add_municipal_routing_scope`: aplicada el `2026-10-15T13:07:07.001Z`.
- `prisma validate`: aprobado.
- `prisma migrate diff --from-config-datasource --to-schema prisma/schema.prisma --script --exit-code`: aprobado; diff vacio.
- Historial posterior: las cuatro migraciones locales figuran completadas y no hay pendientes.
- Checksums historicos anteriores al despliegue: sin cambios.
- Conteos finales: las 13 tablas operativas siguen con cero filas. No se ejecuto seed, backfill, reset ni borrado de datos.

Al preparar la publicacion se detecto que `core.autocrlf=true` normalizaba a LF los dos archivos SQL ya aplicados con CRLF. Se agregaron atributos `-text` limitados a esas dos migraciones para conservar sus bytes y checksums, sin cambiar el contenido ejecutado ni la configuracion global de Git. Se comprueba la igualdad byte por byte entre el archivo local y el indice antes del commit.

La validacion fue real contra PostgreSQL para estas migraciones; no equivale a una prueba de extremo a extremo de Gateway, NATS, MS-Routing y MS-Reclamos.

## Archivos modificados

- `ms-routing/prisma/migrations/20261015130000_align_routing_allocation_generation/migration.sql`: correccion de tipos/defaults, ordenada antes de la migracion municipal.
- `ms-routing/.gitattributes`: evita normalizar los dos archivos SQL ya aplicados al prepararlos para publicacion.
- `technical-doc/IMPLEMENTACION_13_AISLAMIENTO_MUNICIPAL_ROUTING.md`: actualiza el estado de aplicacion y separa la validacion local inicial de la verificacion remota posterior.
- `technical-doc/REPORTE_AUDITORIA_TESIS_VS_CODIGO_2026-10.md`: refleja el despliegue de las migraciones en la base verificada.

## Observacion historica y activacion pendiente

El esquema fisico esta alineado, pero la discrepancia historica del checksum de `0_baseline` permanece documentada: no se reescribio `_prisma_migrations`, no se uso `migrate resolve` ni se simulo una coincidencia. Para reconciliar ese antecedente debe recuperarse el artefacto SQL original o acordarse un procedimiento auditado. No usar `migrate reset` sobre esta base para resolverlo.

Antes de habilitar el nuevo flujo hay que versionar y desplegar el codigo y ambas migraciones, configurar el municipio de administradores/operadores, renovar sus tokens y registrar las reglas municipales. Como esta base estaba vacia, no requirio backfill; otros entornos deben verificar sus propios datos e historial. No se hizo commit ni push durante esta operacion.