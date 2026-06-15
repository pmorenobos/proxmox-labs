# 117 - Proxmox Homelab: Fallo de Disco y Recuperación

Guía paso a paso del laboratorio donde se valida la salud inicial de los arreglos ZFS, se configuran notificaciones, se simula la falla de un disco del `rpool`, se revisa el comportamiento de Proxmox y se reemplaza el disco para recuperar el mirror.

> ⚠️ **Advertencia importante**  
> Este laboratorio implica manipular discos físicos de un arreglo ZFS Mirror. No ejecutes estos pasos en producción sin respaldo actualizado, ventana de mantenimiento y plena certeza de qué disco vas a retirar o reemplazar.

> 🔐 **Nota de seguridad**  
> Nunca publiques contraseñas, tokens, claves de aplicación o credenciales reales en GitHub. En este documento se usan marcadores como `TU_CORREO@gmail.com` y `APP_PASSWORD_DE_GMAIL`.

---

## Paso 1 - Validar la salud de los arreglos ZFS

Antes de provocar cualquier falla, primero validamos que el sistema esté sano.

### Validar estado de `rpool`

```bash
zpool status rpool
```

### Ver discos detectados por el sistema

```bash
lsblk -o NAME,SIZE,MODEL,SERIAL,FSTYPE
```

### Validar estado de arranque de Proxmox

```bash
proxmox-boot-tool status
```

### Validar discos, seriales, sistemas de archivos y UUID

```bash
lsblk -o NAME,MODEL,SERIAL,FSTYPE,UUID
```

> Nota: aquí es importante identificar claramente qué disco pertenece al mirror, qué partición usa ZFS y qué partición corresponde al arranque EFI.

---

## Paso 2 - Revisar y configurar notificaciones en Proxmox

Ruta en GUI:

```text
Datacenter → Notifications
```

### Concepto de Target

Un **Target** es el destino donde configuramos el medio o mecanismo que utilizará Proxmox para entregar las notificaciones que su framework de notificaciones genere.

Ejemplos de eventos soportados:

- Tareas de backup.
- Replicaciones.
- Migraciones.
- Actualizaciones de tareas.
- Otros eventos soportados por el sistema de notificaciones de Proxmox.

Actualmente Proxmox permite varios tipos de **Notification Target**, entre ellos:

- Gotify.
- Sendmail.
- SMTP.
- Webhook.

En este laboratorio se utiliza un Target de tipo **SMTP**, usando Gmail como relay para enviar las alertas.

> ⚠️ Aclaración importante: no todos los servicios del sistema usan esta configuración. Servicios como `smartd` y ciertos eventos de `ZFS-ZED` continúan dependiendo de un mecanismo de correo configurado a nivel del sistema operativo, por ejemplo Postfix. Esa parte no se configura en este video.

No estamos limitados a un solo Target. Podemos crear varios Targets y combinarlos según la necesidad.

### Concepto de Matcher

Un **Matcher** define la lógica de las notificaciones en Proxmox. Decide qué eventos serán notificados y a qué Target serán enviados.

En otras palabras:

```text
Evento generado → Matcher → Target
```

Los **Targets** son los destinos.  
Los **Matchers** son las reglas que determinan cuándo y dónde entregar cada alerta.

---

## Paso 3 - Preparar Gmail para SMTP

Para crear una clave de aplicación en Gmail es necesario tener habilitada la verificación en dos pasos.

URL:

```text
https://myaccount.google.com/apppasswords
```

Ejemplo de nombre de clave:

```text
ProxMox-Relay1
```

Ejemplo de marcador para contraseña:

```text
APP_PASSWORD_DE_GMAIL
```

> 🔐 No pegues la contraseña real en GitHub, documentación pública, comentarios de YouTube ni capturas visibles.

---

## Paso 4 - Actualizar el correo del usuario `root@pam`

Ruta en GUI:

```text
Datacenter → Permissions → Users → root@pam
```

Configura el correo correcto para el usuario `root@pam`.

### Validar en consola

```bash
pveum user list | grep root@pam
```

---

## Paso 5 - Crear SMTP Target Gmail en Proxmox

Ruta en GUI:

```text
Datacenter → Notifications → Notification Targets → Add → SMTP
```

Parámetros de ejemplo:

```text
Endpoint name = gmail-email
Server        = smtp.gmail.com
Encryption    = STARTTLS
Port          = 587
Authenticate  = Enabled
Username      = TU_CORREO@gmail.com
Password      = APP_PASSWORD_DE_GMAIL
From          = TU_CORREO@gmail.com
Recipient     = root@pam
Comment       = SMTP Gmail para alertas Proxmox y ZFS
Author        = hostname de Proxmox
```

Después de crear el Target, ejecuta la prueba de envío y confirma la recepción del correo.

---

## Paso 6 - Asociar el SMTP Target al Matcher

Ruta en GUI:

```text
Datacenter → Notifications → Notification Matchers → default-matcher → Edit
```

Configurar:

```text
Target to notify = gmail-email
```

Deshabilitar:

```text
mail-to-root
```

---

## Paso 7 - Probar notificaciones con eventos reales

Para validar que las notificaciones funcionan, se puede usar una VM en ejecución, por ejemplo una VM FreeBSD.

### Prueba con backup exitoso

1. Ejecutar un backup manual de la VM.
2. Esperar a que finalice correctamente.
3. Revisar la recepción del correo.

### Prueba con backup fallido

1. Ejecutar un backup manual de la VM.
2. Detenerlo antes de que termine.
3. Confirmar que el backup se registró con falla.
4. Revisar la recepción del correo.

> Nota: esta prueba valida eventos del framework de notificaciones de Proxmox. 
No garantiza que todos los eventos de ZFS, `smartd` o `ZFS-ZED` sean notificados por este mismo mecanismo.

---

## Paso 8 - Identificar el disco desde donde arrancó el sistema

Antes de retirar un disco, identificamos el disco de arranque y los discos que forman parte del mirror.

```bash
efibootmgr -v | egrep "BootCurrent|BootOrder|Linux Boot Manager|proxmox"
```

```bash
efibootmgr -v | grep "Boot[0-9A-F][0-9A-F][0-9A-F][0-9A-F]"
```

Ejemplo filtrando discos y particiones específicas:

```bash
lsblk -o NAME,SIZE,MODEL,SERIAL,FSTYPE,UUID,PARTUUID | grep -E "sda|sda2|sdb|sdb2|PARTUUID_A_BUSCAR"
```

> ⚠️ Ajusta el filtro `PARTUUID_A_BUSCAR` a tu entorno real. No copies este valor a ciegas.

### Disco seleccionado para desconectar

Ejemplo:

```text
serie = SERIAL_DEL_DISCO_A_DESCONECTAR
```

> ⚠️ Esta identificación debe hacerse con extremo cuidado. Retirar el disco equivocado puede dejar el sistema sin arranque o provocar pérdida de datos.

---

## Paso 9 - Romper el arreglo `rpool` para degradar el mirror

Secuencia general:

1. Apagar la VM FreeBSD.
2. Apagar Proxmox.
3. Retirar físicamente el disco seleccionado.
4. Encender nuevamente el equipo.

### Apagar FreeBSD

Dentro de la VM FreeBSD:

```bash
shutdown -p now
```

### Apagar Proxmox

Desde Proxmox:

```bash
shutdown -h now
```

Después de apagar, retirar el disco físico seleccionado e iniciar nuevamente el equipo.

---

## Paso 10 - Validar estado degradado en Proxmox

### Validación en GUI

Ruta:

```text
Datacenter → Node → Disks → ZFS
```

### Validación en consola

```bash
zpool status rpool
```

Resultado esperado:

```text
state: DEGRADED
```

### Revisar eventos de ZFS

```bash
zpool events rpool
```

```bash
zpool events rpool -v
```

### Revisar correos

Revisar si llegó alguna notificación relacionada con el estado degradado del arreglo.

---

## Paso 11 - Observaciones importantes del laboratorio

Durante esta prueba se observó lo siguiente:

1. El arreglo queda degradado, pero la GUI de Proxmox no muestra una alerta visible inmediata en la vista principal.
2. El estado degradado se observa al entrar a los detalles de almacenamiento/ZFS.
3. `ZFS-ZED` no necesariamente genera un evento visible relacionado con esta condición del mirror.
4. No necesariamente llega un correo notificando esta degradación usando solo el framework de notificaciones configurado en la GUI.

> Nota: esta situación no es nueva. Para cubrir este tipo de condición de forma confiable, normalmente se requiere monitoreo adicional mediante scripts, Zabbix, 
Nagios, LibreNMS u otra solución de monitoreo.

---

## Paso 12 - Reemplazo de disco y reparación del mirror `rpool`

> ⚠️ Este proceso se realiza desde consola. No se realiza desde la GUI.

Condiciones del laboratorio:

- Instalación con GPT/UEFI.
- `rpool` sobre ZFS Mirror.
- Disco nuevo limpio, sin particiones ni firmas previas.

Secuencia general:

1. Apagar Proxmox.
2. Insertar el disco nuevo.
3. Validar que BIOS/UEFI detecte el disco.
4. Arrancar Proxmox.
5. Identificar el disco sano y el disco nuevo.
6. Copiar tabla de particiones.
7. Ejecutar `zpool replace`.
8. Restaurar partición de arranque.
9. Validar estado final.

---

## Paso 13 - Identificar el nuevo disco

```bash
lsblk -o NAME,SIZE,MODEL,SERIAL,FSTYPE,UUID,PARTUUID
```

### Validar si el disco está limpio

Ejemplo:

```bash
fdisk -l /dev/sda
```

```bash
wipefs /dev/sda
```

> Nota: si ambos comandos no muestran particiones, firmas o sistemas de archivos previos, el disco puede considerarse limpio para este laboratorio.

---

## Paso 14 - Obtener IDs de discos por marca/modelo

Ejemplo con discos Samsung:

```bash
ls -l /dev/disk/by-id/ | grep Samsung
```

Ejemplo:

```text
Disco sano  = ata-Samsung_SSD_850_EVO_250GB_DISCO_SANO
Disco nuevo = ata-Samsung_SSD_850_EVO_250GB_DISCO_NUEVO
```

> ⚠️ Usa siempre `/dev/disk/by-id/` para reducir el riesgo de equivocarte con nombres variables como `/dev/sda` o `/dev/sdb`.

---

## Paso 15 - Copiar tabla de particiones al disco nuevo

Ejemplo:

```bash
sgdisk /dev/disk/by-id/ata-Samsung_SSD_850_EVO_250GB_DISCO_SANO \
  -R /dev/disk/by-id/ata-Samsung_SSD_850_EVO_250GB_DISCO_NUEVO
```

> ⚠️ El orden importa. El disco sano va primero y el disco nuevo va después de `-R`. Si inviertes el orden, puedes destruir la tabla de particiones del disco correcto.

### Validar tabla de particiones

```bash
fdisk -l /dev/sdb
```

```bash
fdisk -l /dev/sda
```

Las tablas de particiones deben ser equivalentes.

---

## Paso 16 - Reemplazar el disco en el pool ZFS

Primero revisar el estado del pool:

```bash
zpool status rpool
```

Identificar el ID del disco en estado `UNAVAIL`.

Ejemplo:

```text
ID_DISCO_UNAVAIL
```

Ejecutar `zpool replace` usando la partición ZFS del disco nuevo, normalmente `part3` en instalaciones Proxmox con ZFS.

```bash
zpool replace rpool \
  ID_DISCO_UNAVAIL \
  /dev/disk/by-id/ata-Samsung_SSD_850_EVO_250GB_DISCO_NUEVO-part3
```

### Validar resilver

```bash
zpool status rpool
```

Durante el proceso debería observarse el avance de `resilver`.

---

## Paso 17 - Validar estado en GUI y correo

Validar en GUI:

```text
Datacenter → Node → Disks → ZFS
```

Revisar si llega correo relacionado con el proceso de recuperación o resilver.

Una vez terminado el proceso, el resultado esperado es:

```text
state: ONLINE
```

---

## Paso 18 - Revisar estado de arranque de Proxmox

```bash
proxmox-boot-tool status
```

Si aparece información del disco dañado o retirado, revisar el archivo:

```bash
nano /etc/kernel/proxmox-boot-uuids
```

> ⚠️ Edita este archivo con cuidado. Elimina solo UUIDs obsoletos o correspondientes al disco retirado, si aplica.

---

## Paso 19 - Restaurar partición de arranque en el disco nuevo

Identificar la partición EFI del disco nuevo. Normalmente es la partición de 1 GB, por ejemplo `/dev/sda2`, pero valida siempre en tu entorno.

```bash
lsblk -o NAME,SIZE,MODEL,SERIAL,FSTYPE,UUID,PARTUUID
```

### Crear sistema de archivos VFAT en la partición EFI

Ejemplo:

```bash
mkfs.vfat -F 32 /dev/sda2
```

> ⚠️ Verifica que `/dev/sda2` sea realmente la partición EFI del disco nuevo. No ejecutes este comando sobre la partición incorrecta.

### Inicializar Proxmox Boot Tool en la nueva partición

```bash
proxmox-boot-tool init /dev/sda2
```

### Revalidar estado de arranque

```bash
proxmox-boot-tool status
```

### Validar UUIDs

```bash
lsblk -o NAME,SIZE,MODEL,FSTYPE,UUID
```

El UUID mostrado debe coincidir con la información almacenada y administrada por `proxmox-boot-tool`.

---

## Paso 20 - Validación final

Validar que el pool volvió a estado sano:

```bash
zpool status rpool
```

Validar estado de arranque:

```bash
proxmox-boot-tool status
```

Validar discos:

```bash
lsblk -o NAME,SIZE,MODEL,SERIAL,FSTYPE,UUID,PARTUUID
```

Resultado esperado:

```text
rpool ONLINE
mirror ONLINE
proxmox-boot-tool sin referencias inválidas al disco retirado
```

---

## Fin del laboratorio

Con esto el arreglo `rpool` vuelve a estar sano, el disco nuevo queda integrado al mirror y el sistema recupera la redundancia tanto a nivel ZFS como a nivel de arranque UEFI.

