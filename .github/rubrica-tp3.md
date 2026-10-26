# TP3: evidencia para completar la rubrica

Esta es una verificacion tecnica, no una nota asignada por la catedra.

Fuentes revisadas:

- Aplicacion y Terraform: PR #5, commit `5532dcfc3a954476f130b7ff340f4b4cf1e47048`.
- Pipeline con apply y estado S3: PR #7, commit `38522560e434b3afb4c2e8cfd4e8d79241c39898`.
- Ejecucion real de apply: https://github.com/itba-cloud/2026Q2-G1/actions/runs/40323713204

| Criterio | Maximo | Evidencia | Estado |
|---|---:|---|---|
| Modulo custom | 2 | `terraform/modules/lambda_function`: parametros para codigo, rol, timeout, entorno, VPC y layers; outputs ARN, invoke ARN y nombre. Se reutiliza 19 veces. | Implementado; las funciones se crearon en el apply real. |
| Funciones y meta-argumentos | 0,5 | Funciones: `jsonencode`, `jsondecode`, `templatefile`, `merge`, `concat`, `cidrsubnet`, entre otras. Meta-argumentos: `for_each`, `count`, `depends_on` y `lifecycle`. | Cumple las cantidades pedidas y tiene usos concretos. |
| Modulo externo | 1 | `terraform-aws-modules/vpc/aws`, version `~> 6.7`: VPC, subredes de aplicacion/datos, tablas de rutas y NACL. | Implementado; aporta la red compartida. |
| Locals y datasources | 0,5 | Locals para AZ/subredes, roles, entorno compartido, rutas, consumidores y reintentos. Datasources de AZ, LabRole, tags, documentos IAM, archives y CA de RDS. | Implementados con usos concretos. |
| README | 2 | El README del PR #5 explica modulos, funciones, meta-argumentos y ejecucion local. `.github/terraform-manual.md` del PR #7 documenta plan/apply/destroy y el estado remoto. | Parcial para la entrega: el README principal todavia afirma que apply no corre en Actions; hay que incorporar la guia nueva. |
| Terraform apply corre de una | 2 | Ejecucion 40323713204: `Apply complete! Resources: 162 added, 0 changed, 0 destroyed.` Ejecucion 40324823709: `No changes. Your infrastructure matches the configuration.` Estado S3 privado/versionado, bloqueo y concurrencia serializada. | Apply real exitoso y segunda ejecucion sin cambios. El intento anterior fallo por credenciales vencidas antes de crear recursos. |
| TP anda | 2 | En AWS pasaron 401 sin token, login Cognito y catalogo/productos via API/Lambda/RDS, con tres comercios. La revision local comprobo el ciclo crear/asignar/entregar/calificar con PostGIS y AWS simulado. | Parcial: API y catalogo reales comprobados; falta probar el recorrido completo cliente/repartidor en AWS. |

## Otros requisitos de la consigna

- Variables y outputs: presentes en el modulo raiz y el modulo propio, con defaults y validaciones. Las contrasenas demo son outputs sensibles.
- Estructura y DRY: Terraform dividido por capas en `terraform/organization`; modulo Lambda compartido, layers, locals y `for_each` para evitar repetir configuracion.
- Diagrama: `arquitectura-aws.drawio` esta alineado con los componentes principales de la aplicacion, la VPC sin NAT y las dos state machines. Falta representar el bucket de estado del pipeline nuevo.
- Pipeline en el PR: el pipeline original del PR #5 todavia tiene fallos conocidos (plan sin secrets en PR desde fork, checkout de docs y alertas de seguridad). La ejecucion manual exitosa no reemplaza esos checks del PR.
- Seguridad: tfsec reporta 2 HIGH, 3 MEDIUM y 41 LOW. Los HIGH son el topic de alertas sin cifrado y el uso de una clave administrada por AWS en el topic de estados. Revisar y justificar las decisiones del Lab, sin esconder alertas para obtener un check verde.
- Hash del campus: todavia no cerrar la entrega. El codigo de la aplicacion y el workflow actualizado estan en dos PR/commits distintos; definir un commit final que reuna la entrega y cuya ejecucion se pueda reproducir.
- Campos de aprobacion/email de la planilla: completar con los datos del grupo y el resultado de la catedra; no se infieren del repositorio.

## Defectos conocidos del codigo revisado

1. Una busqueda concurrente puede terminar asignando dos pedidos al mismo repartidor.
2. El reintento de `crear-pedido` para un ID existente no devuelve `total`, que Step Functions espera.
3. Un pedido rechazado no queda persistido para la API, y el frontend puede seguir esperando indefinidamente.

Estos defectos fueron reproducidos en la revision y no estan corregidos por el cambio de pipeline.

## Despliegue comprobado

- Cuenta AWS: `712638924387`.
- Region: `us-east-1`.
- API: https://nmvfaa3yhe.execute-api.us-east-1.amazonaws.com
- User Pool: `us-east-1_v2d8BmjQx`.
- Configuracion publica del frontend: artefacto `frontend-config` de la ejecucion 40323713204.
- Persistencia e idempotencia: https://github.com/itba-cloud/2026Q2-G1/actions/runs/40324823709
- Las contrasenas demo y el estado no estan publicados como artefactos.
