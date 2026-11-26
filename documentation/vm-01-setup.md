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

> **Actualización:** este resultado corresponde al estado de este hito. Posteriormente se aplicó el hardening de SSH: el acceso se realiza mediante clave y un intento controlado con contraseña fue rechazado. Ver [`ssh-hardening.md`](ssh-hardening.md).

## 4. Decisiones técnicas

| Decisión | Alternativas | Justificación |
|---|---|---|
| Ubuntu Server 24.04.5 LTS ARM64 | Ubuntu 22.04 LTS, otras distribuciones | Versión LTS con soporte prolongado. ARM64 es obligatorio en Apple Silicon. La compatibilidad con la versión de Wazuh elegida está **pendiente de confirmar**. |
| 4 vCPU / 12 GiB | 8 GiB | Margen para el Indexer, el componente de Wazuh que más memoria consume. El anfitrión dispone de 48 GB. |
| Disco de 60 GB (propuesta inicial: 80 GB) | 80 GB | Tamaño ajustado a un laboratorio con poco volumen de datos. Se puede ampliar mediante el espacio libre en LVM o aumentando el disco virtual. |
| Disco sin preasignación | Disco preasignado | En el anfitrión solo ocupa el espacio usado. Puede crecer hasta 60 GB, por lo que conviene vigilar el espacio libre en macOS. |
| LVM con ext4 | Particiones directas, otros sistemas de archivos | LVM permite redimensionar volúmenes. ext4 es el sistema de archivos por defecto del instalador y está ampliamente documentado. |
| Red NAT inicial | Bridged, red solo de anfitrión (host-only) | Permite acceso a Internet para actualizaciones sin exponer la VM en la red local. **Limitación:** no está comprobado que el endpoint Windows físico pueda alcanzar el servidor en este modo. Se revisará antes de incorporar agentes. **Actualización:** el endpoint Windows físico se retiró y la red pasó a `vmnet2`; ver [`architecture.md`](../architecture/architecture.md), sección 4. |
| IP obtenida por DHCP | IP estática, reserva DHCP | Suficiente para la instalación. La IP no se considera permanente: el direccionamiento definitivo está pendiente. |
| Hostname en minúsculas | Mayúsculas, como el nombre de la VM | Convención habitual en sistemas Linux. El nombre de la VM en VMware (`SOC-WAZUH-SERVER`) es independiente del hostname. |
| No ampliar `/` antes de instalar Wazuh (2026-11-02) | Ampliar `/` con el espacio libre de LVM; aumentar el disco virtual | `/` tiene 40 GB disponibles, por debajo de los 50 GB que la [guía de inicio rápido](https://documentation.wazuh.com/current/quickstart.html) indica para 1 a 25 agentes y 90 días de alertas. El laboratorio prevé unos 3 sistemas monitorizados: la [guía del Indexer](https://documentation.wazuh.com/current/installation-guide/wazuh-indexer/index.html) permite estimar unos 6,7 GB de alertas en 90 días, sin contar lo que ocupa Wazuh. El espacio libre de LVM queda como reserva. **Pendiente tras instalar:** definir la retención, vigilar la ocupación y repetir la medición. |
| Instalar Wazuh 4.14.9 con el asistente por etapas (2026-11-02; instalación pendiente) | Asistente todo en uno (`wazuh-install.sh -a`); instalación paso a paso | Permite validar cada componente por separado y aprender el orden real (Indexer, Server, Dashboard), mientras el asistente genera certificados y contraseñas. `wazuh-install-files.tar` contiene secretos y no debe llegar al repositorio. |

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
| Sincronización horaria (2026-11-02, después del cierre del hito) | VM | `timedatectl` | `2026-11-02 14:11:59 UTC`; zona `Etc/UTC`; `System clock synchronized: yes`; `NTP service: active`; `RTC in local TZ: no` |
| Estado de systemd (2026-11-02, después del cierre del hito) | VM | `systemctl is-system-running` y `systemctl --failed --no-pager` | `running`; `0 loaded units listed` |
| Recursos en reposo, sin Wazuh y con ambas VM encendidas (2026-11-02) | VM | `nproc`, `uptime` y `free -h` | 4 vCPU; carga media 0,00 / 0,00 / 0,00; memoria total 11 GiB según `free -h` y 11 GiB disponibles; swap de 4,0 GiB sin usar |
| Almacenamiento real (2026-11-02, repetición) | VM | `df -h /` y `sudo vgs` | `/` de 49 GB, con 6,8 GB usados y 40 GB disponibles (15 %); `ubuntu-vg`: `VSize` `<56.95g`, `VFree` `<6.95g` |
| Arquitectura (2026-11-02) | VM | `dpkg --print-architecture` y `uname -m` | `arm64` y `aarch64` |
| Puertos TCP a la escucha (2026-11-02) | VM | `sudo ss -tlnp` | TCP 22 en todas las interfaces (`sshd`); TCP 53 solo en `127.0.0.53` y `127.0.0.54` (`systemd-resolve`); ningún proceso en los puertos de Wazuh 443, 1514, 1515, 1516, 9200, 9300 y 55000 |

`timedatectl` muestra la hora del sistema, la zona horaria y si la sincronización NTP está activa. `System clock synchronized: yes` es lo que declara el sistema: no mide el desfase real (*offset*) ni identifica el servidor de hora, que no se han comprobado.

`systemctl is-system-running` resume el estado global del sistema, `systemctl --failed` lista las units fallidas y `--no-pager` muestra la salida sin paginador. El resultado refleja solo ese momento: no descarta errores pasados ni demuestra que haya recursos suficientes. **Información técnica general (no verificada en este laboratorio):** un servicio detenido de forma ordenada queda `inactive`, no `failed`, por lo que estos comandos no lo detectan. Para comprobar un servicio concreto hay que consultarlo directamente, por ejemplo con `systemctl is-active <servicio>`.

`nproc` muestra las vCPU disponibles, `uptime` el tiempo encendido y la carga media, `free -h` la memoria y la swap, `df -h /` la ocupación de `/` y `sudo vgs` el espacio libre en el grupo LVM. La medición es una línea base en reposo: no demuestra el comportamiento del sistema con el Indexer en marcha.

`dpkg --print-architecture` y `uname -m` muestran la misma arquitectura con los nombres que usan Debian y el kernel. `ss -tlnp` lista sockets TCP (`-t`) a la escucha (`-l`), en formato numérico (`-n`) y con el proceso asociado (`-p`, requiere `sudo`). Los puertos se compararon con la lista del instalador de Wazuh 4.14. La salida refleja solo ese momento y no evalúa el firewall.

**Compatibilidad de Wazuh 4.14.9 (2026-11-02):** la [guía de inicio rápido](https://documentation.wazuh.com/current/quickstart.html) admite Ubuntu 24.04 y ARM64 para los componentes centrales. La [lista de paquetes](https://documentation.wazuh.com/current/installation-guide/packages-list.html) incluye Indexer, Manager y Dashboard para `arm64`, y el servidor de paquetes respondió HTTP 200 a una consulta de cabeceras de cada uno, sin descargarlos. Es compatibilidad documentada: su funcionamiento en esta VM se comprobará al instalar.

### 5.2 Comprobaciones complementarias recomendadas

Estos comandos **no se han ejecutado** en este hito. Se proponen para completar la validación y generar evidencias adicionales.

| Comprobación | Equipo | Comando recomendado |
|---|---|---|
| Versión del sistema operativo | VM | `cat /etc/os-release` |
| Hostname | VM | `hostnamectl` |
| Dirección IP | VM | `ip -br address` |

Explicación de argumentos:

- `ip -br address`: muestra las direcciones de red en formato breve.

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
- Al cerrar este hito, la autenticación SSH era por contraseña. La migración a claves y el hardening de SSH se realizaron después: ver [`ssh-hardening.md`](ssh-hardening.md).
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
- [x] Sincronización horaria comprobada con `timedatectl` (2026-11-02).
- [x] Estado de systemd comprobado, sin units fallidas (2026-11-02).
- [x] Recursos medidos en reposo y sin Wazuh (2026-11-02).
- [x] Compatibilidad de Wazuh 4.14.9 con Ubuntu 24.04 ARM64 confirmada en la documentación oficial, sin probar su instalación (2026-11-02).

### Pendiente

- [ ] Verificar la firma GPG de `SHA256SUMS` (mejora recomendada).
- [ ] Ejecutar las comprobaciones complementarias de la sección 5.2 y conservar sus salidas como evidencias.
- [ ] Incorporar las evidencias de la sección 7 en `evidence/screenshots/vm-01/`.
- [ ] Reservar la IP fija del servidor en la red `vmnet2`.
- [ ] Instalar Wazuh (Server, Indexer y Dashboard).

> **Actualización (2026-11-04):** el servidor se movió a la red NAT personalizada `vmnet2` y su IP actual es `172.16.50.128`, asignada por DHCP. Las referencias a `172.16.203.128` de este documento corresponden a la red inicial. El endpoint Windows físico, que figuraba aquí como pendiente, se retiró el 2026-11-26 y se sustituirá por una VM. Ver [`architecture.md`](../architecture/architecture.md), sección 4.

> **Actualización:** el endpoint Linux (VM 2, SOC-LINUX-ENDPOINT), que figuraba aquí como pendiente, ya está creado, instalado y validado. La conectividad ICMP entre ambas VM está comprobada. Ver [`vm-02-setup.md`](vm-02-setup.md).

> **Actualización:** el hardening de SSH, que figuraba aquí como pendiente, ya está aplicado: acceso mediante clave comprobado y contraseña rechazada en una prueba controlada. `PermitRootLogin no` está confirmado en la configuración, sin prueba directa de inicio de sesión como `root`. Ver [`ssh-hardening.md`](ssh-hardening.md).

## 10. Conclusiones y limitaciones

La VM SOC-WAZUH-SERVER dispone de un sistema base Ubuntu Server instalado y en funcionamiento, actualizado, con almacenamiento ampliable mediante LVM y administración remota por SSH desde el anfitrión. El acceso SSH se endureció en un hito posterior ([`ssh-hardening.md`](ssh-hardening.md)).

Limitaciones actuales:

- La red NAT y la IP por DHCP son provisionales.
- Las evidencias todavía no están incorporadas al repositorio.
- **Wazuh todavía no está instalado**: el servidor aún no cumple su función en el laboratorio.
