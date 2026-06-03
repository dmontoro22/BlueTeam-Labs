# Blue Team Homelabs & Security Operations

Bienvenido a mi repositorio de laboratorios de ciberseguridad. Soy **Daniel Montoro Sánchez**, Analista Junior en Ciberseguridad, y este espacio documenta mi entorno de pruebas, despliegues y operaciones orientadas a la defensa (Blue Team).

El objetivo de este repositorio es aplicar conocimientos teóricos en entornos prácticos, emulando escenarios reales de fortificación de sistemas, detección de intrusiones, respuesta a incidentes y operaciones de seguridad.

---

## Arquitectura del Laboratorio (Roadmap)

Este entorno está diseñado para evolucionar desde la seguridad de un endpoint individual hasta la monitorización centralizada y la seguridad en la nube.

### 1. Endpoint & Server Security (Fase de Prevención)  En curso
Fortificación de sistemas base y establecimiento de líneas seguras.
* [x] **Hardening Inicial en Servidores Linux (Debian/Ubuntu):** Configuración de políticas de acceso, SSH, gestión de usuarios y despliegue de `auditd`.
* [X] **Benchmark CIS:** Auditoría y aplicación de los controles de seguridad del *Center for Internet Security* sobre la línea base.

### 2. Network Security & Analysis (Fase de Visibilidad)  En curso
Inspección de tráfico y detección de amenazas en la red.
* [x] **Análisis de Tráfico con herramientas de terminal:** Captura y decodificación manual de credenciales y tráfico no cifrado en archivos `.pcap` mediante `tcpdump`.
* [ ] **Network Intrusion Detection System (NIDS):** Despliegue de Suricata/Snort en red doméstica para la generación de alertas automatizadas.

### 3. Security Information and Event Management - SIEM (Fase de Operaciones)  Pendiente
Centralización, correlación de logs y respuesta.
* [ ] **Home Lab SIEM con Wazuh:** Despliegue del manager, conexión de agentes en los endpoints reales e ingesta de alertas del NIDS.
* [ ] **Threat Intelligence Automation:** Script en Python para ingestar Indicadores de Compromiso (IoCs) desde fuentes abiertas hacia el SIEM.
* [ ] **Incident Response Playbooks:** Creación de respuestas activas (Active Response) frente a detecciones específicas.

### 4. Cloud Security & Architecture  Pendiente
Extensión de las operaciones de seguridad a entornos Cloud (Microsoft Azure).
* [ ] **Hardening de entorno Cloud:** Topología de red, NSGs y control de accesos. Proyecto puente para consolidar conocimientos de infraestructura en la nube.

### 5. Awareness & Human Risk  Pendiente
* [ ] **Campaña de Phishing Ético:** Simulacro y redacción de programa de concienciación. 

---

##  Tecnologías y Herramientas utilizadas
* **Sistemas Operativos:** Linux (Debian/Ubuntu)
* **Monitorización y Detección:** Wazuh, Suricata, Auditd
* **Análisis de Red:** tcpdump
* **Automatización:** Python, Bash

---
*Todo esto se encuentra en "Homelabs"* --- *Repositorio en constante actualización.* 
