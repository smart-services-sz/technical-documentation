# Auditoría de alineamiento: tesis vs. sistema implementado

**Fecha de auditoría:** 13 de octubre de 2026
**Proyecto:** SmartService
**Resultado general:** Alineación parcial; el núcleo backend y de ruteo está avanzado, pero hay brechas funcionales, de aislamiento multi-municipio y de preparación para producción que impiden considerar completo el MVP definido por la tesis.

## 1. Alcance y criterio

La referencia principal fue el PDF de tesis fechado el 03/09/2026 indicado en la bitácora de auditoría de CORE. Se contrastaron sus objetivos funcionales y técnicos con el código, esquemas Prisma, endpoints, dashboard, sitio público, Compose y pruebas disponibles. La revisión del PDF se realizó en modo de solo lectura.

Este documento clasifica el grado de diferencia, no asigna un porcentaje global. No existe una matriz ponderada de requisitos que permita justificar una cifra de alineamiento. Tampoco se conectaron bases de datos ni se ejecutaron pruebas de integración que requieran infraestructura externa.

**Escala:**
- **Crítica:** riesgo de seguridad o bloqueo para una prueba piloto multi-municipio.
- **Alta:** requisito funcional del MVP faltante o flujo principal actualmente incompleto.
- **Media:** capacidad parcialmente implementada, persistencia/resiliencia faltante o brecha de operación.
- **Baja / documental:** diferencia de evolución o documentación que no bloquea por sí sola el MVP.

## 2. Hallazgos priorizados

### Crítica

1. **Aislamiento de reclamos por municipio y área no impuesto en el listado.** El endpoint permite `ADMIN` y `AGENT`, pero no toma el usuario autenticado para limitar automáticamente la consulta a su municipio/área. Los filtros territoriales son opcionales y llegan desde la query; si se omiten, MS-Reclamos no aplica condición territorial. Un agente podría consultar reclamos de otros municipios. Es un riesgo estático confirmado en el camino del código; no se probó contra datos reales. Referencias: [Gateway controller](../gateway/src/reclamos/reclamos.controller.ts), [filtros de reclamos](../gateway/src/reclamos/dto/find-reclamos.dto.ts), [consulta en MS-Reclamos](../ms-reclamos/src/reclamos/reclamos.service.ts).

### Alta

2. **El seguimiento público no coincide entre frontend, Gateway y MS-Reclamos.** El sitio manda solo el código de seguimiento; Gateway lo reenvía sin `contactKey`, mientras el DTO del consumidor lo declara obligatorio y MS-Reclamos aplica `ValidationPipe` global. Por ello, el flujo actual probablemente rechaza la consulta antes de devolver el estado. La función alternativa que consulta solo por código no es la que invoca el controlador NATS. Referencias: [sitio público](../public-site/app.js), [Gateway service](../gateway/src/reclamos/reclamos.service.ts), [DTO de seguimiento](../ms-reclamos/src/reclamos/dto/public-tracking.dto.ts), [bootstrap MS-Reclamos](../ms-reclamos/src/main.ts).

3. **No hay formulario ciudadano web conectado para crear reclamos.** La tesis incluye un formulario web complementario dentro del MVP. El sitio actual implementa consulta de seguimiento, pero no el envío de una solicitud; aunque la API de creación existe, no está conectada a una interfaz ciudadana web. Referencias: [página pública](../public-site/index.html), [JavaScript público](../public-site/app.js), [endpoint Gateway de reclamos](../gateway/src/reclamos/reclamos.controller.ts).

4. **No hay carga real ni almacenamiento externo de evidencias.** Las evidencias de intervención guardan una URL y metadatos, pero no encontré endpoints de carga multipart ni integración S3/Wasabi. La tesis describe el almacenamiento multimedia fuera de la base de datos. Referencias: [modelo de ruteo](../ms-routing/prisma/schema.prisma), [DTO de evidencia](../ms-routing/src/routing/dto/attach-intervention-evidence.dto.ts).

5. **La detección de duplicados declarada como diferencial de IA no está implementada.** MS-AI interpreta y clasifica mensajes, pero no encontré lógica de similitud, búsqueda de reclamos existentes ni una decisión de duplicidad en su código fuente. Referencia: [intérprete de IA](../ms-ai/src/ai-process/ai-interpreter.service.ts).

6. **La configuración municipal no está modelada como módulo.** Usuarios y reclamos almacenan identificadores territoriales, pero no existe en estos esquemas un catálogo administrable de municipios, áreas, barrios, categorías y parámetros por cliente. Las reglas y planes de ruteo tampoco tienen una asociación municipal completa. Referencias: [esquema de Auth](../ms-auth/prisma/schema.prisma), [esquema de Routing](../ms-routing/prisma/schema.prisma).

7. **Notificaciones de ciclo de vida incompletas.** MS-Notifications existe y consume el evento versionado de cambio de estado, pero MS-Reclamos lo publica desde el método genérico `update`; cambios directos como cierre, revisión, nueva visita y reapertura no recorren ese publicador. Además, el canal predeterminado es `log`, por lo que no hay entrega ciudadana por WhatsApp salvo configuración adicional. Referencias: [servicio de reclamos](../ms-reclamos/src/reclamos/reclamos.service.ts), [servicio de notificaciones](../ms-notifications/src/notifications/notifications.service.ts), [consumer NATS](../ms-notifications/src/notifications/notifications.controller.ts).

8. **Reportes e indicadores del dashboard no están conectados.** La sección de métricas sigue siendo un placeholder y declara que los gráficos y reportes se agregarán al conectar los microservicios. La tesis los contempla como capacidades del MVP. Referencia: [dashboard administrativo](../admin-dashboard/src/components/admin-dashboard.tsx).

9. **Datos personales registrados en logs de creación.** MS-Reclamos escribe el payload completo a nivel `debug`; puede contener contacto, correo, DNI, ubicación y mensajes. Si `debug` se activa en producción o los logs se centralizan, hay riesgo de exponer datos personales. Referencia: [controlador de reclamos](../ms-reclamos/src/reclamos/reclamos.controller.ts).

### Media

10. **Conversaciones sin persistencia histórica dedicada.** El estado conversacional vive en Redis con expiración; no encontré MongoDB ni otra persistencia histórica de conversaciones y resultados variables de IA. La tesis menciona MongoDB como tecnología que podría utilizarse, por lo que esta diferencia es moderada y no un bloqueo categórico del MVP.

11. **No se encontró BullMQ para trabajos en segundo plano.** Hay NATS y reintentos de notificación dentro del servicio, pero no una cola BullMQ dedicada para tareas diferidas, procesamiento de archivos o reintentos durables.

12. **Despliegue y continuidad operativa incompletos.** La tesis plantea Railway para el despliegue inicial y contempla políticas de backup. Hay Compose para las bases y otro Compose de Gateway para Redis/NATS; no encontré Dockerfiles para los servicios, manifiestos de despliegue Railway, workflows de CI/CD ni automatización de copias. Referencias: [Compose de bases](../docker-compose.yml), [Compose de infraestructura Gateway](../gateway/docker-compose.yml).

13. **Auditoría presente, pero parcial y distribuida.** Reclamos y rutas conservan historiales de acciones, por lo que no corresponde clasificarlos como carentes de auditoría. No se identificó un servicio central que consolide accesos, configuración y operaciones de todos los módulos; la cobertura no equivale aún a la trazabilidad integral propuesta.

14. **Integración climática ausente.** No encontré conexión con APIs de clima ni uso de datos meteorológicos para ajustar prioridad. La tesis la plantea como integración externa/diferencial de evolución, no como el bloqueo más inmediato del MVP.

### Baja / documental

15. **Documentación de alineamiento desactualizada.** El reporte previo de mayo afirma que Routing y Notifications faltan y que PostgreSQL diverge de MySQL. En el código actual existen `ms-routing` y `ms-notifications`; el PDF de tesis revisado especifica PostgreSQL y menciona MongoDB como posible complemento. La matriz de CORE 2-4 también refleja faltantes que ya tienen implementación parcial o endpoints actuales. Referencias: [reporte de alineamiento previo](REPORTE_ALINEAMIENTO_TESIS_VS_CODIGO.md), [bitácora CORE 2-4](CORE_2_3_4_AUDITORIA_SEGUIMIENTO.md), [resumen de arquitectura](TECHNICAL_ARCHITECTURE_SUMMARY.md).

## 3. Coincidencias comprobadas

- Arquitectura de microservicios NestJS/TypeScript con Gateway central y comunicación NATS.
- Autenticación JWT, roles y permisos; gestión de usuarios implementada.
- PostgreSQL para Auth, Reclamos, Routing y Notifications; es consistente con la versión de tesis revisada.
- MS-AI integra OpenAI y usa Redis para estado conversacional; MS-Geo integra geocodificación de Google.
- Reclamos dispone de creación, consulta, actualización, estados e historial; hay además revisión de resolución, nueva visita, cierre y reapertura en backend.
- `ms-routing` implementa simulación/generación de planes, reglas, persistencia, asignación, rutas operativas, intervenciones y evidencia referenciada por URL; incluye estrategia local/Google Maps y fallback.
- `ms-notifications` incluye idempotencia, persistencia de intentos, reintentos y estrategia WhatsApp, sujeta a credenciales, template aprobado y configuración.
- Dashboard administrativo y sitio público existen; el dashboard incorpora vistas de reclamos, planes y rutas.

## 4. Resultado de pruebas

Pruebas ejecutadas sin conexión a bases de datos:

| Proyecto | Resultado |
|---|---|
| `ms-ai` | 10 suites y 50 pruebas aprobadas |
| `ms-routing` | 5 suites y 31 pruebas aprobadas |
| `ms-reclamos` | 1 suite aprobada; 1 prueba de integración omitida |
| `ms-notifications` | 2 suites y 5 pruebas aprobadas; integración NATS omitida |
| `admin-dashboard` | 4 archivos y 24 pruebas aprobadas |
| `gateway` | Suite con fallos de inyección de dependencias de prueba (`NATS_CLIENT`) |
| `ms-auth` | Suite con fallos de inyección de dependencias de prueba (`PrismaService`/`NATS_CLIENT`) |
| `ms-geo` | No define script `test` |

No se habilitaron pruebas de integración con NATS, Redis o PostgreSQL. La revisión de los endpoints y flujos descritos es estática; el seguimiento público y el aislamiento territorial deben confirmarse con pruebas de integración una vez que exista un entorno de prueba seguro.

## 5. Prioridad sugerida para cerrar brechas

1. Imponer aislamiento territorial desde el contexto autenticado y probar acceso cruzado entre municipios.
2. Corregir y probar el contrato del seguimiento ciudadano, incluyendo verificación de identidad conforme al requisito.
3. Completar el canal de alta web y el circuito de evidencias con almacenamiento externo y controles de acceso.
4. Publicar eventos para todos los cambios comunicables y validar entrega real de WhatsApp en un entorno de prueba.
5. Implementar detección de duplicados y reportes mínimos del MVP.
6. Definir modelo de configuración municipal y completar despliegue, backups y pruebas de integración.
7. Actualizar las matrices de documentación técnica una vez que cada brecha se cierre.

**Estado de cambios:** este documento registra la auditoría; no se modificaron servicios ni datos del sistema durante su elaboración.
