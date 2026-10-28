# Arquitectura — Home SOC Lab con Wazuh

## 1. Objetivo de la arquitectura

Diseñar un laboratorio SOC que permita centralizar y analizar eventos de seguridad provenientes de sistemas Linux y Windows mediante Wazuh.

La infraestructura utilizará un MacBook Pro como equipo principal de virtualización y un mini PC independiente como endpoint Windows.

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
- Administrar el laboratorio.
- Analizar eventos y documentar resultados.

### Mini PC GMKtec NUCBOX G3 PLUS

- Procesador: Intel N150.
- Núcleos: 4 físicos y 4 lógicos.
- Memoria RAM: 16 GB.
- SSD: 512 GB.
- Sistema operativo: Windows 11 Pro.
- Arquitectura: x64.

Funciones:
- Actuar como endpoint físico Windows.
- Ejecutar Wazuh Agent de forma nativa.
- Generar eventos de seguridad controlados.

No se utilizarán máquinas virtuales en el mini PC durante la implementación inicial.

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

## 4. Comunicación entre componentes

Los agentes Wazuh recopilarán eventos de sus respectivos sistemas y los enviarán al servidor central.

El servidor analizará los eventos mediante sus mecanismos de decodificación y reglas de detección.

Las alertas generadas se enviarán al indexador para su almacenamiento y consulta, y podrán visualizarse desde Wazuh Dashboard.

Configuración de red actual (provisional): las VM 1 y VM 2 utilizan la red NAT de VMware Fusion con IP asignada por DHCP, que no se considera permanente.

| Máquina | IP observada (temporal) |
|---|---|
| SOC-WAZUH-SERVER | `172.16.203.128` |
| SOC-LINUX-ENDPOINT | `172.16.203.129` |

Comprobaciones realizadas:
- Acceso SSH a la VM 1 desde el anfitrión.
- Conectividad ICMP bidireccional entre la VM 1 y la VM 2 (0 % de pérdida, latencia media inferior a 1 ms).

Sin comprobar:
- Conectividad TCP del endpoint Linux con los puertos de Wazuh (los servicios todavía no existen).
- Alcance del servidor desde el endpoint Windows físico en este modo de red.

El diseño definitivo de direccionamiento IP, los puertos y el modo de conexión de red de las máquinas virtuales se documentará después de verificar la conectividad real.

## 5. Justificación de la arquitectura

La decisión principal consiste en distribuir la carga de procesamiento según los recursos disponibles.

El MacBook Pro dispone de mayor capacidad de memoria y procesamiento, por lo que se propone alojar los componentes centrales de Wazuh en este equipo.

El mini PC conserva Windows 11 Pro y funciona como endpoint físico independiente, evitando la carga adicional de ejecutar máquinas virtuales.

Esta distribución también permite estudiar monitoreo de seguridad en Linux y Windows.

## 6. Restricciones de seguridad

- Utilizar únicamente sistemas propios o autorizados.
- Restringir el acceso al Dashboard.
- No exponer directamente Wazuh a Internet.
- Mantener contraseñas y certificados fuera del repositorio.
- Documentar únicamente evidencias verificadas y debidamente anonimizadas.

## 7. Validaciones pendientes

- Confirmar la compatibilidad de Ubuntu Server 24.04 ARM64 con la versión de Wazuh seleccionada.
- Definir el modo de red definitivo de VMware Fusion (actualmente NAT).
- Definir las direcciones IP del laboratorio.
- Comprobar conectividad con el endpoint Windows físico (la conectividad ICMP entre VM 1 y VM 2 ya está comprobada).
- Confirmar la compatibilidad de Wazuh Agent con Ubuntu 24.04 ARM64.
- Aplicar hardening de SSH en el servidor (autenticación por claves).
- Comprobar el estado del servidor SSH en la VM 2.
- Verificar la sincronización horaria de ambas VM.
- Medir el consumo real de recursos con ambas VM en funcionamiento.
- Verificar el funcionamiento de los servicios centrales.
- Confirmar la recepción de eventos de los agentes.

## 8. Estado

**Fase 2 — Preparación de infraestructura (en proceso).**

- VM 1 (SOC-WAZUH-SERVER): Ubuntu Server instalado, actualizado, en funcionamiento y accesible por SSH desde el anfitrión.
- Wazuh: todavía no instalado.
- VM 2 (SOC-LINUX-ENDPOINT): Ubuntu Desktop instalado, actualizado, en funcionamiento y con conectividad ICMP con la VM 1. Wazuh Agent todavía no instalado.
- Endpoint Windows: pendiente de incorporación.

Este documento se actualizará conforme se validen las decisiones técnicas y se implemente el laboratorio.
