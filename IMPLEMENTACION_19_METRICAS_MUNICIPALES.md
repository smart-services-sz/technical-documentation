# Implementacion 19: metricas operativas municipales

**Fecha:** 16 de octubre de 2026
**Estado:** agregados, endpoints y panel implementados; pruebas y builds locales aprobados. La verificacion desplegada con dos municipios queda pendiente.

## Objetivo

Completar el panel administrativo con indicadores de reclamos y operacion de ruteo, evitando mezclar datos entre municipios y evitando descargar registros individuales para calcular metricas. Los endpoints requieren un municipio; Gateway fuerza el municipio del JWT para usuarios municipales. Un ADMIN de plataforma debe seleccionar explicitamente un municipio.

## Metricas de reclamos

`GET /api/reclamos/metrics` entrega agregados calculados por MS-Reclamos para municipio, rango de fechas y filtros seleccionados. Incluye total del periodo y anterior, variacion, pendientes/en proceso, resueltos, cerrados, rechazados, tasa de resolucion, promedio y mediana de horas de resolucion; desgloses por estado, categoria, prioridad, canal y area; y una serie temporal diaria, semanal o mensual.

Los filtros de estado, categoria, prioridad, canal y area se aplican en backend. El rango se limita a 366 dias. Las consultas agregadas no seleccionan contacto, documento, texto del reclamo ni direccion.

## Metricas de ruteo

`GET /api/routing/metrics` entrega, para el municipio y rango seleccionados, planes por estado, rutas por estado, reclamos asignados y sin asignar, distancia y duracion planificadas, paradas por estado y categoria, intervenciones por resultado, evidencias y serie temporal de planes. Las entidades hijas se filtran por su plan propietario; los registros legados sin municipio no entran en el resultado.

El modelo Routing no dispone de una dimension de area en el plan. Por ello, los indicadores se presentan como totales municipales: comparten municipio y fechas con la vista, pero no heredan filtros de area, estado, categoria, prioridad ni canal de reclamos. La interfaz lo aclara para evitar interpretar esas cifras como segmentadas.

## Dashboard y seguridad

El panel ADMIN muestra las metricas de reclamos y ruteo. Para usuarios con municipio, el selector queda fijado al municipio autenticado; ADMIN de plataforma puede consultar un municipio por vez. Los agregados se solicitan en paralelo y un error de un servicio no oculta el resultado del otro. La vista incluye tarjetas KPI, series temporales y desgloses de estados/categorias/causas, ademas de filtros temporales y dimensiones propias de reclamos.

El Dashboard utiliza Route Handlers autenticados para `/api/reclamos/metrics` y `/api/routing/metrics`. Los servicios retornan conteos y medidas agregadas, no filas de reclamos, direcciones, contactos ni DNI.

## Validacion local

- MS-Routing: 5 suites, 42 pruebas aprobadas; build aprobado. Incluye comprobacion de municipio propietario en agregaciones de planes, rutas, paradas, intervenciones, no asignados y evidencias.
- Gateway: pruebas focalizadas del controlador de metricas de Routing aprobadas; verifican precedencia del municipio del JWT y seleccion requerida para ADMIN de plataforma; build aprobado. La suite general conserva fallos de fixtures no relacionados que no registran `NATS_CLIENT`.
- MS-Reclamos: 5 suites, 21 pruebas aprobadas y 1 integracion omitida; build aprobado.
- Admin Dashboard: 11 archivos, 40 pruebas aprobadas; build de produccion aprobado. Las pruebas cubren aislamiento municipal, consulta con mismo municipio/rango y render de indicadores de ruteo.
- No se conectaron bases de datos, NATS ni proveedores externos, y no se modificaron datos.

## Pendiente

Desplegar Dashboard, Gateway, MS-Reclamos y MS-Routing en un entorno compartido y verificar los endpoints con dos municipios y datos sinteticos. Confirmar filtros, scope del JWT, denominadores y ausencia de informacion personal en respuestas. No inferir aislamiento desplegado a partir de las pruebas unitarias.
