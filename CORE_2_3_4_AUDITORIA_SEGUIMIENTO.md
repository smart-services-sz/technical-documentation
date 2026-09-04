# SmartService - Auditoria y seguimiento CORE 2, CORE 3 y CORE 4

Fecha de inicio: 2026-09-03

Este documento registra la auditoria inicial, decisiones, validaciones ejecutadas, faltantes y avance incremental para cerrar CORE 2 e implementar CORE 3 y CORE 4. Debe actualizarse al finalizar cada incremento verificable.

## Regla operativa sobre bases de datos

- No ejecutar operaciones destructivas sobre bases de datos.
- No eliminar ni reiniciar migraciones existentes.
- Si una tarea requiere conectarse a una base de datos, detenerse y pedir al usuario el string de conexion de la DB correspondiente antes de continuar.
- Para auditorias sin conexion, usar lectura de esquemas Prisma, migraciones existentes, codigo y documentacion.

## Fuente funcional y tecnica

Fuente principal indicada por el usuario:

`C:\Users\zavat\OneDrive\Desktop\TESIS - SMART SERVICE\Documentos Revisados por IA\Primera revision\PRIMER-BORRADOR-03-09-2026-COMPLETA-TECNOLOGICA.pdf`

Estado de acceso al PDF:

- El archivo existe en el entorno.
- No hay extractor de PDF disponible detectado en la auditoria inicial (`pypdf`, `PyPDF2`, `pdfplumber` ni `pdftotext` usable).
- Hasta contar con extraccion de texto, esta bitacora se basa en el alcance pegado por el usuario y la documentacion versionada del workspace.

Documentacion complementaria revisada:

- `docs/diagramas/caso-de-uso.md`
- `docs/diagramas/dominio-asignar-ruta.md`
- `docs/diagramas/clases-asignar-rutas.md`
- `docs/diagramas/robustez-asignar-rutas.md`
- `docs/diagramas/secuencia-asignar-rutas.md`
- `docs/diagramas/prototipo-rustico-core-02-rutas.md`
- `technical-doc/REPORTE_ALINEAMIENTO_TESIS_VS_CODIGO.md`

## Alcance de trabajo

### CORE 2 - Generacion y asignacion de rutas optimizadas

Objetivo: auditar y finalizar seleccion de reclamos, simulacion, generacion, asignacion, confirmacion, consulta de rutas, intervencion operativa inicial, evidencia, trazabilidad, Gateway, dashboard, migraciones y pruebas.

Superficies principales detectadas:

- `ms-routing/src/routing`
- `ms-routing/src/routing/patterns/strategy`
- `ms-routing/src/routing/patterns/state`
- `ms-routing/prisma/schema.prisma`
- `gateway/src/routing`
- `admin-dashboard/src/components/routing-routes-panel.tsx`
- `admin-dashboard/src/components/routing-generated-panel.tsx`
- `admin-dashboard/src/components/routing-plans-panel.tsx`
- `admin-dashboard/src/app/api/routing`

### CORE 3 - Gestion integral de la intervencion y cierre

Objetivo: implementar CU-008-001 a CU-008-011 con ruta asignada, inicio de recorrido, detalle de parada, intervencion, evidencia, nueva visita, finalizacion, revision, cierre e historial integral.

Superficies principales detectadas:

- `ms-routing/src/routing/dto/register-intervention.dto.ts`
- `ms-routing/src/routing/dto/attach-intervention-evidence.dto.ts`
- `ms-routing/prisma/schema.prisma`
- `ms-reclamos/src/reclamos`
- `ms-reclamos/prisma/schema.prisma`
- `gateway/src/routing`

### CORE 4 - Seguimiento y comunicacion ciudadana

Objetivo: implementar CU-009-001 a CU-009-011 con seguimiento publico, WhatsApp, validacion de identidad, notificaciones, reintentos, respuesta ciudadana, reapertura e historial de comunicaciones.

Superficies principales detectadas:

- `gateway/src/webhooks`
- `gateway/src/inbound`
- `gateway/src/whatsapp`
- `public-site`
- `ms-reclamos/src/reclamos`
- Posible nuevo microservicio `ms-notifications`

## Validaciones iniciales ejecutadas

Comandos no destructivos ejecutados durante la auditoria inicial:

| Proyecto | Build | Tests | Lint no destructivo | Resultado |
| --- | --- | --- | --- | --- |
| `ms-routing` | `npm run build` | `npm test` | `npx eslint "{src,apps,libs,test}/**/*.ts"` | Build OK, tests OK con 1 suite minima, lint falla con 110 problemas |
| `gateway` | `npm run build` | `npm test` | `npx eslint "{src,apps,libs,test}/**/*.ts"` | Build OK, tests fallan por providers NATS no mockeados en specs de roles/permisos, lint falla con muchos problemas de tipos/formato |
| `ms-reclamos` | `npm run build` | `npm test` | `npx eslint "{src,apps,libs,test}/**/*.ts"` | Build OK, tests OK con 1 suite minima, lint falla con 30 problemas |
| `admin-dashboard` | `npm run build` | `npm test` | `npm run lint` | Build OK, tests OK con 3 tests, lint falla por `Date.now()` durante render en `auth-context.tsx` |

Observaciones:

- La raiz `C:\smartservice-backend` no es repositorio Git.
- `technical-doc` contiene `.git`, por lo que puede estar versionado de forma independiente.
- Los scripts `lint` de servicios Nest usan `--fix`; para auditoria se uso ESLint directo sin autocorreccion.

## Matriz de cobertura inicial

| Requisito / CU | Estado actual | Evidencia | Faltantes principales | Aceptacion prevista |
| --- | --- | --- | --- | --- |
| CORE 2 / CU-005-001 seleccionar reclamos geolocalizados | Parcial funcional | `RoutingService.fetchClaims` llama `reclamos.find-all`; filtra geo valida despues | Filtros por municipio, area, estado y prioridad; snapshot de autorizacion; seleccion explicita | Reclamos candidatos respetan geo, municipio, area, zona, categoria, estado y prioridad |
| CORE 2 / CU-005-002 generar ruta optimizada | Parcial funcional | Strategy local/Google/fallback; persistencia de plan/rutas/paradas | Usar `pesoPrioridad`; pruebas de estrategia; robustecer fallback y errores externos | Simulacion y generacion asignan rutas ordenadas con cupos y fallback verificable |
| CORE 2 / reglas de cupo y priorizacion | Parcial | `RoutingCategoryRule`, `RoutingCrewRule`, orden por prioridad fija | Aplicar peso configurable; pruebas de cupos por categoria y recurso | Cupos y prioridad explican asignados/no asignados |
| CORE 2 / confirmacion de plan | Parcial funcional | `confirmPlan` cambia plan, rutas y allocations | Publicar evento de dominio; validar conflictos sin sobrescribir estados; autorizacion | Confirmar plan es idempotente, rechaza conflictos y registra trazabilidad |
| CORE 2 / consulta de ruta asignada | Backend parcial | `routing.assigned-route.get`, endpoint Gateway `POST /routing/assigned-route` | Proxy/dashboard operativo; pertenencia de ruta; municipio/area | Agente solo consulta rutas propias/autorizadas |
| CORE 2 / estados de ruta | Parcial | State pattern en `route-lifecycle.*` | Transiciones demasiado laxas; faltan pruebas e historial | Transiciones invalidas son rechazadas y registradas |
| CORE 2 / evidencia | Parcial | `RoutingInterventionEvidence` guarda URL | Storage real, limites de tipo/tamano, idempotencia, autorizacion | Evidencia valida queda asociada y trazada |
| CORE 3 / CU-008-001 consultar ruta asignada | Parcial backend | Reusa CORE 2 | Vista dedicada y reglas de pertenencia | Ruta asignada visible solo para actor autorizado |
| CORE 3 / CU-008-002 iniciar recorrido | Parcial | `updateRouteStatus` permite `in_progress` | Actor, idempotencia, restricciones de estado | Ruta inicia una vez y registra actor/fecha |
| CORE 3 / CU-008-003 consultar detalle de parada | Faltante como caso especifico | Modelo `RoutingStop` existe | Endpoint/DTO/vista; autorizacion por ruta/parada | Detalle muestra datos necesarios sin filtrar internos indebidos |
| CORE 3 / CU-008-004 registrar intervencion | Parcial peligroso | `registerIntervention` hace upsert por `stopId` | Idempotencia real; impedir sobrescritura indebida; valores de dominio finales; validaciones de evidencia/observacion | Intervencion duplicada no altera resultado; casos invalidos fallan |
| CORE 3 / CU-008-005 estados coordinados | Parcial inconsistente | `ms-routing` llama `reclamos.update` por RPC | Contratos versionados; compensacion/error parcial; historial | Estado de parada/ruta/reclamo queda consistente o compensado |
| CORE 3 / CU-008-006 adjuntar evidencia | Parcial | Endpoint NATS y Gateway existen | Dashboard/proxy, storage, limites, idempotencia | Evidencia adjunta una sola vez y se audita |
| CORE 3 / CU-008-007 solicitar nueva visita | Faltante/parcial por estado | `requiere_nueva_visita` -> parada `reprogramado` | Entidad `NuevaVisita`; reglas de replanificacion; historial | Reclamo queda activo y replanificable |
| CORE 3 / CU-008-008 finalizar recorrido | Parcial automatico | Si no quedan pendientes, marca ruta completed | Finalizacion explicita, actor/motivo, reglas para reprogramados/omitidos | Ruta solo finaliza bajo reglas documentadas |
| CORE 3 / CU-008-009 revisar resolucion | Faltante | No hay `RevisionResolucion` | Modelo, servicio, endpoint, dashboard | Resolucion revisada antes de cierre cuando aplique |
| CORE 3 / CU-008-010 cerrar reclamo | Parcial CRUD generico | `reclamos.update` permite `cerrado` | Regla de cierre administrativo; permisos; historial | No cierra sin revision obligatoria |
| CORE 3 / CU-008-011 historial integral | Faltante | Solo `MensajeHistorial` de conversacion inicial | `HistorialAccion` integral y ordenado | Historial incluye ruta, parada, intervencion, evidencia, revision y cierre |
| CORE 4 / CU-009-001 consultar estado por codigo | Faltante real | `public-site` tiene prototipo simulado | Endpoint publico seguro, rate limit, datos publicos | Consulta valida no expone datos sensibles ni permite enumeracion |
| CORE 4 / CU-009-002 consultar estado por WhatsApp | Parcial de infraestructura | Gateway inbound y WhatsApp existen | Flujo especifico de seguimiento, validacion identidad | WhatsApp responde estado publico del reclamo autorizado |
| CORE 4 / CU-009-003 validar identidad | Faltante | Campos `dni`, `correo`, `contactKey` existen | Politica anti enumeracion, limites de intentos | Identidad invalida no revela existencia |
| CORE 4 / CU-009-004 a CU-009-011 notificaciones, respuesta, reapertura, historial | Faltante | Documentacion tecnica indica notificaciones pendientes | `ms-notifications`, eventos, Strategy de canal, reintentos, respuesta ciudadana, reapertura | Cambios comunicables generan notificacion idempotente y reintentos trazados |

## Inconsistencias detectadas

| Tema | Documento / requisito | Codigo actual |
| --- | --- | --- |
| Entidades CORE 2 | `JornadaOperativa`, `RutaTrabajo`, `PuntoRuta`, `HistorialAccion` | Se usan `RoutingPlan`, `RoutingRoute`, `RoutingStop`, sin `JornadaOperativa` ni `HistorialAccion` explicitos |
| Filtros CORE 2 | Municipio, area, zona, categoria, estado, prioridad | Routing solo recibe reclamos genericos y filtra categoria/zona/cupo localmente |
| Prioridad | Pesos configurables | `pesoPrioridad` existe pero la prioridad efectiva es fija |
| Autorizacion | Rol, permiso, municipio, area y pertenencia | Gateway valida rol/permiso; falta aislamiento municipal/area/pertenencia en dominio |
| CORE 3 resultados | `RESOLVED`, `NEW_VISIT`, `NOT_RESOLVED` o valores finales exactos | Codigo usa `resuelto`, `no_resuelto`, `requiere_nueva_visita`, `no_corresponde` |
| Resolucion vs cierre | Separacion operativa/administrativa | Intervencion puede llevar reclamo a `resuelto` directamente |
| Contratos NATS | Version, correlationId, idempotencyKey, errores parciales | Predominan RPC simples sin envelope versionado |
| Evidencia | Evidencia con seguridad y almacenamiento | Solo URL persistida; no storage ni limites robustos |
| CORE 4 | Seguimiento y notificaciones ciudadanas | No existe `ms-notifications`; sitio publico contiene simulaciones |

## Inventario NATS inicial

Patrones detectados:

- Auth/users: `auth.login`, `auth.register`, `auth.refresh`, `auth.logout`, `users.get.all`, `users.get.id`, `users.get.by.phone`, `users.toggle.status`, `users.revoke.sessions`, `users.create`, `users.update`, `users.delete`.
- Roles/permisos: `roles.*`, `user.roles.*`, `role.permissions.*`, `permissions.*`, `permission.*`, `user.permissions.*`.
- AI: `ai.process-inbound-message`.
- Geo: `geo.geocode-address`.
- Reclamos: `reclamos.create`, `reclamos.find-all`, `reclamos.find-one`, `reclamos.update`, `reclamos.remove`.
- Routing: `routing.simulate`, `routing.generate`, `routing.rules.upsert`, `routing.rules.get`, `routing.plan.get`, `routing.plans.list`, `routing.plan.confirm`, `routing.plan.delete`, `routing.area-plans.list`, `routing.area-plans.get`, `routing.area-plans.save`, `routing.area-plans.delete`, `routing.assigned-route.get`, `routing.route.status.update`, `routing.intervention.register`, `routing.intervention.evidence.attach`.
- Audit: `audit.log` emitido desde `ms-auth`, sin servicio consumidor verificado en esta auditoria.

Brecha principal: no hay envelope versionado comun ni eventos de dominio para cambios de estado/notificaciones.

## Migraciones previstas

### `ms-routing`

- Historial de acciones de ruta/parada/intervencion.
- Campos o relaciones para municipio, area y pertenencia operativa si no se resuelven desde auth/claims.
- Idempotencia en comandos de ruta, intervencion y evidencia.
- Restricciones unicas para secuencia por ruta y reclamo por ruta/plan.
- Posible entidad para nueva visita o solicitud de replanificacion.

### `ms-reclamos`

- Historial de estados/acciones.
- Revision de resolucion.
- Cierre administrativo con actor, motivo y referencia a revision.
- Reapertura y respuesta ciudadana.
- Indices para seguimiento publico por codigo e identidad sin enumeracion.

### Posible `ms-notifications`

- `Notification`.
- `NotificationAttempt`.
- `CitizenResponse`.
- Configuracion de canales.
- Politica de reintentos e idempotencia.

## Contratos previstos

Envelope recomendado:

```json
{
  "version": "1",
  "correlationId": "uuid",
  "idempotencyKey": "string",
  "occurredAt": "iso-date",
  "producer": "service-name",
  "actor": {
    "id": "string",
    "roles": []
  },
  "municipalityId": "string",
  "payload": {}
}
```

Contratos candidatos:

- `routing.v1.route.confirmed`
- `routing.v1.route.status.changed`
- `routing.v1.intervention.recorded`
- `routing.v1.evidence.attached`
- `reclamos.v1.status.change.requested`
- `reclamos.v1.status.changed`
- `reclamos.v1.resolution.reviewed`
- `reclamos.v1.claim.closed`
- `reclamos.v1.claim.reopened`
- `notifications.v1.notification.requested`
- `notifications.v1.delivery.attempted`
- `citizen.v1.resolution.responded`

## Plan de implementacion incremental

### Incremento 1 - Cierre tecnico base de CORE 2

- Corregir lint no destructivo de `ms-routing` en la superficie de routing.
- Agregar pruebas unitarias de Strategy local/Google/fallback.
- Agregar pruebas de State de rutas.
- Agregar pruebas de idempotencia de generacion y confirmacion.
- Validar build/test/lint de `ms-routing`.

### Incremento 2 - Seleccion y generacion CORE 2 completa

- Completar filtros de seleccion.
- Aplicar `pesoPrioridad`.
- Endurecer reservas y conflictos.
- Registrar historial.
- Validar Gateway y dashboard existentes.

### Incremento 3 - Consulta/asignacion operativa CORE 2

- Exponer proxies faltantes en `admin-dashboard/src/app/api/routing`.
- Extender `admin-dashboard/src/services/routing.service.ts`.
- Crear/ajustar vistas para ruta asignada y detalle operativo.
- Validar permisos y pertenencia.

### Incremento 4 - CORE 3 vertical

- Implementar detalle de parada.
- Implementar intervencion idempotente sin sobrescritura indebida.
- Implementar evidencia segura.
- Implementar nueva visita.
- Implementar revision y cierre administrativo.
- Implementar historial integral.

### Incremento 5 - CORE 4 vertical

- Definir si se crea `ms-notifications`.
- Implementar seguimiento publico web.
- Implementar seguimiento por WhatsApp.
- Implementar eventos comunicables.
- Implementar estrategia de canales y reintentos.
- Implementar respuesta ciudadana, disconformidad y reapertura.

### Incremento 6 - Verificacion integral

- Ejecutar builds de servicios y frontends afectados.
- Ejecutar tests unitarios, integracion viable y frontend.
- Ejecutar lint no destructivo.
- Verificar migraciones sin reset.
- Verificar contratos NATS.
- Actualizar README y documentacion tecnica.

## Riesgos y decisiones abiertas

| Decision | Estado | Notas |
| --- | --- | --- |
| Extraccion del PDF | Pendiente | Necesaria para alinear paginas especificas y evitar depender solo del resumen pegado |
| Valores finales de intervencion | Pendiente | Confirmar si usar ingles (`RESOLVED`, `NEW_VISIT`, `NOT_RESOLVED`) o mantener valores actuales con mapeo |
| `JornadaOperativa` explicita | Pendiente | Decidir si `RoutingPlan` satisface el concepto o si se requiere entidad separada |
| `ms-notifications` | Recomendado, pendiente de confirmacion | CORE 4 lo justifica segun arquitectura indicada |
| Storage de evidencia | Pendiente | Elegir Wasabi/S3, `ms-files` o estrategia dev/prod |
| Aislamiento municipal/area | Pendiente | Verificar claims/JWT/modelo de auth para saber de donde obtener municipio/area |
| Tests base de Gateway | Pendiente | Falla por `NATS_CLIENT` no mockeado; puede ocultar regresiones si no se corrige temprano |

## Registro de avance

| Fecha | Incremento | Cambios | Validacion | Estado |
| --- | --- | --- | --- | --- |
| 2026-09-03 | Auditoria inicial | Inventario CORE 2/3/4, validaciones base, brechas y plan | Build/test/lint no destructivo en proyectos afectados | Completado con limitacion de PDF sin extraccion |
| 2026-09-03 | Incremento 1 - base tecnica CORE 2 | Formato y limpieza de lint en `ms-routing`; tipado de reglas internas y respuesta NATS de reclamos; pruebas unitarias para Strategy local, fallback y State de rutas | `npm run build`; `npm test`; `npx eslint "{src,apps,libs,test}/**/*.ts"`; diagnostics VS Code sin errores en archivos tocados | Completado parcialmente: base limpia y patrones cubiertos; faltan pruebas de idempotencia/generacion/confirmacion |
| 2026-09-03 | Incremento 1 - filtros CORE 2 | Agregados filtros opcionales `claimStatuses`, `priorities` y `categories` en DTOs de `ms-routing` y Gateway; `ms-routing` propaga filtros compatibles a `reclamos.find-all` y aplica filtrado local defensivo; dashboard refleja filtros de estado/prioridad y envia categorias del plan | `ms-routing`: build/test/lint OK con 14 tests; `gateway`: build OK y lint focal del DTO OK; `admin-dashboard`: test focal y build OK, lint focal de archivos tocados OK | Completado parcialmente: filtros por estado/prioridad/categoria listos; municipio/area queda pendiente por modelo de auth/datos |
| 2026-09-03 | Incremento 1 - prioridad y reservas CORE 2 | `pesoPrioridad` ahora pondera el orden de candidatos y paradas; la paginacion de reclamos ya no se corta si una pagina queda vacia por filtro local; la persistencia de plan chequea reclamos asignados/reservados antes de hacer `upsert` de reservas | `ms-routing`: `npm run build`, `npm test`, `npx eslint "{src,apps,libs,test}/**/*.ts"` OK con 16 tests; `admin-dashboard`: test/build/lint focal OK para contrato de `pesoPrioridad` | Completado parcialmente: prioridad ponderada y conflictos de reserva cubiertos; falta historial/trazabilidad y filtros municipio/area |
| 2026-09-03 | Incremento 1 - historial CORE 2 | Agregado modelo `RoutingActionHistory`, migracion incremental `20260903112000_add_routing_action_history`, escritura append-only para generacion, confirmacion, cambio de estado, intervencion y evidencia; `getPlan` devuelve `actionHistory`; dashboard muestra historial operativo en detalle de plan | `ms-routing`: `npm run db:generate`, `npm run build`, `npm test`, `npx eslint "{src,apps,libs,test}/**/*.ts"` OK con 18 tests; `gateway`: build y lint focal routing OK; `admin-dashboard`: build y lint focal OK | Completado parcialmente: historial local de routing listo; migracion no ejecutada por falta de string de conexion DB y por regla de no conectar sin confirmacion |
| 2026-09-04 | Migracion de historial CORE 2 | Aplicada en la base PostgreSQL de `ms-routing` la migracion `20260903112000_add_routing_action_history` | `npx prisma migrate deploy` OK; `npx prisma migrate status` confirma `Database schema is up to date!` | Completado para persistencia del historial de routing |
| 2026-09-04 | Alcance territorial estructurado | Agregados `municipalityId` y `areaId` opcionales a usuarios/JWT y reclamos; migraciones incrementales preparadas; dashboard de usuarios permite asignarlos | `ms-auth`: `npx prisma generate` y build OK; migracion `20260904130000_add_territorial_scope_to_users` aplicada y status OK; `ms-reclamos`: `npx prisma generate`, build y tests OK; dashboard build OK | Parcial: falta aplicar migracion de `ms-reclamos`, propagar valores desde alta de reclamo/geocodificacion y conectar filtros de routing |
| 2026-09-04 | Alcance territorial conectado | Migracion de `ms-reclamos` aplicada; geocodificacion normaliza `locality` a `municipalityId` y `adminArea` a `areaId`; filtros territoriales propagados por consultas de reclamos, routing y dashboard según el usuario operativo | `ms-reclamos`: migracion deploy/status OK, build/tests OK; `ms-routing`: build OK, 10 tests focales territoriales OK; Gateway build OK; `ms-ai`: build y 39 tests focales OK; dashboard build, test focal y lint focal OK | Parcial: falta autorización territorial efectiva en Gateway/routing y catálogo canónico de municipios/áreas |
| 2026-09-04 | Autorizacion territorial inicial | Gateway impone `municipalityId`/`areaId` del JWT a AGENT/OFFICIAL en simulacion/generacion; ADMIN conserva alcance explicito; consulta de ruta asignada se fuerza al usuario autenticado; dashboard administra los identificadores | Gateway build OK; spec `routing.controller.spec.ts`: 3 tests OK; lint focal OK | Parcial: falta proteger cambio de estado/intervencion/evidencia y validar catálogo territorial canónico |
| 2026-09-04 | Autorizacion de operaciones CORE 2/3 | Gateway fija `actorId`/`actorIsAdmin` desde JWT para cambio de estado, intervención y evidencia; `ms-routing` verifica pertenencia de la ruta antes de ejecutar esos comandos | `ms-routing`: build y 18 tests OK; Gateway: build, 3 tests focales y lint focal OK | Completado para pertenencia operativa; falta catálogo territorial canónico y pruebas de integración con persistencia real |
| 2026-09-04 | Convencion territorial canonica | Se adopta `lower_snake_case` para `municipalityId` y `areaId`, validada en DTOs de auth, reclamos, Gateway y routing; se mantiene el mapeo slug de geocodificacion sin crear un servicio de catálogo fuera del alcance documentado | `ms-auth`, `ms-reclamos`, `ms-routing` y Gateway compilan OK | Completado para formato/validación; queda pendiente una fuente administrativa formal si el proyecto incorpora catálogo territorial |
| 2026-09-04 | Decisiones de dominio CORE 2/3 | Se mantienen los resultados documentados `resuelto`, `no_resuelto`, `requiere_nueva_visita` y `no_corresponde`; `RoutingPlan` se adopta como realización técnica de `JornadaOperativa` porque agrupa fecha, estado, rutas y no asignados | Contraste con `docs/diagramas/dominio-asignar-ruta.md` y código vigente de `ms-routing` | Completado; no se crea una entidad duplicada ni se cambia el contrato de intervención |
| 2026-09-04 | Historial integral CORE 3 - base | Agregado `HistorialAccion` append-only en `ms-reclamos`; creación y actualización de reclamos registran acción, origen, actor, estado anterior/nuevo, motivo y referencia; routing enlaza intervención/evidencia con `referenciaId` | Migracion `20260904150000_add_claim_action_history` deploy/status OK; `ms-reclamos`: generate/build/tests OK; `ms-routing`: build y 18 tests OK | Base de historial integrada; quedan revisión de resolución, cierre administrativo, nueva visita y consulta integral para los casos CORE 3 |
| 2026-09-04 | Consulta integral de historial CORE 3 | Agregado patrón NATS `reclamos.history`, consulta ordenada append-only en `ms-reclamos`, endpoint Gateway `GET /reclamos/:id/history`, proxy Next y contrato de dashboard | `ms-reclamos`: build/tests OK; Gateway build OK; dashboard build OK con ruta `/api/reclamos/[id]/history` | Completado para consulta; quedan revisión de resolución, nueva visita y cierre administrativo |
| 2026-09-04 | Revisión, nueva visita y cierre CORE 3 | Agregados modelos `RevisionResolucion` y `NuevaVisita`; revisión aprobada/rechazada, solicitud de nueva visita y cierre administrativo condicionado a revisión aprobada; todos registran `HistorialAccion`; Gateway expone comandos protegidos con actor JWT | Migración `20260904160000_add_resolution_review_and_new_visit` deploy/status OK; `ms-reclamos`: generate/build/tests OK; Gateway build/lint focal OK | Completado para estos comandos; queda UI específica y pruebas de integración persistentes de CORE 3 |
| 2026-09-04 | UI CORE 3 | `ReclamosPanel` incorpora revisión de resolución, solicitud de nueva visita y cierre administrativo con formularios inline, motivo obligatorio y actualización de listado; servicios cliente y contratos Gateway conectados | `ms-reclamos`: build/tests OK; Gateway build OK; dashboard build/tests OK (3 tests) | Completado para flujo administrativo UI; quedan pruebas de integración persistentes específicas |
| 2026-09-04 | Integración persistente CORE 3 | Agregada suite opt-in `reclamos.integration.spec.ts` que crea/limpia un reclamo aislado y verifica cierre bloqueado sin revisión, revisión rechazada/aprobada, nueva visita, cierre e historial integral | Con `RUN_INTEGRATION_TESTS=true` y `DATABASE_URL` de `ms-reclamos`: 1 test OK; ejecución normal sin variable: 1 test real + 1 suite opt-in omitida | Completado para el flujo persistente base de revisión/nueva visita/cierre |
| 2026-09-04 | CORE 4 - ms-notifications base | Creado `ms-notifications` con Prisma propio, consumidor NATS `reclamos.v1.status.changed`, idempotencia, persistencia de intentos, RetryPolicy, Strategy `log`/WhatsApp, pruebas unitarias y documentación; agregado PostgreSQL 5435 al Compose raíz | `prisma validate`, build, 3 tests y lint de `ms-notifications` OK; no se aplicaron migraciones por no existir todavía conexión DB del servicio | Completado el esqueleto funcional del servicio; queda productor de eventos de `ms-reclamos` y flujo ciudadano completo de CORE 4 |
| 2026-09-04 | CORE 4 - productor de estados | `ms-reclamos` publica `reclamos.v1.status.changed` después de persistir cambios de estado; envelope incluye version, correlationId, idempotencyKey, actor, municipio, trackingCode, estado y destinatario; errores NATS no revierten el cambio local | `ms-reclamos`: build/tests/lint focal OK; `ms-notifications`: build/tests/lint OK | Completado el productor/consumidor base; quedan seguimiento WhatsApp conversacional, respuesta ciudadana, reapertura y prueba end-to-end NATS |
| 2026-09-04 | CORE 4 - seguimiento WhatsApp | `ms-reclamos.public-tracking` valida código y `contactKey` y devuelve solo estado/categoría/fecha pública; `ms-ai` detecta consultas `REC-*` en WhatsApp antes del flujo de creación y responde sin datos internos | `ms-reclamos`: build/tests OK; `ms-ai`: build y 39 tests focales OK; Redis mostró solo errores de infraestructura esperables en tests | Completado el seguimiento WhatsApp base; quedan respuesta ciudadana, reapertura y prueba end-to-end NATS |
| 2026-09-04 | CORE 4 - respuesta y reapertura | Agregados `RespuestaCiudadana` y `Reapertura`; confirmación/disconformidad por código + contactKey; disconformidad reabre el reclamo a `en_proceso`; historial registra `citizen.response` y `claim.reopened`; detección WhatsApp integrada en `ms-ai` | Migración `20260904170000_add_citizen_response_reopening` aplicada; `ms-reclamos`: build/tests OK; `ms-ai`: build y 39 tests focales OK | Completado para respuesta/reapertura; queda prueba end-to-end NATS |
| 2026-09-04 | CORE 4 - prueba end-to-end NATS | Agregada suite opt-in en `ms-notifications` que publica `reclamos.v1.status.changed` por NATS, verifica notificación `sent` e intento persistido y limpia solo el registro creado | Ejecución normal: 5 tests OK y 1 suite E2E omitida; build/lint OK | Completado como prueba automatizada opt-in; requiere NATS y `DATABASE_URL` de notifications para ejecución real |

## Pendientes inmediatos

- Obtener texto del PDF fuente o habilitar extractor para las paginas indicadas.
- Mantener la convención `lower_snake_case` hasta que exista una fuente administrativa formal para el catálogo territorial.
- Ejecutar la prueba E2E NATS en el entorno desplegado con `RUN_E2E_NATS=true`.

### Hallazgo de auditoria territorial - 2026-09-04

- `ms-auth` y `ms-reclamos` ya persisten los identificadores opcionales; el catálogo territorial todavía no existe como entidad propia.
- La implementación actual usa identificadores slug derivados de `locality` y `adminArea` para mantener continuidad con los datos de geocodificación.
- Antes de producción debe reemplazarse o validar ese mapeo contra un catálogo canónico y hacer cumplir autorización por municipio/área en Gateway y `ms-routing`.