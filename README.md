# Home SOC Lab — Wazuh

Laboratorio personal de ciberseguridad defensiva (Blue Team) orientado al aprendizaje de operaciones SOC, administración de sistemas y análisis de eventos de seguridad mediante Wazuh.

## 1. Objetivo

Diseñar, implementar y documentar un entorno SOC que permita centralizar registros de seguridad, monitorear endpoints Linux y Windows, analizar alertas y realizar simulaciones controladas de actividades relevantes para la seguridad.

El proyecto está orientado al desarrollo de competencias técnicas para un puesto de Analista SOC Junior.

## 2. Infraestructura

| Equipo | Características | Función prevista |
|---|---|---|
| MacBook Pro | Apple M5 Pro, 48 GB de memoria, macOS | Virtualización y plataforma central de monitoreo |
| GMKtec NUCBOX G3 PLUS | Intel N150, 16 GB RAM, 512 GB SSD, Windows 11 Pro | Endpoint Windows monitoreado |
| Máquina virtual Linux | Ubuntu Server ARM64, configuración pendiente | Servidor central Wazuh |
| Máquina virtual Linux adicional | Ubuntu ARM64, configuración pendiente | Endpoint Linux de pruebas |

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

**Fase actual: 0 — Diagnóstico y planificación.**

Completado:
- Diagnóstico inicial de conocimientos.
- Identificación de los sistemas operativos.
- Verificación del procesador del mini PC.
- Inicialización del repositorio Git local.

Pendiente:
- Validación definitiva de la arquitectura.
- Configuración de las máquinas virtuales.
- Instalación de Wazuh.
- Incorporación de agentes.
- Ejecución y documentación de laboratorios.
- Validación final.

## 6. Documentación

El repositorio incorporará progresivamente documentación de arquitectura, procedimientos de instalación, configuraciones, laboratorios, evidencias verificadas e investigaciones de seguridad.

Los resultados simulados serán identificados como tales y no se presentarán como incidentes reales.

## 7. Alcance y seguridad

Todas las actividades se realizarán sobre infraestructura propia o autorizada, en un entorno controlado.

El proyecto tiene fines educativos y de desarrollo profesional. No se publicarán contraseñas, claves privadas, tokens ni información sensible.
