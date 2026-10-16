# Implementacion 17: MS-Audit y almacenamiento central de eventos

**Fecha:** 16 de octubre de 2026  
**Estado:** servicio, consulta y migracion implementados; esquema aplicado y smoke test real de escritura/lectura verificado en la base Railway proporcionada. Push pendiente de esta entrega; no se pudo confirmar un deploy Railway desde las herramientas disponibles.

**Correcciones de build Railway (16/10/2026):** el primer build detecto Node 18.20.5, incompatible con Prisma 7.10.1; se fijo Node 22.22.2 mediante `.node-version` y `engines.node`, y Prisma CLI quedo como dependencia de runtime para migrate deploy. Un segundo intento reporto `EBUSY` porque el `npm ci` duplicado dentro de `buildCommand` intentaba borrar `/app/node_modules/.cache`, montado como cache por Railway. Nixpacks ya ejecuta la instalacion en su etapa previa; por eso el `buildCommand` queda solo con `prisma generate` y `nest build`. Se valido localmente esa secuencia con Node 22.22.2.

## Entrega

Se creo `ms-audit` como microservicio NestJS/Prisma 7 para recibir `audit.log` por NATS y almacenar eventos en PostgreSQL. Se adapto la referencia de `C:\actas-be\ms-audit` al patron `@prisma/adapter-pg` usado por SmartService; no se copio su configuracion local ni se leyo su `.env`.

El modelo conserva los campos de auditoria de MS-Auth y agrega `eventId`, `municipalityId` y `correlationId`. `eventId` es opcional y unico para compatibilidad; MS-Auth ahora emite un UUID por evento y el consumidor usa upsert para deduplicar reenvios con el mismo ID. La metadata se redacciona recursivamente por nombres de claves que aparentan contener secretos y se limita a 32 KB.

Gateway expone `GET /api/audit/logs`, con rol `ADMIN`, filtros y paginacion de hasta 100 registros. No se agrego todavia una vista React en el dashboard.

## Base y migracion

La conexion recibida se verifico en solo lectura antes del DDL: PostgreSQL 18.6, base `railway`, schema `public` vacio y sin `_prisma_migrations`. No se levanto base local ni se escribio el connection string a archivos.

Se aplico `20261016110000_init_audit`. `prisma migrate deploy` termino correctamente y `prisma migrate diff --from-config-datasource --to-schema prisma/schema.prisma --script` devolvio diff vacio. Se ejecuto ademas un smoke test transaccional de insert/lectura Prisma y rollback; quedo cero fila sintetica y cero eventos almacenados al finalizar. No se ejecuto seed.

## Archivos

- `ms-audit/package.json`, `package-lock.json`, `nest-cli.json`, `tsconfig*.json`, `prisma.config.ts`: servicio y Prisma 7.
- `ms-audit/prisma/schema.prisma` y `prisma/migrations/20261016110000_init_audit/migration.sql`: modelo, filtros indexados y migracion inicial.
- `ms-audit/src/main.ts`, `app.module.ts`, `prisma/`: runtime NATS y conexion Prisma.
- `ms-audit/src/audit/`: DTOs, consumidor `audit.log`, query `audit.get.logs`, persistencia idempotente, redaccion y pruebas.
- `ms-audit/railway.toml`, `.env.example`, `.gitignore`, `README.md`: build/start/migracion de Railway y configuracion sin secretos versionados.
- `ms-audit/.node-version` y `engines.node`: runtime compatible con Prisma 7; Prisma CLI queda disponible para migrate deploy en start.
- `gateway/src/audit/` y `gateway/src/app.module.ts`: endpoint de consulta ADMIN y filtros.
- `ms-auth/src/utils/audit.publisher.ts` y `.spec.ts`: `eventId` UUID y contexto territorial/correlacion opcional, compatible con el evento existente.
- `technical-doc/IMPLEMENTACION_16_AUDITORIA_LOGGING_TRANSVERSAL.md`: inventario de riesgos y estrategia de redaccion.

## Validacion

- MS-Audit: 7 pruebas aprobadas, build aprobado, `prisma validate` y `prisma generate` aprobados.
- Gateway: 2 pruebas del controlador ADMIN y build aprobados.
- MS-Auth: prueba del publisher y build aprobados.
- Base Railway: migracion aplicada, esquema sin drift; insert/read dentro de transaccion y rollback real aprobados.
- No se conecto NATS productivo ni se hicieron pruebas de evento end-to-end.

## Configuracion y activacion

En el servicio Railway `ms-audit` deben configurarse `DATABASE_URL` con la base proporcionada y `NATS_SERVERS` con el NATS privado compartido por MS-Auth y Gateway. No se guardo `DATABASE_URL` en `.env`, Railway config ni Git. El `railway.toml` ejecuta `npm ci`, genera Prisma, compila y, en start, ejecuta `prisma migrate deploy` antes de arrancar el microservicio. No incluye healthcheck HTTP porque MS-Audit es un consumidor NATS.

El repositorio remoto `https://github.com/smart-services-sz/ms-audit.git` se comprobo accesible y vacio. Falta inicializar/publicar el codigo y confirmar que el servicio Railway este vinculado al repo. No hay herramienta de Railway disponible en este entorno para comprobar el estado final del deploy o cargar variables.

## Limite de entrega y proximo trabajo

Solo los eventos que publican `audit.log` se persisten; hoy MS-Auth es el productor integrado. Los logs operativos de Nest en Gateway, MS-AI, Geo, Reclamos, Routing y Notifications siguen yendo a stdout, no a esta tabla. La API esta disponible para ADMIN, pero no se agrego UI.

MS-Auth publica con NATS Core y fire-and-forget. Si MS-Audit esta caido o desconectado, NATS Core no conserva/reproduce el evento y no se puede afirmar entrega durable. Para garantizarla, evaluar JetStream con consumer durable/ack o transactional outbox; publicar un UUID no resuelve la perdida cuando el consumidor esta offline. Tambien falta desplegar MS-Audit, configurar NATS, verificar consumo real y migrar por fases los productores restantes siguiendo la politica de redaccion del informe 16.