# Despliegue de SIEM (Wazuh) y NIDS (Suricata) en Entorno Heterogéneo

## Arquitectura del Laboratorio

La infraestructura se distribuye en máquinas virtuales bajo el hipervisor Oracle VirtualBox, organizadas de la siguiente manera:

* **Wazuh Manager (AIO):** Servidor central que aloja el Indexer, Server y Dashboard. Configurado en modo Adaptador Puente para actuar como nodo centralizador en la red local.
* **Suricata-Lab (NIDS):** Sensor de red dedicado con el motor Suricata. Cuenta con un agente de Wazuh local encargado de monitorizar y reenviar el archivo de eventos en tiempo real.
* **Windows10-lab-VPN (Endpoint):** Estación de trabajo Windows 10 monitorizada a nivel de sistema mediante el Visor de Eventos nativo (Security, System, Application).
* **UBUNTU-CIS (Endpoint):** Servidor Linux securizado integrado bajo una arquitectura de Red NAT. El flujo de telemetría opera de manera saliente hacia el Manager.

---

## Fases de Implementación

### 1. Preparación del Core SIEM
Se verificó la correcta ejecución de los servicios esenciales en el nodo centralizador para garantizar la disponibilidad del backend y la recepción de conexiones cifradas:

    sudo systemctl status wazuh-manager
    sudo systemctl status wazuh-indexer

### 2. Integración del Sensor de Red (NIDS)
Tras la actualización de las firmas de Emerging Threats mediante `suricata-update`, se enlazó el flujo de alertas del archivo `eve.json` con el motor de análisis de Wazuh. La monitorización local se configuró en el archivo del agente mediante la siguiente estructura:

    <localfile>
      <log_format>json</log_format>
      <location>/var/log/suricata/eve.json</location>
    </localfile>

### 3. Despliegue de Agentes en Endpoints

**Nodo Windows**
La instalación se realizó mediante una consola de PowerShell con privilegios elevados, abstrayendo la instalación gráfica mediante parámetros silenciosos y asignando los parámetros del manager en el registro de Windows:

    msiexec.exe /i wazuh-agent-4.7.5-1.msi /q WAZUH_MANAGER='192.168.1.54'
    NET START WazuhSvc

**Nodo Linux (Entorno Red NAT)**
Para mitigar el aislamiento de la Red NAT y permitir la administración remota por SSH desde el anfitrión, se implementó una regla de reenvío de puertos (Port Forwarding) en el hipervisor (Puerto Anfitrión 2222 -> Puerto Invitado 22). 

Una vez establecida la sesión SSH, se procedió al despliegue del paquete de Debian apuntando al Manager:

    sudo WAZUH_MANAGER='192.168.1.54' dpkg -i ./wazuh-agent.deb
    sudo systemctl enable wazuh-agent && sudo systemctl start wazuh-agent

---

## Verificación y Triaje de Eventos

La validación del funcionamiento del entorno se realizó inyectando tráfico malicioso controlado mediante una petición HTTP diseñada para hacer match con las firmas del NIDS:

    curl http://testmyids.com

### Análisis en el SIEM
El motor de Suricata registró el evento localmente bajo la firma `GPL ATTACK_RESPONSE id check returned root` (SID: 2100498). Posteriormente, el agente de Wazuh remitió el log en formato JSON al Manager, mapeándolo de forma nativa dentro del grupo de reglas de Suricata.

<img width="2557" height="1272" alt="image" src="https://github.com/user-attachments/assets/da500f44-8df5-44b0-9216-8b921d838e23" />

<img width="2555" height="1268" alt="image" src="https://github.com/user-attachments/assets/60a7f184-ded1-462a-93f4-80df8db29496" />

