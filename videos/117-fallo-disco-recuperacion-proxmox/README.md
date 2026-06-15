
# 117 - Proxmox Homelab: Fallo de Disco y Recuperación

Material complementario del video donde se simula la falla de un disco en un arreglo ZFS Mirror de Proxmox VE, se valida el estado degradado del sistema, se revisan las notificaciones y se realiza el reemplazo del disco hasta recuperar el arreglo.

## Objetivo del laboratorio

Validar qué sucede cuando falla un disco que forma parte del `rpool` en Proxmox VE, revisar el comportamiento del sistema, comprobar las alertas disponibles y documentar el proceso de reemplazo del disco usando ZFS y `proxmox-boot-tool`.

## Archivo principal

- `pasos-y-comandos.md`: guía completa con pasos, comandos, validaciones y observaciones usadas durante el video.

## Temas cubiertos

- Validación inicial de `rpool` y discos.
- Configuración de notificaciones SMTP en Proxmox.
- Pruebas de notificación con eventos reales.
- Simulación de falla de disco en ZFS Mirror.
- Validación del estado `DEGRADED`.
- Reemplazo de disco en `rpool`.
- Resilver del arreglo ZFS.
- Restauración del arranque con `proxmox-boot-tool`.

## Advertencia

Este laboratorio implica retirar físicamente un disco del arreglo ZFS Mirror. No ejecutes estos pasos en producción sin respaldo, ventana de mantenimiento y claridad absoluta sobre qué disco estás manipulando.
