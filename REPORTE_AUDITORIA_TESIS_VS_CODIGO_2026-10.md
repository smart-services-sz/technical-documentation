# Auditoría de alineamiento: tesis vs. sistema implementado

**Fecha de auditoría:** 16 de octubre de 2026
**Proyecto:** SmartService
**Resultado general:** El núcleo funcional, el alta pública y el aislamiento territorial están implementados en código. Aún falta validar el despliegue y los flujos externos de extremo a extremo; métricas e higiene de datos personales siguen abiertas.

## 1. Alcance y criterio

La referencia principal es el PDF de tesis fechado el 03/09/2026 indicado en la bitácora de auditoría de CORE. Esta actualización contrasta el estado anterior con las implementaciones 01–15, código, esquemas, endpoints y pruebas registradas. Se distingue la existencia de código de la validación con servicios externos o datos reales; no se afirma que toda la solución esté desplegada en producción.

Este documento clasifica el grado de diferencia, no asigna un porcentaje global. No existe una matriz ponderada de requisitos que permita justificar una cifra de alineamiento. Tampoco se conectaron bases de datos ni se ejecutaron pruebas de integración que requieran infraestructura externa.

**Escala:**
- **Crítica:** riesgo de seguridad o bloqueo para una prueba piloto multi-municipio.
- **Alta:** requisito funcional del MVP faltante o flujo principal actualmente incompleto.
- **Media:** capacidad parcialmente implementada, persistencia/resiliencia faltante o brecha de operación.
- **Baja / documental:** diferencia de evolución o documentación que no bloquea por sí sola el MVP.

## 2. Hallazgos priorizados

### Crítica

1. **Aislamiento territorial de reclamos: implementado; falta prueba integrada de autorización.** Gateway deriva municipio/área del JWT y MS-Reclamos aplica el ámbito recibido. Ruteo exige municipio autenticado también para ADMIN y filtra reglas, planes, rutas, intervenciones y evidencias. Las pruebas cubren filtros falsificados y acceso cruzado con dobles; falta recorrerlo integrado con dos municipios desplegados. Los consumidores internos NATS siguen siendo confiables y deben propagar el ámbito cuando actúan por un usuario. Referencias: [implementación 01](IMPLEMENTACION_01_AISLAMIENTO_TERRITORIAL.md), [implementación 13](IMPLEMENTACION_13_AISLAMIENTO_MUNICIPAL_ROUTING.md).

### Alta

2. **Seguimiento público con código y contacto: implementado; sin prueba integrada real.** El ciudadano aporta código más teléfono o correo; MS-Reclamos devuelve respuesta mínima solo si coinciden y Gateway limita por IP con Redis, fallando cerrado si Redis no está disponible. No es OTP ni verifica posesión actual. Falta confirmar Gateway/NATS/DB desplegados y la IP real detrás del proxy. Referencia: [implementación 02](IMPLEMENTACION_02_SEGUIMIENTO_PUBLICO.md).

3. **Alta ciudadana web y validación municipal: implementadas; dependencias externas por verificar.** El sitio tiene formulario, seguimiento y categorías activas del catálogo. Gateway consulta y valida municipio, categoría y cobertura antes de crear. La UI se probó con APIs simuladas; Google Geocoding, NATS productivo y creación desplegada no se validaron de extremo a extremo. Referencias: [implementación 03](IMPLEMENTACION_03_ALTA_CIUDADANA_WEB.md), [implementación 12](IMPLEMENTACION_12_ALTA_CON_CATALOGO_MUNICIPAL.md).

4. **Evidencias S3/Wasabi: flujo privado implementado; bucket real pendiente de verificar.** Alta ciudadana e intervenciones tienen multipart, validación de contenido, límites y URLs firmadas con autorización. Las pruebas usan S3 simulado; faltan credenciales/política, carga/descarga real y limpieza verificada en el entorno. Referencias: [implementación 04](IMPLEMENTACION_04_EVIDENCIAS_S3.md), [implementación 05](IMPLEMENTACION_05_EVIDENCIAS_INTERVENCION.md).

5. **Sugerencias de reclamos posiblemente duplicados: primera versión implementada.** MS-AI compara reclamos activos por municipio/categoría, proximidad y similitud textual; ofrece hasta tres candidatos a revisión humana, sin bloquear ni fusionar. Falta calibración con datos representativos y evaluación de falsos positivos/negativos. Referencia: [implementación 10](IMPLEMENTACION_10_DUPLICADOS_ASISTIDOS.md).

6. **Configuración municipal y aislamiento de ruteo: implementados; operación multi-municipio integrada pendiente.** Catálogo, alta pública, MS-AI y alcance JWT de ruteo están conectados en código. Las migraciones se aplicaron en las bases verificadas; la base de ruteo estaba vacía. Se corrigió la edición municipal del usuario ADMIN y el usuario confirmó que pudo entrar a Planes. Las pruebas de dos municipios usan dobles; falta validar reglas, cuadrillas, zonas, altas y rutas integradas en un entorno desplegado. El checksum histórico de `0_baseline` difiere y permanece documentado; otros entornos con datos deben revisarse antes de migrar. Referencias: [implementación 11](IMPLEMENTACION_11_CONFIGURACION_MUNICIPAL.md), [implementación 12](IMPLEMENTACION_12_ALTA_CON_CATALOGO_MUNICIPAL.md), [implementación 13](IMPLEMENTACION_13_AISLAMIENTO_MUNICIPAL_ROUTING.md), [implementación 14](IMPLEMENTACION_14_ALINEACION_Y_MIGRACION_DB_ROUTING.md), [implementación 15](IMPLEMENTACION_15_EDICION_MUNICIPIO_USUARIO.md).

7. **Notificaciones de ciclo de vida y WhatsApp: eventos implementados; entrega externa pendiente de prueba.** MS-Reclamos publica eventos para transiciones directas y respuestas ciudadanas. MS-Notifications prioriza WhatsApp y evita duplicados de respuesta. Meta debe aprobar/configurar el template v2 y falta E2E real con NATS/Meta para entrega, errores y reintentos. Referencias: [implementación 06](IMPLEMENTACION_06_WHATSAPP_CONVERSACIONAL.md), [implementación 07](IMPLEMENTACION_07_WHATSAPP_PRIORITARIO_Y_CONVERSACIONAL.md), [implementación 08](IMPLEMENTACION_08_MENU_RAPIDO_Y_HISTORIAL_RECLAMO.md), [implementación 09](IMPLEMENTACION_09_TEMPLATE_META_VARIABLES.md).

8. **Reportes e indicadores del dashboard: pendientes.** La sección de métricas sigue siendo placeholder. Definir primero indicadores mínimos, denominadores, filtros temporales/territoriales y autorización por municipio. Referencia: [dashboard administrativo](../admin-dashboard/src/components/admin-dashboard.tsx).

9. **Secretos y PII en logs: saneamiento implementado; verificar despliegue y regresiones.** La entrega 18 retiró connection strings, payloads, texto/contacto, direcciones/coordenadas, destinatarios y cuerpos de proveedor; añadió pruebas negativas e interceptores de metadata allowlist. Falta confirmar que todos los despliegues usan estas versiones y mantener pruebas de regresión. Referencias: [inventario](IMPLEMENTACION_16_AUDITORIA_LOGGING_TRANSVERSAL.md), [implementación 18](IMPLEMENTACION_18_AUDITORIA_TRANSVERSAL_Y_DASHBOARD.md).

### Media

10. **Persistencia histórica de conversaciones: no implementada.** El estado vive en Redis con expiración. Definir propósito, retención y privacidad antes de agregar almacenamiento histórico; no es el bloqueo inmediato del MVP.

11. **Cola BullMQ: no encontrada.** Hay NATS y reintentos de notificación. Evaluar solo si los requisitos de trabajos durables lo justifican.

12. **Continuidad operativa: CI/CD, backups y despliegue reproducible por cerrar/verificar.** Hay Compose, pero no se evidenció automatización completa de backups y CI/CD. Confirmar Railway por servicio, restauración probada, alertas y runbooks; no inferir el estado de recursos gestionados sin revisar el entorno.

13. **Auditoría central: instrumentación y dashboard implementados; ingesta desplegada pendiente de confirmar.** La entrega 18 añade eventos operativos HTTP/NATS de Gateway y los seis microservicios, persistencia MS-Audit y vista ADMIN con filtros. Pruebas/build aprobados localmente. Falta desplegar Dashboard, Gateway y productores instrumentados, y confirmar que NATS compartido lleve una operación sintética de cada servicio a PostgreSQL. NATS Core `emit` no garantiza replay; JetStream/outbox queda pendiente. Referencias: [implementación 16](IMPLEMENTACION_16_AUDITORIA_LOGGING_TRANSVERSAL.md), [implementación 17](IMPLEMENTACION_17_MS_AUDIT_Y_BASE_DE_LOGS.md), [implementación 18](IMPLEMENTACION_18_AUDITORIA_TRANSVERSAL_Y_DASHBOARD.md).

14. **Integración climática: ausente.** Diferencial de evolución, no bloqueo inmediato del MVP.

### Baja / documental

15. **Documentación histórica: depurada parcialmente.** Esta matriz refleja las implementaciones 01–15; revisar matrices CORE e informes previos que aún describan estados anteriores.

## 3. Coincidencias comprobadas

- Arquitectura de microservicios NestJS/TypeScript con Gateway central y comunicación NATS.
- Autenticación JWT, roles y permisos; gestión de usuarios implementada.
- PostgreSQL para Auth, Reclamos, Routing y Notifications; es consistente con la versión de tesis revisada.
- MS-AI integra OpenAI y usa Redis para estado conversacional; MS-Geo integra geocodificación de Google.
- Reclamos dispone de creación, consulta, actualización, estados e historial; hay además revisión de resolución, nueva visita, cierre y reapertura en backend.
- `ms-routing` implementa planificación municipal, reglas, asignación, operación y evidencias S3 privadas; las pruebas cruzadas usan dobles y las migraciones verificadas se compararon con Prisma.
- `ms-notifications` incluye idempotencia, persistencia de intentos, reintentos y estrategia WhatsApp; la entrega real requiere credenciales, template Meta aprobado y conectividad.
- Dashboard administrativo, panel municipal y sitio ciudadano existen; el formulario público carga categorías activas del catálogo.

## 4. Resultado de pruebas

Validaciones registradas en las entregas; estos resultados no equivalen a pruebas de producción:

| Proyecto | Resultado |
|---|---|
| `ms-ai` | 11 suites y 77 pruebas aprobadas; build aprobado |
| `ms-routing` | 5 suites y 41 pruebas aprobadas; build aprobado |
| `ms-reclamos` | 4 suites y 18 pruebas aprobadas; 1 prueba de integración omitida; build aprobado |
| `ms-notifications` | Pruebas enfocadas WhatsApp: 2 suites y 7 pruebas; Meta/NATS real no ejecutado |
| `admin-dashboard` | 7 archivos y 30 pruebas aprobadas; typecheck y build aprobados |
| `gateway` | 6 suites enfocadas y 50 pruebas aprobadas; build aprobado. La suite general mantiene fallos preexistentes de inyección en fixtures genéricos |
| `ms-auth` | Pruebas municipales enfocadas y build aprobados; la suite general mantiene fallos preexistentes de inyección (`PrismaService`/`NATS_CLIENT`) |
| `ms-geo` | No define script `test` |

Alta pública y UI se probaron con dobles/API interceptada; no se enviaron reclamos reales ni mensajes WhatsApp durante esas pruebas. Las migraciones de MS-Auth/MS-Routing sí se aplicaron en las bases autorizadas verificadas. Falta integrar Gateway, NATS, Redis, Google, S3 y Meta en staging. Repetir conteos al ejecutar una nueva tanda completa.

## 5. Próximos pasos priorizados

1. **Verificar la ingesta desplegada:** desplegar Dashboard, Gateway y productores instrumentados; confirmar NATS común y un evento sintético/no sensible por servicio.
2. **Asegurar durabilidad y retención:** decidir JetStream/outbox, ACK/reintentos, retención y alertas; revisar acceso a IP/User-Agent y metadata.
3. **Prueba integrada de dos municipios:** alta, seguimiento, planes y acceso cruzado con datos sintéticos.
4. **WhatsApp real en sandbox:** template v2, entrega/reintentos y respuesta rápida sin duplicados.
5. **Métricas mínimas y operación:** indicadores territoriales, backup restaurable, CI/CD, alertas y runbooks. Mantener documentada la discrepancia del checksum de `0_baseline`; no usar reset ni reescribir historial.

**Estado de cambios:** se actualizó la matriz de hallazgos, validaciones y prioridades con el estado de las implementaciones 01–18. No se modificaron servicios ni datos durante esta actualización documental.
