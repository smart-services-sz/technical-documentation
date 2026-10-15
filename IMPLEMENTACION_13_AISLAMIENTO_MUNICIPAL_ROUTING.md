# Implementacion 13: aislamiento municipal de ruteo

**Fecha:** 15 de octubre de 2026

**Estado:** implementado y validado localmente con dobles. La migracion municipal fue aplicada posteriormente en la base autorizada de MS-Routing, tras una correccion previa de deriva; ver la implementacion 14.

## Resumen

Reglas, planes por area, planes generados y solicitudes idempotentes de ruteo ahora tienen propiedad municipal. Gateway toma el municipio del JWT autenticado y lo propaga a MS-Routing, sin permitir que el payload del cliente lo sobrescriba. El rol ADMIN mantiene sus permisos administrativos, pero no habilita una consulta global ni permite elegir otro municipio desde el body.

Todo usuario que opere ruteo, incluido ADMIN, debe tener `municipalityId` configurado. MS-Routing exige el alcance en cada entrada del servicio; un mensaje NATS sin municipio no obtiene un listado global. Los filtros existentes de area se conservan en la seleccion de reclamos.

## Comportamiento modificado

- Reemplazar reglas elimina y recrea solamente las del municipio autenticado; ya no hay `deleteMany` globales. Una misma categoria, cuadrilla o zona puede existir en varios municipios.
- Listados, detalle, confirmacion y eliminacion de planes consultan por ID y municipio. El historial se consulta solo despues de autorizar el acceso al plan.
- Guardar un plan por area con ID requiere que ese plan ya pertenezca al municipio. No se usa un upsert que pueda apropiarse de otro registro. Los planes nuevos se crean sin enviar ID, como ya hace el dashboard.
- Rutas asignadas, cambios de estado, intervenciones y evidencias se filtran mediante el municipio del plan propietario. Un ID extranjero devuelve 404 incluso si el actor es administrador municipal. Se mantienen los controles de operador, orden de trabajos y estados de ruta.
- La seleccion de reclamos envia el municipio a MS-Reclamos y conserva un filtro local defensivo. Los reclamos de otro municipio o sin municipio no entran en la simulacion.
- La clave idempotente se resuelve junto con el municipio: dos municipios pueden reutilizar el mismo texto sin compartir la respuesta ni el plan generado.
- Las evidencias usan el municipio autenticado para su almacenamiento. No existe respaldo `platform` para actores sin municipio. Se conserva la limpieza de objetos si MS-Routing rechaza el adjunto.
- La sincronizacion de intervenciones/evidencias envia el municipio a `reclamos.update`. Se corrigio ademas el consumidor de MS-Reclamos, que recibia el alcance pero no lo pasaba al servicio de actualizacion.

## Esquema y migracion

Archivo: `ms-routing/prisma/migrations/20261015140000_add_municipal_routing_scope/migration.sql`.

Agrega `municipalityId` nullable en `RoutingCategoryRule`, `RoutingCrewRule`, `RoutingZoneRule`, `RoutingAreaPlan`, `RoutingPlan` y `RoutingGenerationRequest`, con indices y unicidades compuestas por municipio. Las zonas incorporan una clave primaria UUID interna `key`; conservan su `id` funcional para que las referencias de zonas de las reglas no cambien.

La migracion es transaccional, no elimina filas ni asigna implicitamente municipios. Los registros existentes quedan con `municipalityId = NULL`; no son visibles ni operables desde las APIs municipales. Las asignaciones y reservas de reclamos conservan su coordinacion por UUID global del reclamo; solo se usan tras seleccionar reclamos o autorizar planes dentro del municipio.

**Importante:** preservar filas no equivale a conservar visibilidad operativa. Antes de aplicar la migracion, revisar reglas y rutas activas, respaldar la base y definir una asignacion explicita y verificada de los datos existentes. No inferir propiedad por nombres ni aplicar el municipio de un entorno a todos los registros. No reutilizar la conexion de MS-Auth: esta migracion corresponde exclusivamente a la base de MS-Routing.

## Archivos modificados

- `ms-routing/prisma/schema.prisma`: propiedad municipal, indices y claves compuestas.
- `ms-routing/prisma/migrations/20261015140000_add_municipal_routing_scope/migration.sql`: migracion aplicada en la base autorizada; verificar por separado otros entornos.
- `ms-routing/src/routing/dto/routing-municipality-scope.dto.ts`: validacion compartida del identificador territorial. La ausencia se rechaza en el servicio; los overrides de reglas anidados no determinan el municipio.
- `ms-routing/src/routing/dto/`: los DTO de reglas, IDs de planes, planes por area, rutas asignadas, estado, intervenciones y evidencias conservan el alcance en la validacion NATS.
- `ms-routing/src/routing/routing.controller.ts`: propaga el alcance de los consumidores NATS al servicio.
- `ms-routing/src/routing/routing.service.ts`: consultas y mutaciones municipales, idempotencia separada y persistencia del municipio en planes nuevos.
- `ms-routing/src/routing/routing.service.spec.ts`: pruebas de dos municipios, exclusion de legacy, acceso directo por ID, frontera NATS y persistencia municipal.
- `gateway/src/routing/routing.controller.ts`: municipio y actor derivados del JWT en todas las operaciones de ruteo.
- `gateway/src/routing/routing.service.ts`: payloads NATS municipales y evidencias sin municipio global de respaldo.
- `gateway/src/routing/routing.controller.spec.ts`: controles de administradores/operadores y rechazo de municipio suministrado por cliente.
- `gateway/src/routing/routing.service.spec.ts`: propagacion municipal administrativa y controles de evidencias.
- `ms-reclamos/src/reclamos/reclamos.controller.ts`: pasa el alcance de `reclamos.update` al servicio.
- `ms-reclamos/src/reclamos/reclamos.service.spec.ts`: cubre la propagacion del alcance desde el consumidor NATS.
- `technical-doc/REPORTE_AUDITORIA_TESIS_VS_CODIGO_2026-10.md`: estado actualizado del hallazgo municipal.

## Validacion ejecutada

- Prisma: schema valido, formato aplicado y cliente generado. Los indices que se reemplazan se contrastaron con la migracion baseline.
- MS-Routing: 5 suites y 41 pruebas aprobadas; build aprobado.
- Gateway: 6 suites enfocadas y 50 pruebas aprobadas; build aprobado. No se reejecutaron las suites genericas con fallos de inyeccion de dependencias ya documentados.
- MS-Reclamos: 4 suites y 18 pruebas aprobadas; 1 prueba de integracion omitida; build aprobado.
- Pruebas cruzadas con dobles: listados de dos municipios sin legacy; detalle/historial ajenos bloqueados; confirmacion, eliminacion y sobrescritura ajenas rechazadas; acceso por ID a rutas/intervenciones filtrado; misma clave idempotente con respuestas municipales independientes; reclamos extranjeros y legacy excluidos aun si el servicio remoto los retorna.
- `git diff --check`: sin errores de whitespace en los servicios revisados.
- En la validacion local inicial no se ejecuto la migracion porque Docker no estaba activo. Posteriormente se aplico en la base remota autorizada y se verifico un diff vacio contra Prisma; los detalles estan en [la implementacion 14](IMPLEMENTACION_14_ALINEACION_Y_MIGRACION_DB_ROUTING.md). Sigue pendiente la prueba integrada del circuito con NATS y los servicios.

## Activacion y otros entornos

La base autorizada ya tiene las migraciones aplicadas y estaba vacia, por lo que no requirio asignacion de datos legacy. Las precauciones siguientes siguen siendo necesarias para activar el codigo y para cualquier otro entorno.

1. Confirmar la base de MS-Routing y su historial; ensayar la migracion en una copia de prueba con datos representativos.
2. Revisar la titularidad de registros existentes y preparar el backfill aprobado o el reingreso de reglas por municipio. Evaluar especialmente las rutas activas y reservas antes de habilitar operaciones.
3. Asignar municipio a administradores y operadores en MS-Auth y renovar sus tokens. Los administradores de plataforma sin municipio no tienen un bypass global de ruteo.
4. Coordinar la migracion y las versiones de Gateway, MS-Routing y MS-Reclamos durante una ventana sin operaciones de ruteo; no mezclar consumidores viejos que omiten alcance con productores nuevos.
5. Antes de reanudar, verificar con dos municipios reales que cada uno ve solo sus planes/reglas y que un ID ajeno no puede consultarse ni modificarse.

No se crean nuevas variables de entorno. El catalogo municipal y las reglas JSON de configuracion no reemplazan el algoritmo ni introducen validacion automatica de pertenencia de cuadrillas en MS-Auth; este corte impone propiedad y autorizacion de los datos de ruteo. La prueba integrada de extremo a extremo sigue pendiente; el tratamiento de legacy debe revisarse en cada entorno que contenga datos.