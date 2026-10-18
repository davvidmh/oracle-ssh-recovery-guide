# Recuperación de SSH en Oracle Cloud sin publicar datos privados

Guía general para recuperar acceso a una VM Linux de OCI cuando se pierde su clave privada. Todos los identificadores, nombres y rutas de los ejemplos son ficticios o marcadores. Esta guía no contiene claves ni información de una cuenta real.

## Estado de validación

**Recuperación permanente comprobada.** Se accedió mediante una sesión Managed SSH de Bastion con una clave temporal distinta; se instaló la clave permanente en un archivo independiente y se configuró SSH para consultarlo solo para el usuario objetivo. La conexión directa con la clave permanente pasó **después de confirmar los bastiones eliminados y el plugin desactivado**. También se retiraron la política IAM y el grupo dinámico temporales. Se conservó la verificación contra la identidad del servidor registrada anteriormente y la autenticación por contraseña permaneció desactivada.

La primera prueba había dado un éxito transitorio: usar la misma clave de Bastion en el archivo que gestiona el agente no conservó el acceso al terminar la limpieza. Por eso el procedimiento exige separar las claves y verificar después del cierre completo. El resultado observado no garantiza compatibilidad con cualquier imagen o configuración.

**Run Command no fue la vía de recuperación**: el diagnóstico permaneció aceptado sin respuesta y se solicitó su cancelación. No interpretar un comando aceptado por OCI como un comando ejecutado dentro de Linux. El resultado observado con Bastion no garantiza que el procedimiento funcione en cualquier imagen o configuración.

## 1. Identificar el problema

- Consultar en la consola el estado de la VM, la imagen y la dirección de acceso.
- Si falta la clave privada, la parte pública no permite reconstruirla.
- No recrear ni eliminar la VM o su volumen de arranque para recuperar acceso.
- Si la conexión falla por permisos locales, corregir los permisos de la clave; eso es distinto de perderla.

## 2. Preparar una nueva clave permanente

Guardar la clave fuera de Descargas, carpetas sincronizadas y repositorios. Por ejemplo, en Windows:

```powershell
ssh-keygen -t ed25519 -f "$env:USERPROFILE\.ssh\oci-recovery_ed25519" -C "oci-recovery"
```

Antes de ejecutar, crear la carpeta `.ssh` si no existe y comprobar que el archivo de destino no exista: no sobrescribir claves. Elegir una frase de paso y conservarla en un gestor adecuado. Revisar que la clave privada tenga permisos restringidos; no hacer cambios globales a los permisos del perfil de Windows.

La extensión `.pub` corresponde a la clave pública: esa es la única parte que se instala en la VM. La clave privada permanece en el equipo del administrador. Conservar una copia de recuperación cifrada fuera del repositorio.

## 3. Evaluar OCI Run Command

Run Command funciona mediante Oracle Cloud Agent y puede operar sin SSH. Requiere:

1. Plugin Compute Instance Run Command habilitado y funcionando.
2. Permisos del usuario de OCI que envía el comando.
3. Permisos de la identidad de la instancia para recibirlo y devolver su resultado.
4. Permisos del sistema operativo suficientes para la operación solicitada.

Estos son controles diferentes. **IAM no concede automáticamente `sudo` en Linux**. El usuario del plugin es normalmente `ocarun`; no concederle `NOPASSWD:ALL` como solución automática.

### Identidad limitada a una sola VM

Crear un grupo dinámico temporal con una regla como esta, sustituyendo el marcador:

```text
instance.id = '<INSTANCE_OCID>'
```

Usar una política limitada a los comandos de la propia instancia:

```text
Allow dynamic-group id <DYNAMIC_GROUP_OCID> to use instance-agent-command-execution-family in tenancy where request.instance.id=target.instance.id
```

El grupo debe contener exclusivamente la VM objetivo. El ejemplo no concede acceso a objetos, secretos, redes ni otras instancias. Usar un compartimento más acotado cuando corresponda. Confirmar el alcance efectivo de la política antes de aplicarla.

Oracle documenta que, al añadir una instancia existente al grupo dinámico, puede tardar hasta 30 minutos en comenzar a recibir comandos. No enviar repetidamente comandos de modificación durante la propagación.

### Comprobación inicial de solo lectura

```bash
#!/bin/bash
id
uname -m
df -h /
sudo -n -l 2>&1
```

Configurar un tiempo máximo breve, por ejemplo 60 segundos de ejecución. Ese tiempo empieza cuando el plugin recibe el comando; no garantiza entrega en 60 segundos. No imprimir variables de entorno, tokens, claves privadas ni archivos de credenciales.

Un estado `Accepted` no prueba ejecución. Esperar evidencia de entrega, estado de ejecución, código de salida y respuesta. La ausencia de respuesta tampoco prueba por sí sola que el comando haya terminado o fallado.

## Alternativa: sesión SSH administrada de Bastion

Cuando el agente y el plugin Bastion estén disponibles, evaluar una sesión SSH administrada. Esta vía requiere una VM activa, usuario del sistema válido y conectividad entre Bastion y la VM. La posibilidad de recuperar acceso con una clave nueva debe comprobarse en el entorno; crear la sesión no demuestra autenticación.

1. Habilitar el plugin Bastion y verificar su estado en Oracle Cloud Agent.
2. Crear un bastion temporal en la red de la VM. Limitar la allowlist a la IP pública del administrador con prefijo `/32`; mantener SOCKS5 desactivado si no es necesario.
3. Limitar la duración máxima a 30 minutos.
4. Crear una sesión de tipo **Managed SSH**, con el usuario correcto y una clave pública temporal **distinta de la clave permanente** que se instalará después. Verificar la instancia y la dirección privada de destino. La limpieza del agente puede retirar la clave usada por la sesión; no confundir una conexión directa durante esa sesión con acceso permanente.
5. Esperar a que la sesión esté activa y obtener su comando SSH desde la consola. Sustituir el marcador de clave por la ruta local de la clave privada.
6. Verificar la identidad del servidor y demostrar autenticación antes de instalar la clave de forma permanente.

Al concluir, eliminar la sesión y el bastion temporales y restaurar el estado anterior del plugin si se habilitó solo para recuperar acceso. No ampliar la allowlist a todo Internet para resolver un fallo de conexión.

## 4. Instalar la nueva clave pública

Continuar únicamente si se ha confirmado un método administrativo autorizado y suficiente:

- Para Oracle Linux, el usuario habitual es `opc`; Ubuntu suele usar `ubuntu`. Verificarlo en la consola.
- Respaldar el archivo `authorized_keys` existente antes de modificarlo.
- Agregar la clave **pública** nueva de forma idempotente, sin sobrescribir otras claves.
- Mantener propietario correcto, directorio `.ssh` con modo `700`, archivo `authorized_keys` con modo `600` y contexto SELinux adecuado.
- No habilitar contraseñas ni desactivar la verificación de identidad del servidor para resolver este problema.

### Separar las claves permanentes de las gestionadas por el agente

En el entorno observado, el agente mantenía un respaldo de `authorized_keys` y el acceso agregado allí dejó de funcionar al cerrar Bastion. No modificar a ciegas los respaldos del agente. Una alternativa administrativa es conservar el archivo habitual y configurar **otro archivo de claves para el usuario objetivo**, mediante `AuthorizedKeysFile`.

Por ejemplo, después de comprobar la configuración efectiva y las rutas actuales, un bloque al final de `sshd_config` puede limitar el cambio al usuario correspondiente:

```text
Match User <OS_USER>
    AuthorizedKeysFile .ssh/authorized_keys .ssh/authorized_keys_permanent
```

El ejemplo presupone que `.ssh/authorized_keys` es la ruta previa: preservar todas las rutas efectivamente configuradas. Instalar exclusivamente la clave pública permanente en el segundo archivo, con propietario correcto, modo `600` y contexto SELinux apropiado. Respaldar la configuración, validarla con `sshd -t` y comprobar el resultado de `sshd -T -C` para ese usuario **antes de recargar** el servicio. Si la validación falla, restaurar la configuración anterior y no recargar.

Probar la clave permanente en una conexión nueva. Luego eliminar el canal temporal, esperar su estado terminal, restaurar el estado previo del plugin y repetir la conexión. Solo esta última prueba demuestra que el acceso no dependía de Bastion.

Si `ocarun` no tiene permisos suficientes, detener esta vía. Evaluar la consola serie y el procedimiento oficial de recuperación con una ventana de mantenimiento. No ampliar privilegios del agente de forma general ni reemplazar el volumen sin un plan de respaldo.

## 5. Verificar SSH y cerrar la recuperación

```powershell
ssh -i "$env:USERPROFILE\.ssh\oci-recovery_ed25519" opc@<PUBLIC_IP_OR_PRIVATE_HOSTNAME>
```

Comprobar la identidad del servidor por un canal confiable antes de aceptar una huella nueva. Si ya está registrada, exigir coincidencia; no borrar `known_hosts` para evitar una alerta.

La recuperación se considera terminada cuando la nueva clave permite autenticarse en la VM correcta. Después:

1. Retirar la política IAM y el grupo dinámico temporales creados para recuperar acceso.
2. Retirar sesiones o conexiones temporales de consola que se hayan creado.
3. Revisar qué claves antiguas deben revocarse, sin borrar accesos ajenos o aún necesarios.
4. Verificar que los accesos temporales hayan desaparecido.
5. Documentar ubicación segura de la clave y fecha de verificación, sin copiar secretos al informe.

## Uso posterior como servicio privado

Recuperar SSH no configura por sí mismo una red privada. Para un coordinador de proyectos, planificar conectividad privada autenticada y acceso mínimo por proyecto. No cerrar el acceso administrativo vigente hasta verificar una ruta alternativa. No publicar bases de datos, paneles administrativos ni APIs internas sin autenticación.

El límite de almacenamiento Always Free se comparte entre volúmenes de arranque y datos en la región principal. Revisar documentación y consumo actual antes de ampliar. No confundir una ampliación del volumen en OCI con la ampliación de particiones y del sistema de archivos dentro de Linux.

## Qué nunca publicar

- Claves privadas, tokens, cookies o archivos de credenciales.
- IP reales, OCID reales, identificadores de cuenta o huellas reales del entorno.
- Rutas con nombres personales, registros de sesiones o capturas de consola sin depurar.
- Archivos `.env`, respaldos de `authorized_keys`, `known_hosts` o configuraciones SSH personales.

Revisar también los metadatos de commits, archivos preparados para commit e historial Git. El archivo `.gitignore` reduce errores comunes; no sustituye una revisión.

## Fuentes oficiales

- [Running Commands on an Instance](https://docs.oracle.com/en-us/iaas/Content/Compute/Tasks/runningcommands.htm)
- [Working with Instances: políticas de Run Command](https://docs.oracle.com/en-us/iaas/Content/Compute/Tasks/instances.htm#permissions)
- [Troubleshooting the SSH Connection](https://docs.oracle.com/en-us/iaas/Content/Compute/Tasks/troubleshooting-ssh-connection.htm)
- [Creating a Managed SSH Session in Bastion](https://docs.oracle.com/en-us/iaas/Content/Bastion/Tasks/create-session-managed-ssh.htm)
- [Connecting to a Managed SSH Session](https://docs.oracle.com/en-us/iaas/Content/Bastion/Tasks/connect-managed-ssh.htm)
- [OpenSSH: sshd_config y AuthorizedKeysFile](https://man.openbsd.org/sshd_config)
- [Always Free Resources](https://docs.oracle.com/en-us/iaas/Content/FreeTier/freetier_topic-Always_Free_Resources.htm)

Consultar las versiones vigentes de estas páginas antes de aplicar el procedimiento.
