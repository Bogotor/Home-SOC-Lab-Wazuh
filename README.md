# Home SOC Lab — Wazuh

Laboratorio personal de ciberseguridad defensiva (Blue Team) orientado al aprendizaje de operaciones SOC, administración de sistemas y análisis de eventos de seguridad mediante Wazuh.

## 1. Objetivo

Diseñar, implementar y documentar un entorno SOC que permita centralizar registros de seguridad, monitorear endpoints Linux y Windows, analizar alertas y realizar simulaciones controladas de actividades relevantes para la seguridad.

El proyecto está orientado al desarrollo de competencias técnicas para un puesto de Analista SOC Junior.

## 2. Infraestructura

| Equipo | Características | Función prevista |
|---|---|---|
| MacBook Pro | Apple M5 Pro, 48 GB de memoria, macOS, VMware Fusion Professional | Virtualización y plataforma central de monitoreo |
| GMKtec NUCBOX G3 PLUS | Intel N150, 16 GB RAM, 512 GB SSD, Windows 11 Pro | Endpoint Windows monitoreado |
| VM SOC-WAZUH-SERVER | Ubuntu Server 24.04.5 LTS ARM64 (instalado y en funcionamiento), 4 vCPU, 12 GiB RAM, 60 GB de disco | Servidor central Wazuh (Wazuh todavía no instalado) |
| Máquina virtual Linux adicional (VM 2) | Ubuntu ARM64, pendiente de creación | Endpoint Linux de pruebas |

La arquitectura es preliminar y está sujeta a validaciones de compatibilidad y recursos.

## 3. Tecnologías previstas

- Wazuh: monitoreo, recolección y análisis de eventos de seguridad.
- Linux: administración del servidor y generación de eventos.
- Windows 11 Pro: endpoint físico monitoreado.
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

Ubuntu Server está instalado y en funcionamiento en la VM SOC-WAZUH-SERVER. **Wazuh todavía no está instalado.**

Completado:
- Diagnóstico inicial de conocimientos.
- Identificación de los sistemas operativos.
- Verificación del procesador del mini PC.
- Inicialización del repositorio Git y publicación en GitHub.
- Creación de la VM SOC-WAZUH-SERVER e instalación y validación inicial de Ubuntu Server 24.04.5 LTS ARM64 ([documentación](documentation/vm-01-setup.md)).

Pendiente:
- Validación definitiva de la arquitectura.
- Configuración de red definitiva del laboratorio (actualmente NAT provisional).
- Hardening de SSH en el servidor (migración a autenticación por claves).
- Incorporación de evidencias de la VM SOC-WAZUH-SERVER.
- Instalación de Wazuh.
- Creación del endpoint Linux (VM 2).
- Incorporación de agentes.
- Ejecución y documentación de laboratorios.
- Validación final.

## 6. Documentación

El repositorio incorporará progresivamente documentación de arquitectura, procedimientos de instalación, configuraciones, laboratorios, evidencias verificadas e investigaciones de seguridad.

Los resultados simulados serán identificados como tales y no se presentarán como incidentes reales.

## 7. Alcance y seguridad

Todas las actividades se realizarán sobre infraestructura propia o autorizada, en un entorno controlado.

El proyecto tiene fines educativos y de desarrollo profesional. No se publicarán contraseñas, claves privadas, tokens ni información sensible.
