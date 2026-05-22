# Despliegue de SIEM y Monitorización de Integridad (FIM) con Wazuh

En esta fase del laboratorio, implementamos una arquitectura de monitorización centralizada instalando un Agente Wazuh en el servidor bastionado. El objetivo es recopilar telemetría en tiempo real, detectar intentos de intrusión y monitorizar la integridad de los archivos críticos del sistema (FIM).

## 1. Despliegue del Agente Wazuh (Endpoint)

Para un servidor basado en Ubuntu (familia Debian) con arquitectura de 64 bits, procedemos a la instalación mediante paquetes `.deb` y vinculación con el Wazuh Manager.

### Descarga e Instalación

```bash
# Descarga del paquete oficial del Agente Wazuh
wget [https://packages.wazuh.com/4.x/apt/pool/main/w/wazuh-agent/wazuh-agent_4.7.5-1_amd64.deb](https://packages.wazuh.com/4.x/apt/pool/main/w/wazuh-agent/wazuh-agent_4.7.5-1_amd64.deb)

# Instalación del paquete
sudo apt-get install ./wazuh-agent_4.7.5-1_amd64.deb
```

### Vinculación y Arranque del Servicio

Se configura la IP del Wazuh Manager y se habilita el servicio para que arranque con el sistema operativo.

```bash
# Recargar demonios del sistema
sudo systemctl daemon-reload

# Habilitar e iniciar el Agente Wazuh
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent
```

## 2. Detección de Intrusiones y Mapeo MITRE ATT&CK

Una vez establecida la conexión criptográfica entre el Endpoint y el SIEM, procedemos a validar la ingesta de logs simulando un ataque de fuerza bruta por SSH.

### Simulación de Ataque

Desde un equipo externo, generamos accesos fallidos intencionados contra el puerto securizado:

```bash
ssh -p 4422 intruso@<IP_DEL_SERVIDOR_BASTIONADO>
```

### Validación en el SOC

El panel de Wazuh detecta y categoriza el ataque exitosamente:

*   **Rule ID:** 5710 (`sshd: Attempt to login using a non-existent user`).
*   **Framework MITRE ATT&CK:** Wazuh mapea automáticamente este comportamiento a la táctica de **Credential Access** y la técnica **T1110.001 (Password Guessing)**.

> **Nota:** En este punto, si se superan los reintentos permitidos, el servicio `fail2ban` configurado en fases anteriores banea la IP del atacante, interactuando perfectamente con el SIEM.

## 3. Monitorización de Integridad de Archivos (FIM)

Se configura el módulo `syscheck` para vigilar alteraciones en directorios críticos y detectar posibles persistencias de malware o modificaciones no autorizadas en tiempo real mediante notificaciones *inotify* del kernel de Linux.

### Configuración del Agente (`ossec.conf`)

Editamos el archivo de configuración central del agente:

```bash
sudo nano /var/ossec/etc/ossec.conf
```

Añadimos las directivas de tiempo real y reporte de cambios exactos (`report_changes="yes"`) para los directorios deseados:

```xml
<syscheck>
  <directories realtime="yes" report_changes="yes">/etc</directories>
  <directories realtime="yes">/home/dani-lab/trampa_soc</directories>
</syscheck>
```

Reiniciamos el agente para establecer la nueva Línea Base (*Baseline*):

```bash
sudo systemctl restart wazuh-agent
```

### Prueba de Concepto (PoC) y Validación

Simulamos la creación de un archivo malicioso en un entorno controlado (Sandbox) para forzar la detección del motor FIM:

```bash
# Creación e inyección de datos simulando un malware
echo "Código malicioso" > /home/dani-lab/trampa_soc/virus.txt
```

**Resultado:** El SIEM ingiere la alerta de forma inmediata, catalogando el evento (`syscheck.event: added`) e informando de la ruta exacta comprometida, demostrando la eficacia del sistema FIM ante modificaciones no autorizadas.

## 4. Troubleshooting y Auditoría Avanzada

Durante el despliegue, aplicamos técnicas de diagnóstico forense para garantizar la correcta comunicación Cliente-Servidor esquivando la interfaz gráfica:

```bash
# Auditoría de estado del FIM en el Agente Local
sudo grep "syscheck" /var/ossec/logs/ossec.log | tail -n 10

# Búsqueda de ingesta cruda de alertas en el Wazuh Manager
sudo grep -i "virus.txt" /var/ossec/logs/alerts/alerts.log
```

Esta metodología permite a los analistas descartar falsos negativos generados por umbrales de severidad bajos (como la regla 554 de Wazuh) o cuellos de botella en el motor *inotify* del sistema operativo.
```</IP_DEL_SERVIDOR_BASTIONADO>
