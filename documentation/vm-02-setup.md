# VM 02 — SOC-LINUX-ENDPOINT: preparación e instalación de Ubuntu Desktop

## 1. Objetivo y contexto

Preparar la máquina virtual que funcionará como **endpoint Linux** del laboratorio, según la arquitectura descrita en [`architecture/architecture.md`](../architecture/architecture.md).

Esta VM tiene una doble función:

- **Entorno de aprendizaje diario de Linux**, con escritorio gráfico, para practicar administración del sistema.
- **Futuro endpoint monitorizado por Wazuh**, en el que se instalará Wazuh Agent y se ejecutarán escenarios controlados (autenticación, integridad de archivos, cambios administrativos).

**Fase del proyecto:** Fase 2 — Preparación de infraestructura (en proceso).

Este documento cubre:

- La verificación de la imagen ISO.
- La creación y configuración de la máquina virtual.
- La instalación y validación inicial de Ubuntu Desktop.
- La integración con VMware Fusion mediante open-vm-tools.
- La comprobación de conectividad con la VM SOC-WAZUH-SERVER.

La instalación de Wazuh Agent **no forma parte de este hito** y se documentará por separado.

**Estado:** Ubuntu Desktop está instalado, actualizado y en funcionamiento, con conectividad a Internet y con la VM SOC-WAZUH-SERVER. **Wazuh Agent todavía no está instalado.**

**Convenciones de este documento:**

- **Resultado obtenido:** dato comprobado en este laboratorio.
- **No registrado:** resultado comprobado cuyo comando o método exacto no quedó anotado.
- **Información técnica general:** comportamiento documentado del software, incluido para entender los resultados, pero **no verificado** en este laboratorio.

## 2. Infraestructura y versiones

| Elemento | Valor |
|---|---|
| Equipo anfitrión | MacBook Pro, Apple M5 Pro, ARM64, macOS |
| Hipervisor | VMware Fusion (mismo anfitrión que la VM 1) |
| Nombre de la VM | SOC-LINUX-ENDPOINT |
| Sistema operativo | Ubuntu Desktop 24.04.5 LTS ARM64 |
| Imagen ISO | `ubuntu-24.04.5-desktop-arm64.iso` |
| vCPU | 4 |
| Memoria | 6 GiB asignados (5,8 GiB reconocidos por Linux) |
| Disco virtual | 50 GB, NVMe virtual (preasignación no registrada) |
| Hostname | `soc-linux-endpoint` |
| Usuario | `labuser` |
| Entorno gráfico | GNOME, sesión Wayland |
| Red | NAT mediante VMware Fusion, IP por DHCP |
| Herramientas de integración | `open-vm-tools` y `open-vm-tools-desktop` 13.0.10 |

## 3. Procedimiento ejecutado

### 3.1 Verificación de la imagen ISO

**Equipo:** MacBook Pro (macOS).

1. Se descargó la imagen `ubuntu-24.04.5-desktop-arm64.iso`.
2. Se calculó su suma SHA-256:

   ```bash
   shasum -a 256 ~/Downloads/ubuntu-24.04.5-desktop-arm64.iso
   ```

   - `shasum`: calcula sumas de verificación.
   - `-a 256`: selecciona el algoritmo SHA-256.

3. Se comparó el resultado con la línea correspondiente del archivo oficial `SHA256SUMS` publicado por Ubuntu.

**Resultado obtenido:** la suma calculada coincide con la oficial (comprobado por el estudiante).

**Firma GPG de `SHA256SUMS`: no verificada.** Como en la VM 1, la suma SHA-256 confirma la **integridad** de la descarga, pero no la **autenticidad** del archivo de sumas. Queda como mejora pendiente.

### 3.2 Creación de la máquina virtual

**Equipo:** MacBook Pro (VMware Fusion).

- Creación de una VM nueva a partir de la ISO verificada.
- Asignación de 4 vCPU y 6 GiB de memoria.
- Disco virtual NVMe de 50 GB.
- Adaptador de red en modo NAT.

> Los recursos son superiores a la propuesta inicial de la arquitectura (2 vCPU, 3 GB RAM, 30 GB). El motivo es el cambio a una edición de escritorio: GNOME y las aplicaciones gráficas necesitan más memoria y disco que un sistema sin entorno gráfico. Ver la sección 4.

### 3.3 Instalación de Ubuntu Desktop

**Equipo:** VM SOC-LINUX-ENDPOINT (instalador de Ubuntu Desktop).

- Hostname: `soc-linux-endpoint`.
- Usuario: `labuser`.
- Almacenamiento: particionado sin LVM, con ext4 en la partición raíz.

Distribución de almacenamiento resultante:

| Elemento | Tamaño | Sistema de archivos | Observaciones |
|---|---|---|---|
| Partición EFI | 1 GB | vfat | Arranque UEFI |
| Partición raíz `/` | 48,9 GB | ext4 | Sin LVM |

Ocupación observada del sistema de archivos raíz:

| Tamaño | Usado | Disponible |
|---|---|---|
| 48 GB | 11 GB | 36 GB |

> **Por qué no coinciden exactamente las cifras.** El tamaño que muestra `df` para `/` (48 GB) es algo menor que el de la partición (48,9 GB) porque ext4 dedica parte del espacio a sus propios metadatos. Además, «Usado + Disponible» no suma exactamente el tamaño total: ext4 reserva por defecto una parte de los bloques para el usuario `root`, y `df -h` redondea los valores. Ninguna de estas diferencias indica un problema.

> **Sin LVM:** a diferencia de la VM 1, esta VM no usa LVM. Si en el futuro hiciera falta más espacio, habría que ampliar el disco virtual en VMware Fusion y después la partición y el sistema de archivos (por ejemplo, con `growpart` y `resize2fs`). No se ha realizado ninguna ampliación.

Comandos de referencia para verificar el almacenamiento (**equipo:** VM SOC-LINUX-ENDPOINT):

```bash
lsblk -o NAME,SIZE,FSTYPE,MOUNTPOINTS
df -h /
```

- `lsblk -o NAME,SIZE,FSTYPE,MOUNTPOINTS`: lista los dispositivos de bloque con nombre, tamaño, sistema de archivos y puntos de montaje. En esta VM el disco aparece como `nvme0n1`, con sus particiones `nvme0n1p1`, `nvme0n1p2`, etc.
- `df -h /`: muestra el espacio total, usado y disponible del sistema de archivos montado en `/`, en unidades legibles (`-h`).

> El método exacto con el que se obtuvieron los datos de almacenamiento no quedó registrado; los comandos anteriores son los recomendados para generar la evidencia.

### 3.4 Memoria reconocida por el sistema

**Equipo:** VM SOC-LINUX-ENDPOINT.

**Resultado obtenido:** Linux reconoce 5,8 GiB de los 6 GiB asignados.

> **Información técnica general (no verificada en este laboratorio):** el total de memoria que informa Linux (por ejemplo, con `free -h`) suele ser algo inferior al asignado, porque excluye la memoria que reservan el firmware UEFI y el kernel durante el arranque. Por eso una diferencia de este tamaño no indica, por sí sola, una asignación incorrecta en VMware Fusion. No se ha analizado el desglose concreto de la memoria reservada en esta VM.

### 3.5 Actualización del sistema

**Equipo:** VM SOC-LINUX-ENDPOINT.

Se aplicaron las actualizaciones disponibles y se reinició el sistema.

**Resultado obtenido:** actualizaciones aplicadas y reinicio realizado.

**No registrado:** si se usó la terminal (`apt`) o el actualizador gráfico, y la comprobación posterior de que no quedaban paquetes pendientes. Comandos de referencia, equivalentes a los de la VM 1:

```bash
sudo apt update
apt list --upgradable
sudo apt upgrade
```

- `apt update`: actualiza la lista de paquetes disponibles desde los repositorios.
- `apt list --upgradable`: muestra los paquetes instalados con una versión más reciente disponible.
- `apt upgrade`: instala esas versiones más recientes.

### 3.6 Integración con VMware Fusion: open-vm-tools

**Equipo:** VM SOC-LINUX-ENDPOINT.

Paquetes instalados (versión 13.0.10):

- `open-vm-tools`: servicios básicos de integración entre la VM y el hipervisor. Incluye el demonio `vmtoolsd`, que permite a VMware Fusion apagar o reiniciar la VM de forma ordenada, informar del estado del sistema invitado y sincronizar la hora con el anfitrión cuando se habilita.
- `open-vm-tools-desktop`: componentes para el entorno gráfico: portapapeles compartido, ajuste automático de la resolución de pantalla y arrastrar y soltar.

> `open-vm-tools` es la implementación de código abierto de VMware Tools que mantienen VMware y las distribuciones Linux. En Ubuntu se instala desde los repositorios oficiales y se actualiza con el resto del sistema, por lo que no es necesario instalar VMware Tools desde el menú de VMware Fusion.

**Resultados obtenidos:**

- Paquetes `open-vm-tools` y `open-vm-tools-desktop` instalados, versión 13.0.10.
- Servicio `open-vm-tools` activo.
- Portapapeles compartido entre macOS y Ubuntu probado y funcionando en la sesión Wayland.

> **Wayland:** GNOME en Ubuntu 24.04 usa Wayland por defecto. En Wayland las aplicaciones no pueden leer libremente el portapapeles de otras, como ocurría en X11, por lo que la compatibilidad del portapapeles compartido depende de la versión de las herramientas. Por eso es relevante haberlo comprobado en una sesión Wayland real y no darlo por supuesto.

**No registrado:** el método de instalación y los comandos de verificación. Comandos de referencia:

```bash
systemctl status open-vm-tools
vmware-toolbox-cmd -v
```

- `systemctl status open-vm-tools`: muestra el estado del servicio `open-vm-tools.service`, que ejecuta `vmtoolsd`.
- `vmware-toolbox-cmd -v`: muestra la versión de las herramientas instaladas.

### 3.7 Comprobación de red

**Equipo:** VM SOC-LINUX-ENDPOINT.

La VM usa la red NAT de VMware Fusion, igual que la VM 1, y obtiene su IP por DHCP.

| Comprobación | Resultado obtenido |
|---|---|
| IP de la VM | `172.16.203.129`, asignada temporalmente por DHCP. **No es permanente.** |
| Conectividad a Internet | Comprobada |
| Resolución DNS | Comprobada |

**No registrado:** los comandos usados para obtener la IP y comprobar Internet y DNS. Comandos de referencia:

```bash
ip -br address
ping -c 4 ubuntu.com
```

- `ip -br address`: muestra las direcciones de red de cada interfaz en formato breve.
- `ping -c 4 ubuntu.com`: envía cuatro paquetes ICMP a un nombre de dominio. Comprueba a la vez la resolución DNS y la salida a Internet.

### 3.8 Conectividad entre el endpoint y el servidor

**Equipos:** VM SOC-LINUX-ENDPOINT y VM SOC-WAZUH-SERVER.

Es la primera comprobación de comunicación entre dos máquinas del laboratorio. Ambas VM se encuentran en la misma red NAT de VMware Fusion.

| Origen | Destino | Pérdida | Latencia media |
|---|---|---|---|
| SOC-LINUX-ENDPOINT | SOC-WAZUH-SERVER (`172.16.203.128`) | 0 % | 0,627 ms |
| SOC-WAZUH-SERVER | SOC-LINUX-ENDPOINT (`172.16.203.129`) | 0 % | 0,506 ms |

Comando de referencia (**no registrado** el número exacto de paquetes enviados):

```bash
# En SOC-LINUX-ENDPOINT
ping -c 4 172.16.203.128

# En SOC-WAZUH-SERVER
ping -c 4 172.16.203.129
```

> **Qué demuestra y qué no.** Un `ping` correcto en ambos sentidos confirma que hay conectividad IP entre las dos VM (capa 3, protocolo ICMP). **No demuestra** que el endpoint pueda comunicarse con los puertos de Wazuh, que todavía no existen porque Wazuh no está instalado. La latencia inferior a 1 ms es la esperable entre dos VM del mismo anfitrión.

> **Información técnica general (no verificada en este laboratorio):** según la documentación de Wazuh, el agente se comunica con el servidor por TCP. Los puertos por defecto son 1514 para el envío de eventos y 1515 para el registro (*enrollment*). Habrá que confirmar los puertos de la versión elegida y comprobar su accesibilidad cuando Wazuh esté instalado.

> Las IP son asignadas por DHCP y pueden cambiar. Antes de configurar el agente será necesario fijar el direccionamiento (IP estática o reserva DHCP) o usar un nombre resoluble para el servidor.

## 4. Decisiones técnicas

| Decisión | Alternativas | Justificación |
|---|---|---|
| Ubuntu Desktop 24.04.5 LTS ARM64 | Ubuntu Server (como la VM 1), otras distribuciones | Permite usar la VM como entorno diario de aprendizaje de Linux con escritorio, además de como endpoint. Misma versión LTS que el servidor, lo que simplifica la administración. ARM64 es obligatorio en Apple Silicon. |
| 4 vCPU / 6 GiB / 50 GB (propuesta inicial: 2 vCPU / 3 GB / 30 GB) | Mantener la propuesta inicial | GNOME y las aplicaciones gráficas requieren más recursos que un sistema sin escritorio. Con la VM 1 (12 GiB), el total asignado es de 18 GiB sobre los 48 GB del anfitrión. El consumo real **no se ha medido** todavía. |
| Disco NVMe virtual | SATA | Controlador ofrecido por VMware Fusion para esta VM; buen rendimiento y soporte nativo en el kernel Linux. |
| ext4 sin LVM | LVM con ext4, como en la VM 1 | Particionado más simple, adecuado para un endpoint de pruebas. **Limitación:** ampliar el disco requiere redimensionar la partición en lugar de usar `lvextend`. |
| Sesión Wayland | Sesión X11 (Xorg) | Opción por defecto de Ubuntu 24.04. El portapapeles compartido se ha comprobado y funciona con Wayland, por lo que no ha sido necesario cambiar a X11. |
| Portapapeles compartido activado | Desactivado de forma permanente | Facilita el aprendizaje diario. Se recomienda desactivarlo temporalmente en escenarios controlados que requieran mayor aislamiento (ver sección 8). |
| open-vm-tools desde los repositorios de Ubuntu | VMware Tools desde el menú de VMware Fusion | Opción recomendada para distribuciones Linux actuales. Se actualiza con `apt` junto al resto del sistema. |
| Red NAT con DHCP | Bridged, red solo de anfitrión (host-only), IP estática | Coherente con la VM 1: ambas VM comparten la red NAT y se comunican entre sí, con acceso a Internet y sin exposición en la red local. El direccionamiento definitivo está pendiente. |
| Hostname en minúsculas | Mayúsculas, como el nombre de la VM | Misma convención que la VM 1. El nombre de la VM en VMware (`SOC-LINUX-ENDPOINT`) es independiente del hostname. |

## 5. Verificaciones y resultados

### 5.1 Verificaciones realizadas

Resultados confirmados durante la implementación. Cuando el comando exacto no quedó registrado, se indica en la columna «Comando o método».

| Comprobación | Equipo | Comando o método | Resultado obtenido |
|---|---|---|---|
| Integridad de la ISO | MacBook Pro | `shasum -a 256 ~/Downloads/ubuntu-24.04.5-desktop-arm64.iso` y comparación con `SHA256SUMS` | Coincide con la suma oficial |
| Recursos de la VM | MacBook Pro | Ajustes de VMware Fusion | 4 vCPU, 6 GiB, disco NVMe de 50 GB, red NAT |
| Memoria reconocida | VM | Método no registrado | 5,8 GiB |
| Particionado | VM | Método no registrado | EFI 1 GB vfat; `/` 48,9 GB ext4, sin LVM |
| Ocupación de `/` | VM | Método no registrado | 48 GB totales, 11 GB usados, 36 GB disponibles |
| Hostname y usuario | VM | Configuración del instalador | `soc-linux-endpoint`, `labuser` |
| Entorno gráfico | VM | Método no registrado | GNOME, sesión Wayland |
| Dirección IP | VM | Método no registrado | `172.16.203.129`, temporal por DHCP en la red NAT |
| Internet y DNS | VM | Método no registrado | Conectividad comprobada |
| Endpoint → servidor | VM 2 | `ping` a `172.16.203.128` | 0 % de pérdida, 0,627 ms de media |
| Servidor → endpoint | VM 1 | `ping` a `172.16.203.129` | 0 % de pérdida, 0,506 ms de media |
| Actualización del sistema | VM | Método no registrado | Actualizaciones aplicadas y reinicio realizado |
| open-vm-tools | VM | Método no registrado | `open-vm-tools` y `open-vm-tools-desktop` 13.0.10 instalados; servicio activo |
| Portapapeles compartido | MacBook Pro y VM | Prueba manual de copiar y pegar en ambos sentidos | Funciona en la sesión Wayland |

### 5.2 Comprobaciones complementarias recomendadas

Estos comandos **no se han ejecutado** (o su salida no quedó registrada) en este hito. Se proponen para completar la validación y generar evidencias.

| Comprobación | Equipo | Comando recomendado |
|---|---|---|
| Versión del sistema operativo | VM | `cat /etc/os-release` |
| Hostname y arquitectura | VM | `hostnamectl` |
| Tipo de sesión gráfica | VM | `echo $XDG_SESSION_TYPE` |
| Memoria | VM | `free -h` |
| Almacenamiento | VM | `lsblk -o NAME,SIZE,FSTYPE,MOUNTPOINTS` y `df -h /` |
| Dirección IP | VM | `ip -br address` |
| Paquetes pendientes | VM | `apt list --upgradable` |
| Reinicio pendiente | VM | `ls /run/reboot-required` |
| Servicio open-vm-tools | VM | `systemctl status open-vm-tools` |
| Versión de open-vm-tools | VM | `vmware-toolbox-cmd -v` |
| Sincronización horaria | VM | `timedatectl` |
| Servidor SSH | VM | `systemctl status ssh` |
| Puertos a la escucha | VM | `sudo ss -tlnp` |

Explicación de argumentos:

- `echo $XDG_SESSION_TYPE`: muestra el tipo de sesión gráfica actual (`wayland` o `x11`).
- `free -h`: muestra la memoria total, usada y disponible en unidades legibles.
- `ls /run/reboot-required`: si el archivo existe, alguna actualización requiere reiniciar.
- `timedatectl`: muestra la hora, la zona horaria y si la sincronización NTP está activa. Es importante en un endpoint: si la hora no es correcta, los eventos enviados a Wazuh no se podrán correlacionar bien con los del servidor.
- `systemctl status ssh`: muestra el estado del servidor SSH. **Información técnica general (no verificada en este laboratorio):** a diferencia de Ubuntu Server, el instalador de Ubuntu Desktop no incluye el servidor OpenSSH (`openssh-server`) por defecto. **En esta VM su estado no se ha comprobado.** La arquitectura lo prevé para las prácticas de autenticación (LAB 01).
- `ss -tlnp`: lista sockets TCP (`-t`) a la escucha (`-l`), en formato numérico (`-n`) y con el proceso asociado (`-p`, requiere `sudo`).

## 6. Incidencias y resolución

No se han registrado incidencias durante este hito.

## 7. Evidencias

Las capturas se guardarán en `evidence/screenshots/vm-02/` (en la raíz del repositorio) una vez revisadas y anonimizadas.

**Estado: pendientes.** Ninguna evidencia está incorporada al repositorio todavía.

| # | Evidencia necesaria | Qué demuestra | Revisión antes de publicar | Estado |
|---|---|---|---|---|
| 1 | Salida de `shasum -a 256` junto a la línea correspondiente de `SHA256SUMS` | Integridad de la ISO | Ocultar el usuario de macOS en el prompt | Pendiente |
| 2 | Ajustes de la VM: procesadores, memoria y disco | 4 vCPU, 6 GiB, NVMe de 50 GB | — | Pendiente |
| 3 | Ajustes de la VM: adaptador de red | Modo NAT | Ocultar la dirección MAC | Pendiente |
| 4 | Salida de `cat /etc/os-release` y `hostnamectl` | Versión del sistema, hostname y arquitectura | Ocultar `Machine ID` y `Boot ID` | Pendiente |
| 5 | Salida de `free -h` | 5,8 GiB reconocidos | — | Pendiente |
| 6 | Salida de `lsblk -o NAME,SIZE,FSTYPE,MOUNTPOINTS` y `df -h /` | Particionado y ocupación de `/` | — | Pendiente |
| 7 | Salida de `echo $XDG_SESSION_TYPE` o pantalla «Acerca de» de GNOME | GNOME en sesión Wayland | Ocultar datos personales del escritorio | Pendiente |
| 8 | Salida de `ip -br address` y `ping -c 4 ubuntu.com` | IP por DHCP, Internet y DNS | Ocultar la dirección MAC si aparece | Pendiente |
| 9 | Salida de `ping` del endpoint al servidor (`172.16.203.128`) | Conectividad VM 2 → VM 1 | — | Pendiente |
| 10 | Salida de `ping` del servidor al endpoint (`172.16.203.129`) | Conectividad VM 1 → VM 2 | — | Pendiente |
| 11 | Salida de `apt list --upgradable` y comprobación de `/run/reboot-required` | Sistema actualizado y sin reinicio pendiente | — | Pendiente |
| 12 | Salida de `systemctl status open-vm-tools` y `vmware-toolbox-cmd -v` | Servicio activo y versión 13.0.10 | — | Pendiente |

> El funcionamiento del portapapeles compartido es difícil de demostrar con una captura estática. Basta con describir la prueba realizada (texto copiado en macOS y pegado en Ubuntu, y viceversa) sin incluir contenido real del portapapeles.

Recomendaciones:

- Nombres de archivo descriptivos, por ejemplo `vm02-06-almacenamiento.png`.
- Eliminar los metadatos de las imágenes antes de publicarlas.
- No incluir contraseñas ni contenido personal visible en el escritorio, el navegador o el portapapeles.

## 8. Seguridad

- La VM no está expuesta a Internet: se encuentra tras el NAT de VMware Fusion.
- No se documentan contraseñas ni claves en este repositorio.
- **Portapapeles compartido:** se mantiene **activado** como funcionalidad de uso habitual para el aprendizaje diario de Linux, por ejemplo para copiar comandos o salidas entre macOS y Ubuntu.
  - **Riesgo:** el portapapeles compartido, y el arrastrar y soltar si está activo, crean un canal directo de intercambio de datos entre el anfitrión y la VM, fuera de la red.
  - **Recomendación:** desactivarlo temporalmente en los ajustes de la VM en VMware Fusion durante los escenarios controlados que requieran mayor aislamiento entre el endpoint y el anfitrión, y volver a activarlo al terminar.
  - **Estado:** recomendación documentada. Todavía no se ha ejecutado ningún escenario controlado, ni se ha probado la desactivación.
- El estado del servidor SSH no se ha comprobado. Si se instala para las prácticas de autenticación, deberá restringirse a la red del laboratorio.

## 9. Estado del hito

### Completado y verificado por el estudiante

- [x] ISO de Ubuntu Desktop 24.04.5 LTS ARM64 verificada mediante SHA-256.
- [x] VM creada con 4 vCPU, 6 GiB de memoria, disco NVMe de 50 GB y red NAT.
- [x] Ubuntu Desktop 24.04.5 LTS ARM64 instalado con ext4 sin LVM, en funcionamiento.
- [x] Hostname `soc-linux-endpoint` y usuario `labuser` configurados.
- [x] Sesión GNOME sobre Wayland en funcionamiento.
- [x] Conectividad a Internet y DNS comprobada.
- [x] Conectividad ICMP bidireccional con la VM SOC-WAZUH-SERVER comprobada.
- [x] Actualizaciones aplicadas y reinicio realizado.
- [x] `open-vm-tools` y `open-vm-tools-desktop` 13.0.10 instalados y servicio activo.
- [x] Portapapeles compartido macOS–Ubuntu funcionando en Wayland.

### Pendiente

- [ ] Verificar la firma GPG de `SHA256SUMS` (mejora recomendada).
- [ ] Ejecutar las comprobaciones complementarias de la sección 5.2 y conservar sus salidas.
- [ ] Incorporar las evidencias de la sección 7 en `evidence/screenshots/vm-02/`.
- [ ] Verificar la sincronización horaria del sistema.
- [ ] Comprobar el estado del servidor SSH y, si procede, instalarlo y configurarlo para el LAB 01.
- [ ] Definir el direccionamiento IP definitivo del laboratorio.
- [ ] Confirmar la compatibilidad de Wazuh Agent con Ubuntu 24.04 ARM64 para la versión de Wazuh elegida.
- [ ] Instalar Wazuh Agent (después de instalar Wazuh en la VM 1).
- [ ] Comprobar la conectividad TCP con los puertos del servidor Wazuh y el registro del agente.
- [ ] Medir el consumo real de recursos con ambas VM en funcionamiento.

## 10. Conclusiones y limitaciones

La VM SOC-LINUX-ENDPOINT dispone de un sistema Ubuntu Desktop actualizado, integrado con VMware Fusion y con conectividad verificada hacia Internet y hacia la VM SOC-WAZUH-SERVER. Es la primera comunicación comprobada entre dos máquinas del laboratorio.

Limitaciones actuales:

- La red NAT y las IP por DHCP son provisionales.
- La conectividad comprobada es solo ICMP; la comunicación con los puertos de Wazuh no puede comprobarse hasta instalar el servidor.
- Varios resultados se verificaron sin registrar el comando exacto; las evidencias todavía no están incorporadas al repositorio.
- **Wazuh Agent todavía no está instalado**: la VM aún no cumple su función de endpoint monitorizado.
