# VM 01 — SOC-WAZUH-SERVER: preparación e instalación de Ubuntu Server

## 1. Objetivo y contexto

Preparar la máquina virtual que alojará los componentes centrales de Wazuh (Server, Indexer y Dashboard) en el MacBook Pro, según la arquitectura descrita en [`architecture/architecture.md`](../architecture/architecture.md).

**Fase del proyecto:** Fase 2 — Preparación de infraestructura (en proceso).

Este documento cubre:

- La verificación de la imagen ISO.
- La creación y configuración de la máquina virtual.
- La instalación y validación inicial de Ubuntu Server.
- Las incidencias encontradas durante la instalación.

La instalación de Wazuh **no forma parte de este hito** y se documentará por separado.

**Estado:** Ubuntu Server está instalado, actualizado, en funcionamiento y accesible por SSH desde el anfitrión. **Wazuh todavía no está instalado.**

## 2. Infraestructura y versiones

| Elemento | Valor |
|---|---|
| Equipo anfitrión | MacBook Pro, Apple M5 Pro, ARM64, macOS |
| Hipervisor | VMware Fusion Professional 26H1u1 (build 2689522) |
| Nombre de la VM | SOC-WAZUH-SERVER |
| Sistema operativo | Ubuntu Server 24.04.5 LTS ARM64 |
| Imagen ISO | `ubuntu-24.04.5-live-server-arm64.iso` |
| vCPU | 4 |
| Memoria | 12 GiB |
| Disco virtual | 60 GB, sin preasignación (crecimiento dinámico) |
| Hostname | `soc-wazuh-server` |
| Usuario administrativo | `socadmin` |
| Red | NAT mediante VMware Fusion |
| Cliente SSH en el anfitrión | Ghostty (macOS) |

## 3. Procedimiento ejecutado

### 3.1 Verificación de la imagen ISO

**Equipo:** MacBook Pro (macOS).

1. Se descargó `ubuntu-24.04.5-live-server-arm64.iso` desde el sitio oficial de Ubuntu.
2. Se calculó la suma SHA-256 de la imagen:

   ```bash
   shasum -a 256 ~/Downloads/ubuntu-24.04.5-live-server-arm64.iso
   ```

   - `shasum`: calcula sumas de verificación.
   - `-a 256`: selecciona el algoritmo SHA-256.

3. Se comparó el resultado con la línea correspondiente del archivo oficial `SHA256SUMS` publicado por Ubuntu.

**Resultado obtenido:** la suma calculada coincide con la oficial.

**Firma GPG de `SHA256SUMS`: no verificada.**

> La comparación SHA-256 confirma la **integridad** de la descarga respecto a la suma publicada. Verificar la firma GPG de `SHA256SUMS` confirmaría además su **autenticidad**, es decir, que el archivo de sumas fue publicado por Ubuntu y no ha sido sustituido. Queda como mejora pendiente.

### 3.2 Creación de la máquina virtual

**Equipo:** MacBook Pro (VMware Fusion).

- Creación de una VM nueva a partir de la ISO verificada.
- Asignación de 4 vCPU y 12 GiB de memoria.
- Disco virtual de 60 GB sin preasignación.
- Adaptador de red en modo NAT.

### 3.3 Instalación de Ubuntu Server

**Equipo:** VM SOC-WAZUH-SERVER (instalador de Ubuntu Server).

- Almacenamiento: esquema LVM con sistema de archivos ext4 en el volumen raíz.
- Hostname: `soc-wazuh-server`.
- Usuario administrativo: `socadmin`.
- Instalación del servidor OpenSSH.

Distribución de almacenamiento resultante:

| Elemento | Tamaño aproximado | Observaciones |
|---|---|---|
| Partición EFI | 1 GB | Arranque UEFI |
| Partición `/boot` | 2 GB | Kernel e initramfs |
| Volumen lógico `/` | 50 GB | ext4 sobre LVM |
| Espacio libre en el grupo de volúmenes LVM | 6,9 GB | Sin asignar; disponible para ampliar volúmenes |

> El espacio libre del grupo de volúmenes permite ampliar el volumen raíz más adelante sin modificar el disco virtual (por ejemplo, con `lvextend` y `resize2fs`). No se ha realizado ninguna ampliación.

Verificación tras la instalación (**equipo:** VM SOC-WAZUH-SERVER):

```bash
lsblk -o NAME,SIZE,FSTYPE,MOUNTPOINTS
df -h /
```

- `lsblk -o NAME,SIZE,FSTYPE,MOUNTPOINTS`: lista los dispositivos de bloque mostrando solo las columnas indicadas: nombre, tamaño, sistema de archivos y puntos de montaje.
- `df -h /`: muestra el espacio total, usado y disponible del sistema de archivos montado en `/`, en unidades legibles (`-h`).

### 3.4 Comprobación de red

**Equipo:** VM SOC-WAZUH-SERVER.

```bash
ping -c 4 ubuntu.com
```

- `-c 4`: envía exactamente cuatro paquetes ICMP y termina.
- Usar un nombre de dominio en lugar de una IP comprueba a la vez la resolución DNS y la salida a Internet.

**Resultado obtenido:** 4 respuestas, 0 % de pérdida.

### 3.5 Comprobación del servicio SSH

**Equipo:** VM SOC-WAZUH-SERVER.

```bash
sudo systemctl status ssh
systemctl status ssh.socket
```

- `systemctl status ssh`: muestra el estado de la unidad `ssh.service` (el demonio `sshd`).
- `systemctl status ssh.socket`: muestra el estado del socket que escucha las conexiones entrantes.

**Resultado obtenido:** `ssh.socket` en estado `active (listening)` en TCP 22.

> **Activación por socket en Ubuntu 24.04:** `ssh.socket` escucha en el puerto 22 y arranca `ssh.service` cuando llega una conexión. Por eso el indicador relevante de que SSH acepta conexiones es el estado de `ssh.socket`.

### 3.6 Actualización del sistema

**Equipo:** VM SOC-WAZUH-SERVER.

```bash
sudo apt update
apt list --upgradable
sudo apt upgrade
```

- `apt update`: actualiza la lista de paquetes disponibles desde los repositorios.
- `apt list --upgradable`: muestra los paquetes instalados que tienen una versión más reciente disponible.
- `apt upgrade`: instala las versiones más recientes de los paquetes ya instalados.

Después se comprobó si el sistema solicitaba reinicio mediante la existencia de `/run/reboot-required`. Ubuntu crea este archivo cuando una actualización (por ejemplo, del kernel) requiere reiniciar.

**Resultado obtenido:** `apt list --upgradable` sin actualizaciones pendientes tras la actualización, y el sistema no solicita reinicio.

### 3.7 Acceso SSH desde el anfitrión

**Equipo:** MacBook Pro (Ghostty, macOS).

```bash
ssh socadmin@172.16.203.128
```

- `socadmin`: usuario administrativo de la VM.
- `172.16.203.128`: IP privada asignada **temporalmente por DHCP** en la red NAT de VMware Fusion. Puede cambiar y no debe usarse como dirección permanente.

**Resultado obtenido:** conexión remota correcta, con autenticación por contraseña.

## 4. Decisiones técnicas

| Decisión | Alternativas | Justificación |
|---|---|---|
| Ubuntu Server 24.04.5 LTS ARM64 | Ubuntu 22.04 LTS, otras distribuciones | Versión LTS con soporte prolongado. ARM64 es obligatorio en Apple Silicon. La compatibilidad con la versión de Wazuh elegida está **pendiente de confirmar**. |
| 4 vCPU / 12 GiB | 8 GiB | Margen para el Indexer, el componente de Wazuh que más memoria consume. El anfitrión dispone de 48 GB. |
| Disco de 60 GB (propuesta inicial: 80 GB) | 80 GB | Tamaño ajustado a un laboratorio con poco volumen de datos. Se puede ampliar mediante el espacio libre en LVM o aumentando el disco virtual. |
| Disco sin preasignación | Disco preasignado | En el anfitrión solo ocupa el espacio usado. Puede crecer hasta 60 GB, por lo que conviene vigilar el espacio libre en macOS. |
| LVM con ext4 | Particiones directas, otros sistemas de archivos | LVM permite redimensionar volúmenes. ext4 es el sistema de archivos por defecto del instalador y está ampliamente documentado. |
| Red NAT inicial | Bridged, red solo de anfitrión (host-only) | Permite acceso a Internet para actualizaciones sin exponer la VM en la red local. **Limitación:** no está comprobado que el endpoint Windows físico pueda alcanzar el servidor en este modo. Se revisará antes de incorporar agentes. |
| IP obtenida por DHCP | IP estática, reserva DHCP | Suficiente para la instalación. La IP no se considera permanente: el direccionamiento definitivo está pendiente. |
| Hostname en minúsculas | Mayúsculas, como el nombre de la VM | Convención habitual en sistemas Linux. El nombre de la VM en VMware (`SOC-WAZUH-SERVER`) es independiente del hostname. |

## 5. Verificaciones y resultados

### 5.1 Verificaciones realizadas

Comandos ejecutados y resultados confirmados durante la implementación. Cuando el método exacto no quedó registrado, se indica en la columna «Comando o método».

| Comprobación | Equipo | Comando o método | Resultado obtenido |
|---|---|---|---|
| Integridad de la ISO | MacBook Pro | `shasum -a 256 ~/Downloads/ubuntu-24.04.5-live-server-arm64.iso` y comparación con `SHA256SUMS` | Coincide con la suma oficial |
| Distribución del almacenamiento | VM | `lsblk -o NAME,SIZE,FSTYPE,MOUNTPOINTS` y `df -h /` | EFI ≈ 1 GB, `/boot` 2 GB, `/` 50 GB en ext4 sobre LVM |
| Espacio libre en el grupo LVM | VM | Configuración del instalador (comando de verificación no registrado) | ≈ 6,9 GB sin asignar |
| Dirección IP | VM | Método no registrado | `172.16.203.128/24`, asignada temporalmente por DHCP en la red NAT de VMware |
| DNS y salida a Internet | VM | `ping -c 4 ubuntu.com` | 4 respuestas, 0 % de pérdida |
| Servicio SSH | VM | `sudo systemctl status ssh` y `systemctl status ssh.socket` | `ssh.socket` `active (listening)` en TCP 22 |
| Actualización de paquetes | VM | `sudo apt update`, `apt list --upgradable` y `sudo apt upgrade` | Sin actualizaciones pendientes tras la actualización |
| Reinicio pendiente | VM | Comprobación de `/run/reboot-required` | El sistema no solicita reinicio |
| Acceso SSH desde el anfitrión | MacBook Pro | `ssh socadmin@172.16.203.128` (Ghostty) | Conexión remota correcta con contraseña |

### 5.2 Comprobaciones complementarias recomendadas

Estos comandos **no se han ejecutado** en este hito. Se proponen para completar la validación y generar evidencias adicionales.

| Comprobación | Equipo | Comando recomendado |
|---|---|---|
| Versión del sistema operativo | VM | `cat /etc/os-release` |
| Hostname | VM | `hostnamectl` |
| Espacio libre en el grupo LVM | VM | `sudo vgs` |
| Dirección IP | VM | `ip -br address` |
| Puertos a la escucha | VM | `sudo ss -tlnp` |
| Sincronización horaria | VM | `timedatectl` |

Explicación de argumentos:

- `vgs`: resume los grupos de volúmenes LVM, incluido el espacio libre (`VFree`).
- `ip -br address`: muestra las direcciones de red en formato breve.
- `ss -tlnp`: lista sockets TCP (`-t`) a la escucha (`-l`), en formato numérico (`-n`) y con el proceso asociado (`-p`, requiere `sudo`).
- `timedatectl`: muestra la hora del sistema, la zona horaria y si la sincronización NTP está activa.

## 6. Incidencias y resolución

### INC-01: «Failed unmounting cdrom.mount - /cdrom» al reiniciar tras la instalación

| Campo | Detalle |
|---|---|
| Fase | Reinicio final del instalador de Ubuntu Server |
| Síntoma | Durante el reinicio apareció el mensaje `Failed unmounting cdrom.mount - /cdrom`. |
| Acciones realizadas | 1. Se confirmó en VMware Fusion que la ISO estaba desconectada de la VM. 2. Se reinició la máquina virtual desde VMware Fusion. |
| Resultado | Ubuntu arrancó correctamente y permitió iniciar sesión. |
| Causa raíz | **No determinada.** No se ha realizado un análisis que permita atribuirla. |
| Impacto observado | Ninguno tras el reinicio: el sistema arranca y funciona con normalidad. |
| Estado | Resuelta |

Posible investigación futura (no realizada): revisar los registros del arranque en el que apareció el error con `journalctl --list-boots` y `journalctl -b <id>`, si se conservan.

## 7. Evidencias

Las capturas se guardarán en `evidence/screenshots/vm-01/` (en la raíz del repositorio) una vez revisadas y anonimizadas.

**Estado: pendientes.** Ninguna evidencia está incorporada al repositorio todavía.

| # | Evidencia necesaria | Qué demuestra | Revisión antes de publicar | Estado |
|---|---|---|---|---|
| 1 | Salida de `shasum -a 256` junto a la línea correspondiente de `SHA256SUMS` | Integridad de la ISO | Ocultar el usuario de macOS en el prompt | Pendiente |
| 2 | VMware Fusion → Acerca de | Versión y build del hipervisor | — | Pendiente |
| 3 | Ajustes de la VM: procesadores, memoria y disco | 4 vCPU, 12 GiB y 60 GB | — | Pendiente |
| 4 | Ajustes de la VM: adaptador de red | Modo NAT | Ocultar la dirección MAC | Pendiente |
| 5 | Salida de `lsblk -o NAME,SIZE,FSTYPE,MOUNTPOINTS` y `df -h /` (y de `sudo vgs`, si se ejecuta) | Distribución de almacenamiento y espacio libre en LVM | — | Pendiente |
| 6 | Salida de `hostnamectl` y `cat /etc/os-release` (comandos de la sección 5.2, todavía no ejecutados) | Hostname y versión del sistema | Ocultar `Machine ID` y `Boot ID` | Pendiente |
| 7 | Salida de `ping -c 4 ubuntu.com` (y de `ip -br address`, si se ejecuta) | Salida a Internet, DNS e IP por DHCP | Ocultar la dirección MAC si aparece | Pendiente |
| 8 | Salida de `sudo systemctl status ssh` y `systemctl status ssh.socket` | SSH a la escucha en TCP 22 | — | Pendiente |
| 9 | Sesión `ssh socadmin@172.16.203.128` iniciada desde Ghostty | Acceso remoto desde el anfitrión | Ocultar el usuario de macOS y la ruta en el prompt local | Pendiente |
| 10 | Salida de `apt list --upgradable` tras `sudo apt upgrade` y comprobación de `/run/reboot-required` | Sistema actualizado y sin reinicio pendiente | — | Pendiente |
| 11 | Mensaje `Failed unmounting cdrom.mount - /cdrom`, si se conserva | Síntoma de INC-01 | — | Pendiente |

Recomendaciones:

- Nombres de archivo descriptivos, por ejemplo `vm01-05-almacenamiento.png`.
- Eliminar los metadatos de las imágenes antes de publicarlas.
- No incluir contraseñas, claves ni el contenido de `~/.ssh/`.

## 8. Seguridad

- La VM no está expuesta a Internet: se encuentra tras el NAT de VMware Fusion.
- La autenticación SSH es por contraseña. La migración a claves y el hardening de SSH están pendientes.
- No se documentan contraseñas ni claves en este repositorio.

## 9. Estado del hito

### Completado y verificado por el estudiante

- [x] ISO verificada mediante SHA-256 contra la suma publicada por Ubuntu.
- [x] VM creada con 4 vCPU, 12 GiB de memoria y disco de 60 GB sin preasignación.
- [x] Ubuntu Server 24.04.5 LTS ARM64 instalado con LVM y ext4, en funcionamiento.
- [x] Hostname y usuario administrativo configurados.
- [x] Conectividad DNS e Internet comprobada desde la VM.
- [x] OpenSSH instalado y `ssh.socket` a la escucha en TCP 22.
- [x] Acceso SSH desde el anfitrión comprobado.
- [x] Sistema actualizado, sin actualizaciones ni reinicios pendientes.
- [x] Incidencia INC-01 resuelta.

### Pendiente

- [ ] Verificar la firma GPG de `SHA256SUMS` (mejora recomendada).
- [ ] Ejecutar las comprobaciones complementarias de la sección 5.2 y conservar sus salidas como evidencias.
- [ ] Incorporar las evidencias de la sección 7 en `evidence/screenshots/vm-01/`.
- [ ] Verificar la sincronización horaria del sistema, necesaria para correlacionar eventos.
- [ ] Definir la configuración de red definitiva y el direccionamiento IP del laboratorio.
- [ ] Comprobar la conectividad entre el servidor y el endpoint Windows físico.
- [ ] Hardening de SSH: autenticación por claves y desactivación del acceso por contraseña.
- [ ] Confirmar la compatibilidad de la versión de Wazuh elegida con Ubuntu 24.04 ARM64.
- [ ] Instalar Wazuh (Server, Indexer y Dashboard).
- [ ] Crear el endpoint Linux (VM 2).

## 10. Conclusiones y limitaciones

La VM SOC-WAZUH-SERVER dispone de un sistema base Ubuntu Server instalado y en funcionamiento, actualizado, con almacenamiento ampliable mediante LVM y administración remota por SSH desde el anfitrión.

Limitaciones actuales:

- La red NAT y la IP por DHCP son provisionales.
- El acceso SSH sigue usando contraseña.
- Las evidencias todavía no están incorporadas al repositorio.
- **Wazuh todavía no está instalado**: el servidor aún no cumple su función en el laboratorio.
