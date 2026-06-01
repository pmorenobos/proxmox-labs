# Video 116 - Homelab con Proxmox VE 9.2 desde cero

Guía paso a paso con los comandos y validaciones usados durante el video.

> ⚠️ **Aviso importante**
>
> Esta guía está enfocada en un entorno de **homelab/laboratorio**. Algunos ajustes pueden no ser recomendables para producción sin evaluar riesgos, respaldos, UPS, tolerancia a fallos y criticidad
de las máquinas virtuales.

---

## Paso 1 - Descargar Proxmox VE 9.2

Descargar la imagen ISO oficial de Proxmox VE 9.2 desde el sitio de Proxmox.

> Nota: siempre valida que estés descargando la ISO desde la fuente oficial.

---

## Paso 2 - Crear USB booteable con Rufus

Usar Rufus para grabar la imagen ISO de Proxmox VE en una memoria USB.

Recomendación general:

  https://rufus.ie/es/
- Usar una USB confiable.
- Confirmar que seleccionaste la unidad correcta antes de escribir la imagen.
- Respaldar cualquier dato importante de la USB antes de continuar.

---

## Paso 3 - Preparar BIOS/UEFI para arrancar desde USB

Entrar al BIOS/UEFI del equipo físico y configurar el arranque desde USB.

Validaciones recomendadas:

- Confirmar que el equipo detecta la memoria USB.
- Seleccionar la USB como primer dispositivo de arranque o usar el menú de boot temporal.
- Validar modo de arranque según el hardware disponible.

---

## Paso 4 - Ajuste especial de arranque por problema de resolución

En mi caso, el instalador intenta usar una resolución demasiado alta y sobrepasa los límites de mi monitor. Para esta situación específica, se puede forzar un arranque más conservador agregando 
temporalmente el siguiente parametro en el boot kernel de proxmox, presionando la letra e:

```text
nomodeset vga
```

> ⚠️ **Nota:** este paso no necesariamente aplica para todos. Úsalo solo si el instalador no se visualiza correctamente por problemas de resolución o video.

---

## Paso 5 - Instalación de Proxmox VE

Durante la instalación, en mi caso usaré un arreglo de discos tipo **mirror** para tener tolerancia a fallos.

> ⚠️ **Nota importante:**
>
>Instalar Proxmox sobre un solo disco funciona para laboratorio, pero si ese disco falla, normalmente tendrás que reinstalar o restaurar desde respaldo. 
>Para un entorno más serio, un arreglo tipo mirror o mas avanzados aporta mayor tolerancia a fallos.

---

## Paso 6 - Acceso web y consola SSH

>Después de la instalación, acceder a la interfaz web de Proxmox desde el navegador:

```text
https://IP-PROXMOX:8006
```

Datos de acceso:

```text
Usuario: root
Contraseña: la definida durante la instalación
```

Para acceso por SSH, usar la misma IP del servidor Proxmox:

```bash
ssh root@IP-PROXMOX
```

---

## Paso 7 - Validar conexión a Internet desde terminal

### 7.1 Prueba de conectividad con ping

```bash
ping -c3 freebsd.org
```

### 7.2 Prueba de resolución DNS

```bash
dig proxmox.com
```

### 7.3 Confirmar fecha y hora

```bash
date
```

---

## Paso 8 - Actualizar Proxmox VE

### 8.1 Deshabilitar repositorios enterprise para homelab

```bash
cd /etc/apt/sources.list.d
ls -la
mv pve-enterprise.sources pve-enterprise.sources.disabled
mv ceph.sources ceph.sources.disabled
```

> ⚠️ **Nota:** en un laboratorio es común usar el repositorio `no-subscription`. En producción conviene evaluar una suscripción de Proxmox para soporte y 
>repositorios enterprise.

### 8.2 Agregar repositorio no-subscription

Crear el archivo:

```bash
nano /etc/apt/sources.list.d/pve-no-subscription.sources
```

Agregar el siguiente contenido:

```text
Types: deb
URIs: http://download.proxmox.com/debian/pve
Suites: trixie
Components: pve-no-subscription
Signed-By: /usr/share/keyrings/proxmox-archive-keyring.gpg
```

### 8.3 Ejecutar actualización de repos

```bash
apt update
```

Validar que no aparezcan errores relacionados con repositorios.

### 8.4 Ejecutar actualización general

```bash
apt full-upgrade -y
```

> ⚠️ **Importante:** al finalizar, revisa que no haya errores antes de reiniciar.

### 8.5 Reiniciar Proxmox

```bash
reboot
```

### 8.6 Validar versiones después del reinicio

```bash
pveversion
cat /etc/debian_version
uname -r
```

Valores esperados en mi caso:

```text
Proxmox VE: 9.2.3
Debian: 13.5
Kernel: 7.0.2-6-pve
```

---

## Paso 9 - Crear mirror `vmdata` para almacenamiento de VMs desde GUI

Este paso se realiza desde la interfaz gráfica de Proxmox.

### 9.1 Limpiar discos destinados a VMs

Desde la GUI, hacer wipe de los discos que se usarán para el almacenamiento de máquinas virtuales.

> ⚠️ **Advertencia:** hacer wipe elimina datos del disco. Valida dos veces que seleccionaste los discos correctos.

### 9.2 Crear ZFS Mirror llamado `vmdata`

Crear un pool ZFS tipo mirror para almacenar máquinas virtuales.

Nombre sugerido:

```text
vmdata
```

### 9.3 Validar desde terminal

```bash
zpool status
zpool list
```

---

## Paso 10 - Tuning de discos/ZFS

### 10.1 Revisar estado de los pools

```bash
zpool status
```

### 10.2 Obtener parámetros actuales del pool `vmdata`

```bash
zfs get compression,atime,xattr,sync,acltype,relatime,dnodesize vmdata
```

### 10.3 Modificar parámetros del pool `vmdata`

```bash
zfs set atime=off vmdata
zfs set acltype=posixacl vmdata
zfs set dnodesize=auto vmdata
```

### 10.4 Ajustar `atime` en `rpool`

```bash
zfs set atime=off rpool
```

### 10.5 Revalidar cambios en `vmdata`

```bash
zfs get compression,atime,xattr,sync,acltype,relatime,dnodesize vmdata
```

### 10.6 Activar TRIM en `rpool`

```bash
zpool set autotrim=on rpool
```

> Nota: en esta guía solo se activa `autotrim` en `rpool` ya que son disco modernos, para discos mecanicos no tiene impacto. Consulta en los grupos de apoyo si conviene en tus disco actuales
si tienes dudas.

### 10.7 Validar TRIM en los pools

```bash
zpool get autotrim
```

---

## Paso 11 - Validar estado de discos con SMART

### 11.1 Validar si el servicio `smartd` está operando

```bash
systemctl status smartd
```

### 11.2 Validar salud de discos

```bash
smartctl -H /dev/sda
smartctl -H /dev/sdb
smartctl -H /dev/sdc
smartctl -H /dev/sdd
```

Resultado esperado:

```text
SMART overall-health self-assessment test result: PASSED
```

> Nota: los nombres `/dev/sda`, `/dev/sdb`, etc. pueden cambiar según tu equipo. Valida tus discos con:
>
> ```bash
> lsblk
> ```

---

## Paso 12 - Sensores de temperatura

### 12.1 Instalar `lm-sensors`

```bash
apt install lm-sensors -y
```

### 12.2 Detectar sensores

```bash
sensors-detect
```

Durante el asistente, responder `YES` según corresponda.

### 12.3 Consultar valores actuales

```bash
sensors
```

### 12.4 Opcional: deshabilitar sensor/módulo `jc42` si genera errores

Este paso solo aplica si el sistema muestra errores relacionados con `jc42` en `journalctl`, que fue lo que sucedio con mi hardware.

Editar:

```bash
nano /etc/modules
```

Comentar o remover la carga del módulo si aparece, esto es para mi equipo:

```text
jc42
```

> ⚠️ **Nota:** no apliques este paso si no tienes errores relacionados con `jc42`.

---

## Paso 13 - Habilitar scrub para ZFS

### 13.1 Habilitar scrub mensual para `rpool`

```bash
systemctl enable --now zfs-scrub-monthly@rpool.timer
```

### 13.2 Habilitar scrub mensual para `vmdata`

```bash
systemctl enable --now zfs-scrub-monthly@vmdata.timer
```

### 13.3 Validar timers activos

```bash
systemctl list-timers | grep zfs
```

---

## Paso 14 - Validar RAM para ZFS ARC 10%(32GB)
> ⚠️ **Nota importante:**
>
> Cuando utilizamos ZFS, proxmox tiene como regla utilizar el 10% de la memoria RAM.
### 14.1 Validar ARC

```bash
arc_summary | grep "Max target size:"
```

---

## Paso 15 - Agregar memoria swap

Proxmox no generar swap cuando se instala usando ZFS o btrfs. En este laboratorio agregaré una swap pequeña de **3 GB**.

> ⚠️ **Nota:** la swap no reemplaza tener suficiente RAM física. Este ajuste es solo un apoyo para laboratorio.

### 15.1 Crear volumen ZFS para swap de 3 GB

```bash
zfs create -V 3G -b $(getconf PAGESIZE) \
  -o compression=zle \
  -o logbias=throughput \
  -o sync=always \
  -o primarycache=metadata \
  -o secondarycache=none \
  -o com.sun:auto-snapshot=false \
  rpool/swap
```

### 15.2 Formatear como swap

```bash
mkswap -f /dev/zvol/rpool/swap
```

### 15.3 Habilitar swap

```bash
swapon /dev/zvol/rpool/swap
```

### 15.4 Agregar swap al arranque

Editar:

```bash
nano /etc/fstab
```

Agregar o validar estas líneas:

```text
# <file system> <mount point> <type> <options> <dump> <pass>
proc /proc proc defaults 0 0
/dev/zvol/rpool/swap none swap defaults 0 0
```

### 15.5 Reiniciar

```bash
reboot
```

### 15.6 Validar swap

```bash
swapon --show
free -h
```

---

## Paso 16 - Revisar errores críticos del sistema

```bash
journalctl -p 3 -xb
```

> Nota: no todos los mensajes son necesariamente críticos para tu laboratorio. Hay que interpretar el contexto del error antes de tomar acciones.

---

## Paso 17 - Revisar servicios caídos

```bash
systemctl --failed
```

Resultado esperado:

```text
0 loaded units listed
```

---

## Paso 18 - Confirmar que el sistema está listo para recibir VMs

Checklist rápido:

- Acceso web funcionando.
- Acceso SSH funcionando.
- Internet y DNS funcionando.
- Proxmox actualizado.
- Pools ZFS en estado saludable.
- SMART sin fallos críticos.
- Scrub habilitado.
- Swap validada.
- Sin servicios fallidos.
- Sin errores críticos relevantes en `journalctl`.

---

## Paso 19 - Instalar FreeBSD como VM de prueba

Instalar FreeBSD como máquina virtual sencilla para validar que Proxmox está listo para crear y ejecutar VMs.

> Nota: el objetivo no es enseñar FreeBSD a profundidad, sino confirmar que el homelab de Proxmox quedó funcional.
