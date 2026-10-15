# Implementación 11: configuración municipal

**Fecha:** 15 de octubre de 2026  
**Hallazgo de origen:** faltaba administrar municipios y sus catálogos locales desde el panel.  
**Estado:** implementado y validado localmente; migración creada pero no aplicada.

## Resumen

Se agregó un catálogo municipal administrable desde el dashboard. Cada municipio permite configurar país, provincia, ciudad predeterminada, zona horaria, cobertura geográfica, áreas, categorías, barrios y reglas de negocio en JSON. El acceso HTTP está restringido al rol `ADMIN`.

La configuración vive en MS-Auth, que ya administra usuarios y sus identificadores municipales. El Gateway expone la API protegida y el dashboard utiliza Route Handlers autenticados para listar, crear y actualizar configuraciones. Los cambios del agregado y sus catálogos se guardan en una transacción; los elementos se desactivan en lugar de borrarse y una colección omitida no provoca eliminaciones implícitas. MS-Auth publica auditoría con el municipio y conteos de elementos, sin incluir los valores de las reglas.

La migración es aditiva y mantiene los identificadores municipales existentes como strings, sin incorporar relaciones obligatorias con datos legacy. No se ejecutó ninguna conexión a una base de datos ni se aplicó la migración.

## Archivos modificados

- `ms-auth/prisma/schema.prisma`: modelos `Municipality`, `MunicipalityArea`, `MunicipalityCategory`, `MunicipalityNeighborhood` y `MunicipalityRule`.
- `ms-auth/prisma/migrations/20261015120000_add_municipal_configuration/migration.sql`: creación aditiva de tablas, claves e índices; pendiente de despliegue.
- `ms-auth/src/municipalities/`: DTOs, controller NATS, módulo, servicio y pruebas de catálogo, transacción y auditoría.
- `ms-auth/src/app.module.ts`: registro del módulo municipal.
- `gateway/src/municipal-config/`: DTOs, proxy NATS, endpoints HTTP y prueba de autorización/contexto.
- `gateway/src/app.module.ts`: registro del módulo del Gateway.
- `admin-dashboard/src/services/municipal-config.service.ts`: tipos y acceso a la API municipal.
- `admin-dashboard/src/app/api/municipalities/config/`: Route Handlers autenticados para listar, crear, consultar y actualizar.
- `admin-dashboard/src/components/municipal-config-panel.tsx`: panel para administrar municipios y catálogos.
- `admin-dashboard/src/components/municipal-config-panel.test.tsx`: prueba de alta municipal y persistencia de un área.
- `admin-dashboard/src/components/admin-dashboard.tsx`: navegación al panel para administradores.
- `admin-dashboard/src/components/admin-dashboard.test.tsx`: prueba de navegación al panel municipal.

## Validación ejecutada

- MS-Auth: `prisma validate` y `prisma generate` aprobados; prueba del servicio municipal (3 pruebas) y build aprobados.
- Gateway: pruebas del controller municipal (2 pruebas) y build aprobados.
- Admin Dashboard: 6 archivos y 28 pruebas aprobados; typecheck y build de producción aprobados.
- `git diff --check`: sin errores de whitespace en los repositorios revisados.
- Suites completas: MS-Auth registra 6 suites fallidas y Gateway 6 suites fallidas por tests existentes de Auth/Roles/Permissions que no proveen dependencias requeridas (`PrismaService` y/o `NATS_CLIENT`). Las pruebas específicas de esta entrega pasan; esos fixtures quedan fuera de alcance.

## Alcance pendiente

Este corte administra el catálogo; todavía no lo consume el alta pública ni MS-AI, y no particiona las reglas de ruteo por municipio. La migración debe desplegarse mediante el proceso autorizado del entorno correspondiente antes de usar la configuración en una base persistente.
