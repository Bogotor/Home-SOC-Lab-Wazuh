# Arquitectura — Home SOC Lab con Wazuh

## 1. Objetivo de la arquitectura

Diseñar un laboratorio SOC que permita centralizar y analizar eventos de seguridad provenientes de sistemas Linux y Windows mediante Wazuh.

La infraestructura utilizará un MacBook Pro como equipo principal de virtualización y un mini PC independiente como endpoint Windows.

**Estado:** diseño preliminar. Los componentes Wazuh todavía no han sido implementados.

## 2. Infraestructura física

### MacBook Pro

- Procesador: Apple M5 Pro.
- Arquitectura: ARM64.
- Memoria unificada: 48 GB.
- Sistema operativo: macOS.
- Plataforma de virtualización prevista: VMware Fusion.

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

### VM 1 — Servidor central

Sistema operativo previsto: Ubuntu Server 24.04 LTS ARM64.

Componentes:
- Wazuh Server.
- Wazuh Indexer.
- Wazuh Dashboard.

Recursos iniciales propuestos:
- 4 vCPU.
- 12 GB RAM.
- 80 GB de almacenamiento virtual.

Estos recursos serán revisados antes de crear la máquina virtual.

### VM 2 — Endpoint Linux

Sistema operativo previsto: Ubuntu ARM64.

Componentes:
- Wazuh Agent.
- Servicio SSH para prácticas de autenticación.
- Archivos de prueba supervisados.

Recursos propuestos:
- 2 vCPU.
- 3 GB RAM.
- 30 GB de almacenamiento virtual.

## 4. Comunicación entre componentes

Los agentes Wazuh recopilarán eventos de sus respectivos sistemas y los enviarán al servidor central.

El servidor analizará los eventos mediante sus mecanismos de decodificación y reglas de detección.

Las alertas generadas se enviarán al indexador para su almacenamiento y consulta, y podrán visualizarse desde Wazuh Dashboard.

El diseño de direccionamiento IP, los puertos y el modo de conexión de red de las máquinas virtuales se documentarán después de verificar la conectividad real.

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

- Confirmar los recursos y compatibilidad del servidor Linux ARM64 con la versión de Wazuh seleccionada.
- Verificar la configuración de red de VMware Fusion.
- Definir las direcciones IP del laboratorio.
- Comprobar conectividad entre servidor y endpoints.
- Medir el consumo real de recursos.
- Verificar el funcionamiento de los servicios centrales.
- Confirmar la recepción de eventos de los agentes.

## 8. Estado

**Fase 0 — Diseño preliminar de arquitectura.**

Este documento se actualizará conforme se validen las decisiones técnicas y se implemente el laboratorio.
