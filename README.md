# Home SOC Lab — Wazuh

Laboratorio personal de ciberseguridad defensiva (Blue Team) orientado al aprendizaje de operaciones SOC, administración de sistemas y análisis de eventos de seguridad mediante Wazuh.

## 1. Objetivo

Diseñar, implementar y documentar un entorno SOC que permita centralizar registros de seguridad, monitorear endpoints Linux y Windows, analizar alertas y realizar simulaciones controladas de actividades relevantes para la seguridad.

El proyecto está orientado al desarrollo de competencias técnicas para un puesto de Analista SOC Junior.

## 2. Infraestructura

| Equipo | Características | Función prevista |
|---|---|---|
| MacBook Pro | Apple M5 Pro, 48 GB de memoria, macOS, VMware Fusion Professional | Virtualización de todas las máquinas y plataforma central de monitoreo |
| VM SOC-WAZUH-SERVER | Ubuntu Server 24.04.5 LTS ARM64 (instalado y en funcionamiento), 4 vCPU, 12 GiB RAM, 60 GB de disco | Servidor central Wazuh (Wazuh todavía no instalado) |
| VM SOC-LINUX-ENDPOINT | Ubuntu Desktop 24.04.5 LTS ARM64 (instalado y en funcionamiento), GNOME/Wayland, 4 vCPU, 6 GiB RAM, 50 GB de disco | Endpoint Linux y entorno de aprendizaje de Linux (Wazuh Agent todavía no instalado) |
| VM SOC-WINDOWS-ENDPOINT (prevista) | Windows 11 para ARM | Endpoint Windows monitoreado (sin crear) |

El endpoint Windows iba a ser un mini PC físico, que dejó de estar disponible; se sustituye por una VM. Todas las VM comparten una red NAT de VMware Fusion, sin puertos expuestos en la red local.

La arquitectura es preliminar y está sujeta a validaciones de compatibilidad y recursos.

## 3. Tecnologías previstas

- Wazuh: monitoreo, recolección y análisis de eventos de seguridad.
- Linux: administración del servidor y generación de eventos.
- Windows 11 para ARM: endpoint Windows virtualizado y monitoreado.
- VMware Fusion: virtualización en Apple Silicon.
- Git y GitHub: control de versiones y documentación.

## 4. Laboratorios planificados

- LAB 01: Análisis de intentos fallidos de autenticación SSH.
- LAB 02: Monitoreo de integridad de archivos (FIM).
- LAB 03: Detección de cambios administrativos.
- Investigación y documentación de alertas generadas en el entorno de pruebas.

## 5. Estado del proyecto

Fases del proyecto:

- Fase 0: Diagnóstico.
- Fase 1: Fundamentos SOC y arquitectura.
- Fase 2: Preparación de infraestructura.

**Fase actual: 2 — Preparación de infraestructura (en proceso).**

Las dos máquinas Linux del laboratorio están preparadas: Ubuntu Server en la VM SOC-WAZUH-SERVER y Ubuntu Desktop en la VM SOC-LINUX-ENDPOINT, con conectividad ICMP comprobada entre ambas. **Wazuh todavía no está instalado** (ni el servidor ni los agentes).

Completado:
- Diagnóstico inicial de conocimientos.
- Identificación de los sistemas operativos.
- Verificación del procesador del mini PC (equipo retirado posteriormente).
- Inicialización del repositorio Git y publicación en GitHub.
- Creación de la VM SOC-WAZUH-SERVER e instalación y validación inicial de Ubuntu Server 24.04.5 LTS ARM64 ([documentación](documentation/vm-01-setup.md)).
- Creación de la VM SOC-LINUX-ENDPOINT e instalación y validación inicial de Ubuntu Desktop 24.04.5 LTS ARM64, con open-vm-tools y conectividad ICMP bidireccional con el servidor ([documentación](documentation/vm-02-setup.md)).
- Hardening de SSH en SOC-WAZUH-SERVER: acceso mediante clave Ed25519 comprobado y autenticación por contraseña rechazada en una prueba controlada ([documentación](documentation/ssh-hardening.md)).
- Comprobaciones previas a Wazuh: hora y NTP, estado de systemd, recursos, arquitectura y puertos libres. Elegido Wazuh 4.14.9 con el asistente de instalación por etapas.
- Red NAT personalizada del laboratorio creada en VMware Fusion, con el servidor ya conectado a ella ([arquitectura](architecture/architecture.md)).

Pendiente:
- Validación definitiva de la arquitectura.
- Completar la red del laboratorio: reservar la IP del servidor y mover el endpoint Linux a la nueva red.
- Prueba del comportamiento de `PermitRootLogin no` en SOC-WAZUH-SERVER (confirmado en la configuración, sin prueba directa de inicio de sesión como root).
- Incorporación de evidencias de las VM SOC-WAZUH-SERVER y SOC-LINUX-ENDPOINT y del hardening de SSH.
- Instalación de Wazuh.
- Instalación de Wazuh Agent en el endpoint Linux.
- Creación de la VM Windows 11 para ARM y prueba de Wazuh Agent en ella (no hay paquete ARM64 documentado).
- Ejecución y documentación de laboratorios.
- Validación final.

## 6. Documentación

El repositorio incorporará progresivamente documentación de arquitectura, procedimientos de instalación, configuraciones, laboratorios, evidencias verificadas e investigaciones de seguridad.

Los resultados simulados serán identificados como tales y no se presentarán como incidentes reales.

## 7. Alcance y seguridad

Todas las actividades se realizarán sobre infraestructura propia o autorizada, en un entorno controlado.

El proyecto tiene fines educativos y de desarrollo profesional. No se publicarán contraseñas, claves privadas, tokens ni información sensible.
