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
Para leerlas desde una maquina propia con Terraform 1.16.4, AWS CLI y las
credenciales vigentes del mismo Lab, entrar al codigo del PR #5 y ejecutar:

```bash
cd terraform/organization
if [ ! -e override.tf ]; then
  cat > override.tf <<'HCL'
terraform {
  backend "s3" {}
}
HCL
else
  echo 'Se conserva override.tf existente; verificar que configure backend S3.'
fi
cuenta=$(aws sts get-caller-identity --query Account --output text)
terraform init -input=false \
  -backend-config="bucket=itba-cloud-2026q2-g1-tfstate-${cuenta}-us-east-1" \
  -backend-config='key=delivery/terraform.tfstate' \
  -backend-config='region=us-east-1' \
  -backend-config='encrypt=true' \
  -backend-config='use_lockfile=true'
terraform output -json usuarios_demo
```

`override.tf` es una configuracion local ignorada por Git. Si ya existe,
conservarla y revisar su contenido antes de cambiarla. Usar siempre el mismo
backend del workflow para administrar este despliegue; el flujo local del
README original usa otro estado si no se agrega este backend.

Las contrasenas se muestran solo en la terminal propia. No copiarlas en el
README, issues o logs publicos. Cambiarlas directamente en Cognito dejaria
desactualizado el valor que usa la prueba automatica del pipeline.

## Evidencia del despliegue

La ejecucion [40323713204](https://github.com/itba-cloud/2026Q2-G1/actions/runs/40323713204)
creo 162 recursos y paso el smoke test de autenticacion y catalogo. Cuenta
`712638924387`, region `us-east-1`. El User Pool es `us-east-1_v2d8BmjQx` y la
API es https://nmvfaa3yhe.execute-api.us-east-1.amazonaws.com.

La segunda ejecucion [40324823709](https://github.com/itba-cloud/2026Q2-G1/actions/runs/40324823709)
reutilizo el estado y devolvio `No changes. Your infrastructure matches the configuration.`

El smoke test comprueba autenticacion, catalogo y conexion Lambda -> RDS.
Las condiciones de carrera y el manejo de pedidos rechazados encontrados en
la revision del PR siguen siendo limitaciones del codigo de la aplicacion.
