# Arquitectura — Home SOC Lab con Wazuh

## 1. Objetivo de la arquitectura

Diseñar un laboratorio SOC que permita centralizar y analizar eventos de seguridad provenientes de sistemas Linux y Windows mediante Wazuh.

La infraestructura utiliza un MacBook Pro como único equipo físico: aloja todas las máquinas del laboratorio, incluida una futura VM con Windows 11 para ARM como endpoint Windows. El mini PC previsto inicialmente para ese papel dejó de estar disponible (sección 2).

**Estado:** VM del servidor central y VM del endpoint Linux creadas, con sus sistemas operativos instalados y en funcionamiento. **Wazuh todavía no está instalado** (ni el servidor ni los agentes).

## 2. Infraestructura física

### MacBook Pro

- Procesador: Apple M5 Pro.
- Arquitectura: ARM64.
- Memoria unificada: 48 GB.
- Sistema operativo: macOS.
- Plataforma de virtualización: VMware Fusion Professional 26H1u1 (build 2689522).

Funciones:
- Alojar el servidor central de Wazuh mediante Linux virtualizado.
- Ejecutar un endpoint Linux de pruebas.
- Ejecutar un endpoint Windows virtualizado (previsto).
- Administrar el laboratorio.
- Analizar eventos y documentar resultados.

### Mini PC GMKtec NUCBOX G3 PLUS (retirado)

Estaba previsto como endpoint Windows físico (Intel N150 x64, 16 GB de RAM, SSD de 512 GB, Windows 11 Pro), con Wazuh Agent nativo. Dejó de estar disponible en noviembre de 2026, antes de incorporarse al laboratorio, y se sustituye por la VM 3 (sección 3). Se conserva esta nota como histórico de la decisión.

## 3. Arquitectura lógica propuesta

### VM 1 — Servidor central (SOC-WAZUH-SERVER)

Sistema operativo: Ubuntu Server 24.04.5 LTS ARM64 (instalado y en funcionamiento).

Hostname: `soc-wazuh-server`.

Componentes previstos (todavía no instalados):
- Wazuh Server.
- Wazuh Indexer.
- Wazuh Dashboard.

Recursos configurados:
- 4 vCPU.
- 12 GiB RAM.
- 60 GB de almacenamiento virtual sin preasignación (propuesta inicial: 80 GB).

Almacenamiento: ext4 sobre LVM para el volumen raíz (`/` 50 GB), `/boot` 2 GB, EFI ≈ 1 GB y ≈ 6,9 GB libres en el grupo de volúmenes LVM para futuras ampliaciones.

Procedimiento y verificaciones: [`documentation/vm-01-setup.md`](../documentation/vm-01-setup.md).

Hardening de SSH: [`documentation/ssh-hardening.md`](../documentation/ssh-hardening.md).

### VM 2 — Endpoint Linux (SOC-LINUX-ENDPOINT)

**Estado:** VM creada; Ubuntu Desktop instalado, actualizado y en funcionamiento. **Wazuh Agent todavía no está instalado.**

Sistema operativo: Ubuntu Desktop 24.04.5 LTS ARM64, entorno GNOME en sesión Wayland.

Hostname: `soc-linux-endpoint`.

Funciones:
- Entorno de aprendizaje diario de Linux.
- Futuro endpoint Linux monitorizado por Wazuh.

Componentes previstos (todavía no instalados o no comprobados):
- Wazuh Agent.
- Servicio SSH para prácticas de autenticación (estado no comprobado en la VM; Ubuntu Desktop no incluye el servidor OpenSSH por defecto).
- Archivos de prueba supervisados.

Componentes instalados: `open-vm-tools` y `open-vm-tools-desktop` 13.0.10 (integración con VMware Fusion y portapapeles compartido).

Recursos configurados:
- 4 vCPU.
- 6 GiB RAM (5,8 GiB reconocidos por Linux).
- 50 GB de almacenamiento virtual NVMe.

La propuesta inicial (2 vCPU, 3 GB RAM, 30 GB) se amplió al elegir una edición con escritorio gráfico.

Almacenamiento: partición EFI 1 GB (vfat) y partición raíz `/` 48,9 GB en ext4, sin LVM.

Procedimiento y verificaciones: [`documentation/vm-02-setup.md`](../documentation/vm-02-setup.md).

### VM 3 — Endpoint Windows (SOC-WINDOWS-ENDPOINT, prevista)

**Estado:** sin crear. Se creará después de instalar Wazuh central.

Sistema operativo previsto: Windows 11 para ARM. En Apple Silicon, VMware Fusion solo ejecuta sistemas ARM, por lo que no es posible usar Windows x64.

Funciones:
- Endpoint Windows monitorizado por Wazuh.
- Laboratorios de Windows Event Logs, PowerShell y servicios de Windows.

Red prevista: `vmnet2`, la misma que el resto de VM (sección 4).

Recursos: por definir al crear la VM. Microsoft exige al menos 4 GB de RAM, 64 GB de almacenamiento y TPM 2.0 (información técnica general). Fusion añade un TPM virtual a las VM con Windows 11 (información técnica general, sin comprobar en 26H1).

Riesgos y requisitos por confirmar:
- **Agente de Wazuh:** la lista oficial de paquetes publica el MSI de Windows como «32/64 bits», sin versión ARM64. Windows 11 para ARM ejecuta programas x64 por emulación, pero el funcionamiento del agente no está documentado y debe probarse. Si fallara, el endpoint Windows quedaría bloqueado hasta disponer de otro equipo x64.
- **Licencia:** el uso de Windows requiere una licencia válida.
- **Representatividad:** un Windows ARM virtualizado no equivale a un puesto x64 típico de empresa; las diferencias se documentarán si afectan a los laboratorios.

## 4. Comunicación entre componentes

Los agentes Wazuh recopilarán eventos de sus respectivos sistemas y los enviarán al servidor central.

El servidor analizará los eventos mediante sus mecanismos de decodificación y reglas de detección.

Las alertas generadas se enviarán al indexador para su almacenamiento y consulta, y podrán visualizarse desde Wazuh Dashboard.

### Diseño de red

Todas las VM del laboratorio se conectarán a `vmnet2`, una red NAT personalizada de VMware Fusion creada el 2026-11-04:

| Elemento | Valor |
|---|---|
| Subred | `172.16.50.0/24` |
| MacBook Pro en la red | `172.16.50.1` |
| Puerta de enlace NAT | `172.16.50.2` |
| Rango DHCP | `172.16.50.128` a `172.16.50.254` |
| Reservas DHCP | `172.16.50.10` para la VM 1, fuera del rango dinámico y asociada a su MAC estática |
| Reenvío de puertos | Ninguno |

Con todas las VM en la misma red, los agentes alcanzan directamente al servidor y no hace falta reenviar puertos. Ningún servicio queda expuesto en la red local física: el Dashboard y la API solo son accesibles desde el MacBook Pro y las VM.

**Decisión anterior descartada:** mientras el endpoint Windows era el mini PC, que estaba en la red local física, se diseñó un reenvío de los puertos `1514` y `1515` a través del MacBook Pro. Exponía esos puertos en cualquier red a la que se conectara el portátil. Al retirarse el mini PC, ese reenvío ya no es necesario y no llegó a configurarse.

### Estado de la red

| Máquina | Red | IP | Estado |
|---|---|---|---|
| SOC-WAZUH-SERVER | `vmnet2` | `172.16.50.10` | Fija: reserva DHCP y MAC estática (2026-11-26) |
| SOC-LINUX-ENDPOINT | NAT inicial (`172.16.203.0/24`) | `172.16.203.129` | Pendiente de mover a `vmnet2` |
| SOC-WINDOWS-ENDPOINT | `vmnet2` (prevista) | — | Sin crear |

Mientras la VM 2 siga en la NAT inicial, no puede alcanzar a la VM 1 (inferido por estar en redes distintas, sin comprobar).

Comprobaciones realizadas:
- Acceso SSH a la VM 1 desde el anfitrión mediante clave Ed25519, con autenticación por contraseña rechazada en una prueba controlada.
- Conectividad ICMP bidireccional entre la VM 1 y la VM 2 cuando ambas estaban en la NAT inicial (0 % de pérdida, latencia media inferior a 1 ms).
- VM 1 en `vmnet2` (2026-11-04): IP `172.16.50.128`, ruta por defecto por `172.16.50.2`, `ping` a `ubuntu.com` con 0 % de pérdida y acceso SSH desde el anfitrión con la nueva IP.
- VM 1 con IP fija (2026-11-26): MAC estática `00:50:56:00:50:10` e IPv4 `172.16.50.10/24` comprobadas en la VM, y SSH por clave correcto con contraseña rechazada en la nueva IP. Detalle en [`documentation/vm-01-setup.md`](../documentation/vm-01-setup.md).

Sin comprobar:
- Conectividad de la VM 2 con la VM 1 en `vmnet2`.
- Conectividad TCP de los endpoints con los puertos de Wazuh (los servicios todavía no existen).

## 5. Justificación de la arquitectura

La decisión principal consiste en distribuir la carga de procesamiento según los recursos disponibles.

El MacBook Pro dispone de mayor capacidad de memoria y procesamiento, por lo que se propone alojar los componentes centrales de Wazuh en este equipo.

El endpoint Windows también se virtualiza en el MacBook Pro (decisión del 2026-11-26), porque el mini PC dejó de estar disponible. Compartir la red `vmnet2` simplifica la conectividad y evita exponer puertos en la red local. A cambio, el endpoint Windows es ARM y el agente de Wazuh depende de la emulación x64 de Windows, pendiente de probar (sección 3).

Esta distribución también permite estudiar monitoreo de seguridad en Linux y Windows.

## 6. Restricciones de seguridad

- Utilizar únicamente sistemas propios o autorizados.
- Restringir el acceso al Dashboard.
- No exponer directamente Wazuh a Internet.
- Mantener contraseñas y certificados fuera del repositorio.
- Documentar únicamente evidencias verificadas y debidamente anonimizadas.

## 7. Validaciones pendientes

- Comprobar al instalar la compatibilidad ARM64 documentada de Wazuh 4.14.9.
- Mover la VM 2 a `vmnet2` y comprobar su conectividad con la VM 1.
- Comprobar el comportamiento de `PermitRootLogin no` en la VM 1 (confirmado en la configuración, sin prueba directa de inicio de sesión como root).
- Comprobar el estado del servidor SSH en la VM 2.
- Crear la VM 3: confirmar los requisitos de Windows 11 para ARM en Fusion 26H1 y disponer de licencia.
- Comprobar que Wazuh Agent funciona en Windows 11 para ARM, sin paquete ARM64 documentado.
- Verificar la sincronización horaria de la VM 3.
- Medir el consumo real de recursos con Wazuh en funcionamiento.
- Verificar el funcionamiento de los servicios centrales.
- Confirmar la recepción de eventos de los agentes.

## 8. Estado

**Fase 2 — Preparación de infraestructura (en proceso).**

- VM 1 (SOC-WAZUH-SERVER): Ubuntu Server instalado, actualizado, en funcionamiento y accesible por SSH desde el anfitrión mediante clave.
- Wazuh: todavía no instalado. Versión prevista: 4.14.9, con el asistente por etapas. Compatibilidad con ARM64 documentada, sin probar su instalación.
- VM 2 (SOC-LINUX-ENDPOINT): Ubuntu Desktop instalado, actualizado, en funcionamiento y con conectividad ICMP con la VM 1. Wazuh Agent todavía no instalado.
- Endpoint Windows: el mini PC dejó de estar disponible. Se sustituirá por una VM con Windows 11 para ARM en `vmnet2`, después de instalar Wazuh central.
- Red: `vmnet2` creada (2026-11-04). La VM 1 ya está en ella con IP fija `172.16.50.10` (2026-11-26); la VM 2 sigue en la NAT inicial.
- Sincronización horaria: comprobada en la VM 1 y la VM 2 con `timedatectl` (2026-11-02), sin medir el desfase real.
- Recursos: medidos en reposo, sin Wazuh y con ambas VM encendidas (2026-11-02). El detalle está en los documentos de cada VM. En el MacBook Pro: 597 GiB de disco disponibles y 72 % de memoria libre.
- Almacenamiento de la VM 1: se mantiene sin ampliar antes de instalar Wazuh, con el espacio libre de LVM como reserva. La decisión está justificada en [`documentation/vm-01-setup.md`](../documentation/vm-01-setup.md).

Este documento se actualizará conforme se validen las decisiones técnicas y se implemente el laboratorio.
