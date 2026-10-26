# Probar el TP desde GitHub Actions

El workflow `Terraform manual PR5` usa el commit revisado
`5532dcfc3a954476f130b7ff340f4b4cf1e47048` del PR 5. Puede ejecutarse desde
una rama interna del repositorio, sin fusionar el codigo de la aplicacion.

En **Actions -> Terraform manual PR5 -> Run workflow**, elegir la rama que
contiene estos cambios y una operacion:

- `plan`: muestra los cambios pendientes del despliegue.
- `apply`: guarda un plan, lo aplica y comprueba la API con un usuario demo.
- `destroy`: prepara y aplica un plan de destruccion del TP. Ejecutarlo solo
  cuando se quiera eliminar el despliegue.

Se usan los secrets `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY` y
`AWS_SESSION_TOKEN`. La sesion del Learner Lab debe estar activa. El despliegue
se hace en la cuenta asociada a esas credenciales y en **us-east-1**.

## Estado persistente

El workflow prepara el bucket privado y versionado
`itba-cloud-2026q2-g1-tfstate-<cuenta>-us-east-1`. Conserva el estado en
`delivery/terraform.tfstate` y usa bloqueo S3 nativo. Las ejecuciones se
serializan y no cancelan un apply en curso.

El bucket del estado se conserva despues de `destroy`. No borrar sus objetos:
el estado permite actualizar y eliminar los recursos del TP. Nunca publicar
el estado ni los planes como artefactos: contienen credenciales generadas.

Si existen recursos `delivery` sin estado remoto, el workflow se detiene para
evitar duplicarlos. En ese caso hay que migrar el estado local que los creo.

## Probar el frontend

Despues de un apply correcto, el resumen de la ejecucion muestra la URL del
API y el User Pool. El artefacto `frontend-config` contiene unicamente la
configuracion publica `.env.local`; copiarla a `frontend/.env.local` del
codigo del PR y ejecutar `npm ci` y `npm run dev` desde `frontend/`.

Los usuarios demo son `cliente.demo@example.com` y
`repartidor.demo@example.com`. Sus contrasenas no se imprimen en los logs.
Un administrador del Lab puede establecerlas desde Cognito, en el User Pool
mostrado en el resumen de la ejecucion.

El smoke test comprueba autenticacion, catalogo y conexion Lambda -> RDS.
Las condiciones de carrera y el manejo de pedidos rechazados encontrados en
la revision del PR siguen siendo limitaciones del codigo de la aplicacion.
