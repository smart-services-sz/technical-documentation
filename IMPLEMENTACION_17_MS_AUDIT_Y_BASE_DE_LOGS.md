# Implementacion 17: MS-Audit y almacenamiento central de eventos

**Fecha:** 16 de octubre de 2026  
**Estado:** servicio y consulta implementados; migracion aplicada; smoke test real de escritura/lectura verificado con rollback. El usuario confirma el deploy de Railway. La verificacion posterior aun no observa eventos almacenados, por lo que falta confirmar la ingesta desde NATS.

**Correcciones de build Railway (16/10/2026):** el primer build detecto Node 18.20.5, incompatible con Prisma 7.10.1; se fijo Node 22.22.2 mediante `.node-version` y `engines.node`, y Prisma CLI quedo como dependencia de runtime para migrate deploy. Un segundo intento reporto `EBUSY` porque el `npm ci` duplicado dentro de `buildCommand` intentaba borrar `/app/node_modules/.cache`, montado como cache por Railway. Nixpacks ya ejecuta la instalacion en su etapa previa; por eso el `buildCommand` queda solo con `prisma generate` y `nest build`. Se valido localmente esa secuencia con Node 22.22.2.

**Correccion de arranque Railway (16/10/2026):** el log `Function._resolveFilename` correspondia a `node dist/src/main.js`, pero Nest compila el entrypoint en `dist/main.js`. Se corrigieron `start`, `start:prod` y `railway.toml`; el build local confirma que `dist/main.js` existe y la configuracion ya no refiere la ruta antigua.

## Entrega

Se creo `ms-audit` como microservicio NestJS/Prisma 7 para recibir `audit.log` por NATS y almacenar eventos en PostgreSQL. Se adapto la referencia de `C:\actas-be\ms-audit` al patron `@prisma/adapter-pg` usado por SmartService; no se copio su configuracion local ni se leyo su `.env`.

El modelo conserva los campos de auditoria de MS-Auth y agrega `eventId`, `municipalityId` y `correlationId`. `eventId` es opcional y unico para compatibilidad; MS-Auth ahora emite un UUID por evento y el consumidor usa upsert para deduplicar reenvios con el mismo ID. La metadata se redacciona recursivamente por nombres de claves que aparentan contener secretos y se limita a 32 KB.

Gateway expone `GET /api/audit/logs`, con rol `ADMIN`, filtros y paginacion de hasta 100 registros. La entrega 18 agrega la vista React de Auditoria en el dashboard.

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
- `technical-doc/IMPLEMENTACION_18_AUDITORIA_TRANSVERSAL_Y_DASHBOARD.md`: instrumentacion HTTP/NATS y consulta en dashboard.

## Validacion

- MS-Audit: 7 pruebas aprobadas, build aprobado, `prisma validate` y `prisma generate` aprobados.
- Gateway: 2 pruebas del controlador ADMIN y build aprobados.
- MS-Auth: prueba del publisher y build aprobados.
- Base Railway: migracion aplicada, esquema sin drift; insert/read dentro de transaccion y rollback real aprobados.
- Tras el deploy reportado, una consulta de solo lectura conto eventos agrupados por servicio: total cero. No se leyo metadata ni PII. Esto no distingue si aun no hubo una accion auditada o si falta conectar/verificar NATS.
- No se ejecuto una prueba end-to-end de evento a traves de NATS.

## Configuracion y activacion

En el servicio Railway `ms-audit` deben configurarse `DATABASE_URL` con la base proporcionada y `NATS_SERVERS` con el NATS privado compartido por los productores. No se guardo `DATABASE_URL` en `.env`, Railway config ni Git. Nixpacks instala dependencias; `buildCommand` ejecuta `prisma generate` y compila, y `startCommand` ejecuta `prisma migrate deploy` antes de arrancar. No incluye healthcheck HTTP porque MS-Audit es un consumidor NATS.

El repositorio remoto `https://github.com/smart-services-sz/ms-audit.git` tiene el codigo publicado en `main`; el usuario informa que el deploy se realizo. No hay Railway CLI/proyecto vinculado en este entorno para inspeccionar las variables o los logs del proceso.

## Limite de entrega y proximo trabajo

Solo los eventos que publican `audit.log` se persisten. MS-Auth tiene eventos de negocio y la entrega 18 añade eventos operativos HTTP/NATS en Gateway y los seis microservicios. La consulta ADMIN y la vista dashboard estan en codigo, pero falta confirmar el despliegue de esas versiones y su ingesta real.

Los productores operativos publican con NATS Core y fire-and-forget. El conteo de la base era cero en la ultima consulta. Siguiente smoke test: confirmar que `NATS_SERVERS` en Gateway y todos los productores coincide con MS-Audit; ejecutar una accion auditada no sensible por servicio y comprobar que aumenta el conteo. Si no aumenta, inspeccionar logs de conexion/consumer sin volcar payloads. NATS Core no conserva/reproduce eventos si MS-Audit esta desconectado; evaluar JetStream durable/ack u outbox para garantizar entrega.