# Implementacion 15: guardar municipio en usuarios existentes sin datos laborales

**Fecha:** 15 de octubre de 2026

**Estado:** corregido y validado localmente. Sin cambios de datos remotos, migraciones, commit ni push.

## Diagnostico

El JWT presentado contenia `municipalityId: null`. En una consulta autorizada de solo lectura a MS-Auth se confirmo que el usuario administrador indicado seguia con municipio nulo en la base, mientras el municipio `funes` existia y estaba activo. El identificador del usuario coincidía con el `sub` del token. No se consultaron hashes de contrasena ni se modifico la cuenta.

El flujo de login ya toma el municipio persistido y lo incluye al firmar el token. El formulario y los proxies tambien conservan el campo. El bloqueo se reprodujo en la validacion HTML: Cargo, Legajo y Area tenian `required` incluso al editar usuarios existentes que tienen esos datos nulos. Aunque se escribiera `funes`, el formulario era invalido y el navegador impedia su envio.

## Cambios

- `admin-dashboard/src/components/access-management-panel.tsx`: Cargo, Legajo y Area siguen siendo obligatorios al crear usuarios, pero no al editar los existentes. Nombre y email permanecen obligatorios; no se exige ni cambia la contrasena al editar si se deja vacia.
- `admin-dashboard/src/components/access-management-panel.test.tsx`: reproduce la cuenta sin datos laborales, comprueba `form.checkValidity()`, verifica el envio de `municipalityId: "funes"` y la ausencia de una contrasena nueva. Tambien comprueba que el alta mantenga sus requisitos.

No se cambio la firma del JWT, no se elimino la validacion municipal de ruteo y no se asignaron municipios automaticamente.

## Validacion

- Antes del cambio, la prueba de edicion fallo porque `form.checkValidity()` devolvia `false`.
- Despues del cambio, las dos pruebas nuevas pasaron y se verifico el payload de actualizacion con un doble del servicio.
- Dashboard completo: 7 archivos y 30 pruebas aprobados; typecheck y build de produccion aprobados.
- Diagnosticos de los dos archivos modificados: sin errores.
- No se ejecuto una actualizacion de usuario real ni un login remoto; la prueba no cambia el estado de la cuenta en produccion.

## Activacion

1. Publicar la correccion del dashboard.
2. Editar el administrador en Usuarios, ingresar `funes` en Municipio y guardar. No es necesario inventar datos laborales para esta cuenta existente.
3. Recargar Usuarios y volver a abrir la edicion para comprobar que el municipio se haya persistido.
4. Cerrar sesion e ingresar nuevamente: un token emitido despues del guardado debe contener `municipalityId: "funes"`.
5. Volver a Planes. La carga de reglas de ruteo, si todavia faltan, es una configuracion distinta del municipio del usuario.