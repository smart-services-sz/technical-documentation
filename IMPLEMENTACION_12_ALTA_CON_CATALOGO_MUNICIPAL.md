# Implementacion 12: alta ciudadana conectada al catalogo municipal

**Fecha:** 15 de octubre de 2026

**Estado:** implementado y validado localmente con dobles de servicios. Sin migracion nueva, cambios de datos ni modificaciones de variables en Railway.

## Resumen

El alta web y el alta conversacional de MS-AI consultan `municipalities.config.get` en MS-Auth antes de geocodificar o crear el reclamo. El municipio lo vincula el servidor; no se deduce de la direccion ni se acepta desde metadata ciudadana. Nombre, ciudad, pais, cobertura, categorias habilitadas y prioridad inicial se obtienen del catalogo.

La creacion se bloquea si falta el vinculo municipal, el municipio no existe o esta inactivo, no responde el catalogo o la categoria no esta habilitada. La direccion debe tener geocodificacion exitosa, coordenadas validas, localidad igual a la ciudad configurada y pais permitido; cuando hay cobertura administrativa configurada, debe coincidir exactamente con la provincia o localidad, normalizando mayusculas y acentos. Se rechazan componentes geograficos faltantes. La provincia ya no se usa como `areaId` operativo en MS-AI.

Gateway publica `GET /api/reclamos/public-config` con una proyeccion explicita del municipio y de las categorias activas compatibles. No expone reglas, cobertura interna ni parametros privados. El formulario carga esas opciones, bloquea el envio mientras carga o si falla el catalogo y ofrece reintento. Se verifican nuevamente las reglas al enviar; una lista previamente cargada no permite eludir una desactivacion posterior.

El endpoint historico `POST /api/reclamos` deja de ser anonimo: exige JWT, rol `ADMIN` o `AGENT` y permiso `MANAGE_RECLAMOS`. Aplica el alcance territorial autenticado para impedir que funcione como bypass del alta ciudadana validada. Los clientes ciudadanos deben utilizar `POST /api/reclamos/public`.

## Archivos modificados

- `gateway/src/reclamos/reclamos.service.ts`: consulta del catalogo, proyeccion publica, validacion territorial/categorias y prioridad municipal antes de almacenar evidencias o crear el reclamo.
- `gateway/src/reclamos/reclamos.controller.ts`: endpoint de configuracion publica y restriccion del alta administrativa.
- `gateway/src/reclamos/dto/public-claim-submission.dto.ts`: exporta los codigos de categoria admitidos para reutilizarlos en la proyeccion publica.
- `gateway/src/reclamos/public-claim-submission.spec.ts`: casos de alta valida, cobertura, categorias inactivas, municipio incorrecto y catalogo no disponible.
- `gateway/src/reclamos/reclamos.service.spec.ts`: adapta el caso de limpieza de evidencias al nuevo contrato.
- `gateway/src/reclamos/reclamos.controller.spec.ts`: comprueba metadata de acceso, configuracion publica y alcance del alta administrativa.
- `ms-ai/src/conversation/conversation-state-support.service.ts`: vinculo municipal configurado en servidor, cobertura del catalogo y prioridad; elimina inferencia de municipio/area desde geocodificacion.
- `ms-ai/src/conversation/conversation-state-support.service.spec.ts`: prueba fallos de catalogo, cobertura cruzada, ID independiente del nombre de localidad y metadata ciudadana ignorada.
- `ms-ai/src/ai-process/ai-process.service.ts`: advertencia de arranque si falta el vinculo municipal; el alta queda bloqueada, no abierta.
- `ms-ai/src/conversation/states/collecting-claim-data.state.ts`: distingue categoria no habilitada y muestra la cobertura del catalogo; no promete reintentos internos inexistentes.
- `ms-ai/src/conversation/states/collecting-claim-data.state.spec.ts`: respuestas ante categoria no habilitada, cobertura invalida y fallo tecnico.
- `public-site/app.js`: carga del catalogo, estados de error/reintento y bloqueo de envios simultaneos; URL local valida al abrir el HTML como archivo.
- `public-site/index.html`: opciones dinamicas, municipio visible y controles inicialmente deshabilitados.
- `public-site/styles.css`: estados ocultos/deshabilitados y margen del ancla para no tapar el formulario con la cabecera fija.
- `public-site/README.md`: requisitos y configuracion del nuevo flujo.
- `technical-doc/REPORTE_AUDITORIA_TESIS_VS_CODIGO_2026-10.md`: estado actualizado del hallazgo municipal, sin dar por terminada la integracion de ruteo.

## Requisitos antes del despliegue

1. Registrar o activar el municipio en el panel ADMIN, con pais, ciudad predeterminada y cobertura correctos. Si el nombre es institucional, indicar `defaultCity` con la localidad real.
2. Habilitar las categorias que atiende el municipio y sus prioridades. Se mantienen los diez codigos admitidos por el dominio: `agua_y_cloacas`, `alumbrado`, `baches_y_pavimento`, `arbolado`, `residuos`, `electricidad`, `gas`, `transporte`, `infraestructura` y `otros`. Los codigos arbitrarios no se publican ni se aceptan en el alta web.
3. Configurar `PUBLIC_MUNICIPALITY_ID` en Gateway y `AI_MUNICIPALITY_ID` en MS-AI con el ID correspondiente al canal. Para un mismo municipio ambos deben coincidir. No son valores suministrados por el ciudadano. Sin `AI_MUNICIPALITY_ID` se bloquean las altas conversacionales.
4. Desplegar Gateway, MS-AI y sitio publico con MS-Auth disponible en NATS. Los servicios consultan el catalogo al crear cada reclamo; no hay fallback a cobertura abierta ni a variables geograficas antiguas.
5. Verificar en un entorno de prueba una alta permitida y una direccion de otro municipio rechazada antes de habilitar trafico real. Adaptar clientes del endpoint historico al nuevo requisito de autenticacion.

## Validacion ejecutada

- MS-AI: 11 suites y 77 pruebas aprobadas; build aprobado.
- Gateway: 3 suites enfocadas y 31 pruebas aprobadas; build aprobado. No se volvieron a ejecutar las suites genericas con fallos de inyeccion ya documentados en la entrega 11.
- Sitio publico: `node --check` aprobado. Prueba Playwright con API interceptada en escritorio (1440 x 1000) y movil (390 x 844): opciones municipales, envio multipart sin territorio ciudadano, catalogo caido/vacio, recuperacion por reintento y ausencia de desbordes en controles. Navegacion al ancla sin superposicion con la cabecera.
- Diagnosticos TypeScript de los archivos de implementacion revisados: sin errores. `git diff --check`: sin errores de whitespace.
- Las pruebas no crearon reclamos reales ni llamaron a NATS, WhatsApp o Google. No se conecto ninguna base remota durante esta entrega.

## Limites y siguiente paso

Este corte vincula un municipio por instancia/canal configurado; no resuelve el multiplexado de varios municipios por distintos numeros de WhatsApp en una misma instancia. La cobertura se valida por componentes de direccion, no por poligonos. El interprete mantiene las categorias del dominio y no incorpora taxonomias nuevas de forma dinamica.

No se asignan automaticamente barrios ni areas operativas. Reglas municipales JSON, particion de reglas/planes de ruteo y prueba integrada de extremo a extremo con dos municipios siguen pendientes. La proxima entrega debe abordar esa asociacion territorial de ruteo sin modificar datos existentes implicitamente.