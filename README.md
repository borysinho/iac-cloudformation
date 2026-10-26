# Globomatics con AWS CloudFormation

El flujo principal utiliza **dos plantillas independientes: `network.yaml` y `compute.yaml`**, adaptadas a frontend público y backend privado. Primero despliega la red y luego los servidores, pasando las salidas de red como parámetros. Este flujo no utiliza `main.yaml`, empaquetado ni un bucket S3 de plantillas. No necesita ensamblado ni Python.

`main.yaml` se conserva como alternativa opcional para quien quiera administrar todo mediante stacks anidados. Su uso requiere empaquetar las plantillas hijas en S3.

| Plantilla | Recursos | Entradas y salidas |
|---|---|---|
| `network.yaml` | VPC, subred pública, subred privada, tabla pública con ruta a Internet, tabla privada local, Internet Gateway y dos grupos de seguridad | Recibe proyecto, entorno y CIDR SSH del bastión; devuelve VPC, ambas subredes y grupos |
| `compute.yaml` | Clave pública importada y dos EC2 con sus discos raíz | Recibe proyecto, entorno, ambas subredes, grupos, clave pública y parámetro AMI; devuelve identificadores, direcciones y comandos SSH |
| `main.yaml` (opcional) | Stack anidado de red y stack anidado de servidores | Recibe los parámetros comunes, conecta las salidas de red y publica las salidas de ambas plantillas |

**Orden del flujo principal:** crea `network.yaml`, espera a que termine y pasa sus salidas a `compute.yaml`. No hay `Export`/`ImportValue`: debes administrar los parámetros y el orden entre los dos stacks. Ninguna de estas dos plantillas depende de `main.yaml`.

El README de OpenTofu indica que ese laboratorio fue retirado y recreado con Pulumi. Estas plantillas crean infraestructura nueva; no importan ni toman control de recursos existentes. No administres un mismo recurso con dos herramientas. Puedes utilizar otro `Environment` para diferenciar las etiquetas de un laboratorio paralelo.

## Qué se simplificó

- Se eliminaron `config.yaml` y `outputs.yaml`: sus parámetros y salidas pertenecen ahora a las plantillas que los utilizan.
- Se eliminaron el ensamblador, el generador de parámetros, `requirements.txt` y la plantilla generada de `dist/`. La consola y AWS CLI aceptan directamente los dos YAML.
- Se omitió el Resource Group: los stacks identifican sus recursos sin un grupo adicional.
- Se eliminaron los dos launch templates. IMDSv2, interfaces y discos se declaran directamente en `AWS::EC2::Instance`.
- Los discos heredan las etiquetas del EC2 mediante `PropagateTagsToVolumeOnCreation`, incluido `Name=ec2-cliente-...` o `ec2-backend-...`. Ya no tienen nombres separados `ebs-...`.
- Los grupos de seguridad reciben nombres físicos generados por CloudFormation y conservan etiquetas `Name` legibles. `AWS::EC2::KeyPair` requiere un nombre: se utiliza `key-${AWS::StackName}` para distinguir cada stack de servidores.
- No se trasladaron el backend S3, bloqueo, estado, restricciones del proveedor ni políticas IAM de OpenTofu. CloudFormation administra el estado; la identidad y región se seleccionan con AWS CLI.
- Se mantienen descripciones y metadatos breves para explicar decisiones y restricciones. No crean infraestructura adicional.

## Configuración de red y servidores

| Elemento | Valor |
|---|---|
| Región de referencia | `us-east-2`, seleccionada al desplegar |
| Proyecto / entorno | `globomatics` / `lab` por defecto en ambas plantillas |
| VPC | `10.42.0.0/16`; DNS habilitado |
| Subred pública | `10.42.1.0/24`, frontend y bastión |
| Subred privada | `10.42.2.0/24`, backend |
| Zona | Primera devuelta por `Fn::GetAZs`; ambas subredes están en la misma zona |
| Internet público | Ruta `0.0.0.0/0` de la subred pública al Internet Gateway |
| Internet privado | **Sin Internet:** tabla privada con solo la ruta local implícita de la VPC, sin ruta predeterminada ni NAT |
| Cliente | `t3.micro`, privada `10.42.1.10`, pública dinámica, raíz gp3 cifrada de 12 GiB |
| Backend | `t3.micro`, privada `10.42.2.20`, **sin IP pública**, raíz gp3 cifrada de 20 GiB |
| Sistema | Amazon Linux 2023 x86_64, familia kernel 6.18, mediante parámetro SSM |
| Metadatos EC2 | Endpoint habilitado y tokens IMDSv2 obligatorios |
| SSH al frontend | TCP 22 desde `SshSourceCidr`; por defecto `0.0.0.0/0`, cualquier IPv4 |
| SSH al backend | TCP 22 solo desde el grupo del frontend, que actúa como bastión |
| Web | TCP 80 y 443 públicos solo en el cliente |
| API | TCP 8090 al backend desde el grupo del cliente, por IP privada |
| Salida frontend | IPv4 hacia Internet para actualizaciones y configuración manual |
| Salida backend | Solo destinos alcanzables dentro de la VPC; sin salida a Internet |
| Base de datos | No se crea un servicio ni se abren puertos de base de datos |

Solo el frontend tiene IP pública explícita. El backend tiene únicamente IP privada y no dispone de salida a Internet. Se importa solo la clave pública existente, nunca la privada. No se incluye `UserData`, software, roles IAM ni instalación de la aplicación; consulta `../asistencia/docs/despliegue-manual-aws.md` para configurarla por SSH.

Las reglas de seguridad se adaptan al acceso mediante bastión y conservan la API privada en TCP 8090. No se mantienen las etiquetas individuales `sgr-*` de OpenTofu: las reglas declaradas dentro del grupo no admiten etiquetas. `ManagedBy` pasa a `CloudFormation`. `Fn::GetAZs` puede devolver zonas diferentes de la consulta de OpenTofu según las subredes predeterminadas disponibles.

## Comunicación y acceso SSH

```text
Internet ── HTTP/HTTPS ──> frontend 10.42.1.10 (IP pública)
Internet ── SSH :22 ─────> frontend como bastión
                           ├── API :8090 ──> backend 10.42.2.20 (sin IP pública)
                           └── SSH :22 ────> backend 10.42.2.20
backend ──> ruta local de la VPC; sin salida a Internet
```

El requisito de SSH desde cualquier IP se implementa en el **punto de entrada público, el frontend**. Al llegar al backend, la conexión SSH proviene del bastión; su grupo permite TCP 22 y TCP 8090 únicamente desde el grupo del frontend. No se abre TCP 22 del backend a Internet. La referencia de grupo permite los recursos que tengan asociado ese grupo, no exclusivamente una IP concreta. Cualquier otro EC2 privado que deba seguir esta política debe utilizar `BackendSecurityGroupId`.

Desde tu equipo, con la clave privada local y sin copiarla al frontend:

```bash
ssh -i ~/globomatics-ec2 \
  -o "ProxyCommand=ssh -i ~/globomatics-ec2 -W %h:%p ec2-user@IP_PUBLICA_FRONTEND" \
  ec2-user@10.42.2.20
```

La salida `BackendSshCommand` genera ese comando con los valores del stack. El frontend debe permitir reenvío TCP en su configuración SSH para que funcione el bastión; no se instala ni modifica `sshd` mediante estas plantillas. No se requiere copiar claves privadas ni habilitar agent forwarding.

La API continúa escuchando en TCP 8090. Al seguir la guía histórica `../asistencia/docs/despliegue-manual-aws.md`, **sustituye la antigua IP del backend `10.42.1.20` por `10.42.2.20`** en `BACKEND_HOST`, en `proxy_pass` de Nginx y en las pruebas. MySQL permanece en `127.0.0.1:3306`; no se abre el puerto 3306 entre subredes. El navegador utiliza `/api` en el frontend, que reenvía la petición por la red privada.

No se crean NAT Gateway ni Elastic IP. La tabla privada tiene únicamente la ruta local que AWS agrega a toda tabla de la VPC; no tiene ruta predeterminada ni al Internet Gateway ni a un NAT. Las dos subredes tienen tabla de rutas propia. Las reglas de salida del grupo backend no crean conectividad a Internet: sin una ruta de salida el tráfico no puede alcanzarlo.

El backend no podrá ejecutar directamente instalaciones o actualizaciones desde repositorios de Internet (`dnf`, `npm`, descargas de Git, etc.). Para configurarlo utiliza una AMI con las dependencias preparadas, repositorios internos accesibles o transfiere los paquetes y artefactos desde tu equipo mediante el bastión. La guía histórica de instalación debe adaptarse a este funcionamiento sin Internet; estas plantillas no habilitan proxies ni reenvío de Internet a través del frontend.

## Desplegar desde la consola

Los pasos siguientes corresponden al flujo principal con dos stacks independientes. Puedes cargar directamente `network.yaml` y `compute.yaml` en la consola; no necesitas `main.yaml` ni un bucket S3 de plantillas.

1. En `us-east-2`, crea un stack con `network.yaml`. Indica `Project`, `Environment` y `SshSourceCidr`. Su valor predeterminado `0.0.0.0/0` permite SSH al frontend desde cualquier IPv4; puedes restringirlo con tu IP y `/32`.
2. Espera `CREATE_COMPLETE`. Copia las salidas `PublicSubnetId`, `PrivateSubnetId`, `ClientSecurityGroupId` y `BackendSecurityGroupId`.
3. Crea otro stack con `compute.yaml`. Pega esas salidas en los parámetros del mismo nombre y utiliza el mismo proyecto y entorno para conservar etiquetas coherentes.
4. En `SshPublicKey`, pega el contenido de tu archivo público OpenSSH RSA o Ed25519. Conserva el parámetro SSM predeterminado salvo que quieras cambiar deliberadamente la familia de AMI.
5. Revisa los recursos y crea el stack. Sus salidas muestran las direcciones y comandos SSH.

Las dos subredes y ambos grupos deben pertenecer a la misma VPC y región. La pública debe contener `10.42.1.10` y la privada `10.42.2.20`; pasar identificadores de otra red puede impedir el lanzamiento. Los tipos de parámetros AWS comprueban identificadores, pero no garantizan por sí solos estas relaciones.

## Preparar y revisar stacks independientes con AWS CLI

Los comandos siguientes son instrucciones de uso; no se ejecutaron durante la conversión. Requieren AWS CLI y un perfil autorizado. Comprueba que la cuenta sea `222234418877`, la del código de referencia, antes de continuar:

```bash
export AWS_PROFILE=mi-perfil
aws sts get-caller-identity --query '{Account:Account,Arn:Arn}'

aws cloudformation validate-template \
  --region us-east-2 --template-body file://network.yaml
aws cloudformation validate-template \
  --region us-east-2 --template-body file://compute.yaml
```

Prepara la propuesta de red; el ejemplo mantiene SSH al bastión desde cualquier IPv4:

```bash
aws cloudformation deploy \
  --region us-east-2 \
  --stack-name globomatics-cfn-network \
  --template-file network.yaml \
  --parameter-overrides \
    Project=globomatics Environment=lab SshSourceCidr=0.0.0.0/0 \
  --no-execute-changeset
```

`--no-execute-changeset` prepara el change set sin aprovisionar los recursos. Revisa el change set en la consola y ejecútalo cuando corresponda, o utiliza el comando `execute-change-set` que devuelve AWS CLI. Después, espera a que la red esté completamente creada y recoge sus salidas:

```bash
aws cloudformation wait stack-create-complete \
  --region us-east-2 --stack-name globomatics-cfn-network

PUBLIC_SUBNET_ID=$(aws cloudformation describe-stacks \
  --region us-east-2 --stack-name globomatics-cfn-network \
  --query "Stacks[0].Outputs[?OutputKey=='PublicSubnetId'].OutputValue | [0]" --output text)
PRIVATE_SUBNET_ID=$(aws cloudformation describe-stacks \
  --region us-east-2 --stack-name globomatics-cfn-network \
  --query "Stacks[0].Outputs[?OutputKey=='PrivateSubnetId'].OutputValue | [0]" --output text)
CLIENT_SECURITY_GROUP_ID=$(aws cloudformation describe-stacks \
  --region us-east-2 --stack-name globomatics-cfn-network \
  --query "Stacks[0].Outputs[?OutputKey=='ClientSecurityGroupId'].OutputValue | [0]" --output text)
BACKEND_SECURITY_GROUP_ID=$(aws cloudformation describe-stacks \
  --region us-east-2 --stack-name globomatics-cfn-network \
  --query "Stacks[0].Outputs[?OutputKey=='BackendSecurityGroupId'].OutputValue | [0]" --output text)
```

En la misma terminal, prepara los servidores usando tu clave **pública** existente:

```bash
aws cloudformation deploy \
  --region us-east-2 \
  --stack-name globomatics-cfn-compute \
  --template-file compute.yaml \
  --parameter-overrides \
    Project=globomatics Environment=lab \
    PublicSubnetId="$PUBLIC_SUBNET_ID" \
    PrivateSubnetId="$PRIVATE_SUBNET_ID" \
    ClientSecurityGroupId="$CLIENT_SECURITY_GROUP_ID" \
    BackendSecurityGroupId="$BACKEND_SECURITY_GROUP_ID" \
    SshPublicKey="$(cat ~/globomatics-ec2.pub)" \
  --no-execute-changeset
```

Revisa y ejecuta ese change set. Luego:

```bash
aws cloudformation wait stack-create-complete \
  --region us-east-2 --stack-name globomatics-cfn-compute
aws cloudformation describe-stacks \
  --region us-east-2 --stack-name globomatics-cfn-compute \
  --query 'Stacks[0].Outputs' --output table
```

Los comandos SSH de las salidas suponen la privada `~/globomatics-ec2`; ajusta esa ruta si utilizaste otra clave. No es necesario crear recursos IAM ni indicar capacidades IAM.

## Actualizar y retirar

**Migración desde la versión con una sola subred:** el cambio de subred e IP del backend implica reemplazar su EC2. Como su disco raíz se elimina con la instancia, respalda los datos antes de ejecutar el change set. La salida `BackendPublicIp` se elimina y se añade `PrivateSubnetId`; ajusta cualquier integración que consumiera esas salidas. Esta edición solo prepara las plantillas y no aplica la migración en AWS.

Para actualizar, edita la plantilla correspondiente y prepara otro change set con `deploy --no-execute-changeset`. Conserva los valores deseados de los parámetros y revisa especialmente los reemplazos. `LatestAmiId` es un nombre de parámetro SSM: una nueva AMI puede provocar el reemplazo de instancias al actualizar el stack.

Con stacks independientes, si cambias o reemplazas la subred o los grupos de red, comprueba las nuevas salidas y actualiza los parámetros de servidores. En ese modo no hay propagación automática ni protección mediante `ImportValue`. AWS también puede impedir borrar recursos de red mientras los EC2 los utilizan. Retira primero el stack de servidores y después el de red. Con `main.yaml`, las referencias entre hijos las gestiona el principal.

Como en OpenTofu, `DeleteOnTermination: true` elimina los discos raíz al terminar o reemplazar los EC2. **Respalda los datos de la aplicación, incluido MySQL si se instaló manualmente, antes de reemplazar o borrar servidores.** Una IP privada fija puede impedir crear un reemplazo antes de eliminar la instancia anterior; prevé la interrupción.

Las IP públicas son dinámicas y pueden cambiar tras detener/iniciar un servidor. Consulta EC2 para las direcciones vigentes: las salidas de CloudFormation no se refrescan automáticamente ante cambios externos. EC2, EBS y la IPv4 pública del frontend pueden generar cargos al desplegar. El laboratorio depende de una única zona; no se crean NAT ni Elastic IP.

## Alternativa opcional: desplegar todo con main.yaml

Utiliza este modo para un laboratorio nuevo administrado desde un solo stack. No ejecutes además el despliegue independiente para ese mismo laboratorio: el principal crearía recursos nuevos, no adoptaría stacks existentes.

Desde esta carpeta, selecciona tu perfil, comprueba la cuenta y define un bucket S3 existente de plantillas en `us-east-2`, con permisos para cargar y leer los objetos. El bucket almacena plantillas, **no el estado de OpenTofu**. Conserva sus objetos para futuras operaciones del stack; el laboratorio no crea ni elimina este bucket.

```bash
export AWS_PROFILE=mi-perfil
aws sts get-caller-identity --query '{Account:Account,Arn:Arn}'
export CFN_TEMPLATE_BUCKET=mi-bucket-de-plantillas
mkdir -p dist

aws cloudformation package \
  --region us-east-2 \
  --template-file main.yaml \
  --s3-bucket "$CFN_TEMPLATE_BUCKET" \
  --s3-prefix globomatics/templates \
  --output-template-file dist/main.packaged.yaml

aws cloudformation validate-template \
  --region us-east-2 \
  --template-body file://dist/main.packaged.yaml

aws cloudformation deploy \
  --region us-east-2 \
  --stack-name globomatics-cfn-lab \
  --template-file dist/main.packaged.yaml \
  --parameter-overrides \
    Project=globomatics Environment=lab SshSourceCidr=0.0.0.0/0 \
    SshPublicKey="$(cat ~/globomatics-ec2.pub)" \
  --no-execute-changeset
```

Comprueba que la identidad pertenezca a la cuenta esperada `222234418877` y sustituye el bucket del ejemplo antes de ejecutar. El CIDR SSH permite cualquier IPv4; cámbialo a tu IP `/32` si quieres restringir el acceso al bastión. `package` sube únicamente las plantillas hijas referenciadas y reemplaza sus rutas por URL de S3 en la copia generada; no modifica los tres YAML originales. Revisa el change set y ejecuta el comando `execute-change-set` devuelto por AWS CLI cuando corresponda. Después:

```bash
aws cloudformation wait stack-create-complete \
  --region us-east-2 --stack-name globomatics-cfn-lab
aws cloudformation describe-stacks \
  --region us-east-2 --stack-name globomatics-cfn-lab \
  --query 'Stacks[0].Outputs' --output table
```

El parámetro `LatestAmiId` del principal es un **String con el nombre del parámetro SSM**. Se pasa a `compute.yaml`, donde el tipo `AWS::SSM::Parameter::Value<AWS::EC2::Image::Id>` lo resuelve. Así se evita intentar resolver dos veces la AMI.

Para actualizar, vuelve a ejecutar `package` y prepara el cambio sobre el mismo stack principal. Gestiona los stacks hijos desde ese principal. La eliminación respeta la dependencia y retira primero los servidores y luego la red; se mantiene el borrado de discos raíz descrito más abajo. `main.yaml` no requiere macros ni recursos IAM.

## Validación

Se revisaron por separado la sintaxis YAML, las referencias internas, las dependencias, el contrato entre salidas y parámetros, las dos tablas de rutas, la ausencia de IP pública en el backend y las reglas de acceso API/SSH a través del frontend. Esto no sustituye la validación de esquemas ni una ejecución en AWS.

Si tienes `cfn-lint` instalado, valida **cada plantilla directamente**:

```bash
cfn-lint --template network.yaml compute.yaml --regions us-east-2
# Solo si vas a utilizar el coordinador opcional:
cfn-lint --template main.yaml --regions us-east-2
```

`cfn-lint` y `cfn-guard` no están instalados en el entorno utilizado; no se instalaron herramientas ni se evaluó cumplimiento con Guard. No se realizaron operaciones AWS, comprobaciones de AMI, permisos o cuotas. Las entradas HTTP/HTTPS públicas y el SSH público del frontend son intencionales. El backend permite únicamente TCP 22 y 8090 desde el grupo del frontend.

Referencias oficiales: [subredes privadas y NAT](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-example-private-subnets-nat.html), [tablas de rutas](https://docs.aws.amazon.com/vpc/latest/userguide/subnet-route-tables.html), [empaquetado de plantillas locales](https://docs.aws.amazon.com/cli/latest/reference/cloudformation/package.html), [stacks anidados y salidas](https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/aws-resource-cloudformation-stack.html), [EC2 y propagación de etiquetas a volúmenes](https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/aws-resource-ec2-instance.html), [IMDSv2 directamente en EC2](https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/aws-properties-ec2-instance-metadataoptions.html) e [importación de una clave pública](https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/aws-resource-ec2-keypair.html).
