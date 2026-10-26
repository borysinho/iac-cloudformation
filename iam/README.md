# Rol compartido de CloudFormation para network.yaml y compute.yaml

En el asistente Git Sync mostrado en esta conversación, el rol de servicio del stack es obligatorio. Es distinto del rol que sincroniza el repositorio.

## Crear desde la consola

1. En IAM, abre **Roles → Crear rol**. Elige entidad de confianza **Servicio de AWS** y caso de uso **CloudFormation**. Como alternativa, utiliza **Política de confianza personalizada** y pega `cloudformation-trust.json`.
2. Puedes continuar sin políticas adjuntas durante la creación y asignar el nombre `globomatics-network-execution`.
3. Abre el rol creado → **Permisos → Agregar permisos → Crear política insertada**. Selecciona el editor JSON y pega `network-execution-policy.json`. Guarda con el nombre `globomatics-network-resources`.
4. Regresa a CloudFormation, actualiza la lista de roles con el botón circular y selecciona `globomatics-network-execution`. Puedes seleccionar ese mismo rol al crear el stack de `compute.yaml`. Aunque el nombre conserve `network`, la política ahora cubre ambos stacks.

`cloudformation-trust.json` define quién puede asumir el rol. `network-execution-policy.json` define qué operaciones puede ejecutar. No intercambies estos archivos en los editores.

## Si el rol ya existe

En IAM abre `globomatics-network-execution` → **Permisos** → política insertada `globomatics-network-resources` → **Editar → JSON**. Reemplaza su documento completo por el contenido actualizado de `network-execution-policy.json` y guarda. No hace falta recrear el rol, cambiar su confianza ni adjuntar una segunda política. La edición de este archivo local o su publicación en Git no actualiza automáticamente la política que creaste manualmente en IAM.

## Alcance

La política está preparada para `network.yaml` y `compute.yaml`, la cuenta de referencia `222234418877` y la región `us-east-2`. Comprueba que sean los de tu despliegue antes de utilizarla. Conserva los permisos anteriores de red y agrega:

- Consultas de imágenes, instancias, discos y claves públicas.
- Lanzamiento, inicio, parada, reinicio, terminación y modificación de instancias, incluidas las opciones IMDSv2.
- Gestión de discos e interfaces de red, y etiquetas de los recursos de cómputo.
- Importación y eliminación de claves públicas con nombres `key-*`, como los generados por `compute.yaml`. No concede `CreateKeyPair`: se sigue importando solo la pública.
- Lectura del parámetro público SSM exacto de la AMI configurada. Su ARN tiene el campo de cuenta vacío porque pertenece al catálogo público de AWS. Si cambias `LatestAmiId`, debes ajustar este permiso.
- Eliminación de parámetros SSM limitados a `/ec2/keypair/*` en la cuenta del proyecto, para la limpieza de claves del proveedor CloudFormation. No concede escritura ni lectura de claves privadas en esa ruta.

Las consultas EC2 usan `Resource: "*"`; las escrituras se limitan a tipos de recursos de esa cuenta y región. Se permite referenciar imágenes y snapshots en esa región con los ARN de cuenta vacía que define EC2. Los identificadores son desconocidos antes de crear los recursos, por lo que sus sufijos usan comodines. Esto permite actuar sobre otros recursos de esos mismos tipos en esa cuenta y región; no está aislado exclusivamente por stack.

No concede crear NAT, roles IAM ni perfiles de instancia. `compute.yaml` no adjunta un perfil IAM a sus EC2, por lo que no se concede `iam:PassRole` hacia EC2. Este rol es para CloudFormation y no se adjunta a las instancias. Los discos usan el cifrado EBS predeterminado; si cambias a una clave KMS personalizada, serán necesarios permisos sobre esa clave.

La identidad que pasa el rol a CloudFormation necesita `iam:PassRole` para el ARN real del rol. En Git Sync, comprueba también este permiso en el rol de sincronización que invoca las operaciones del stack. El permiso debe limitarse a ese ARN y al servicio `cloudformation.amazonaws.com`; no concederlo a todos los roles. Si la consola genera o actualiza permisos automáticamente, revisa el resultado antes de agregar permisos duplicados.

Los archivos se comprobaron como JSON localmente y las acciones se contrastaron con las referencias programáticas oficiales de EC2 y SSM. No se crearon roles en AWS ni se validó la política con IAM Access Analyzer ni mediante un despliegue. Si la cuenta impone límites de permisos o SCP, también deben permitir las operaciones.

Referencias: [rol de servicio CloudFormation](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-iam-servicerole.html), [creación de roles de servicio](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_create_for-service.html).
