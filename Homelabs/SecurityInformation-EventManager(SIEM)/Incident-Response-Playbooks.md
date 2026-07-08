# Lab: Wazuh Active Response & Troubleshooting (SSH Brute Force)

## Descripción del Proyecto
Implementación de una respuesta automatizada (Active Response) en Wazuh para mitigar ataques de fuerza bruta por SSH. Este laboratorio simula un entorno operativo real donde es necesario no solo configurar la herramienta, sino diagnosticar y resolver problemas de comunicación y correlación de alertas entre el endpoint y el SIEM mediante análisis de logs.

## Topología del Entorno
*   **SIEM (Manager):** Wazuh Manager (IP: 192.168.1.18)
*   **Víctima (Agente):** Servidor Ubuntu (IP: 10.0.2.3)
*   **Atacante:** Máquina Linux (IP: 10.0.2.4)

## Desarrollo y Resolución de Problemas (Troubleshooting)
Durante el despliegue del script de Respuesta Activa (`firewall-drop`), surgieron desafíos propios de la administración de infraestructuras, los cuales se aislaron y resolvieron mediante análisis forense de los registros del sistema:

### 1. Fallo de Comunicación (Connection Refused)
*   **Síntoma:** El Manager no registraba las alertas originadas en la máquina víctima.
*   **Diagnóstico:** Al auditar el log interno del agente (`/var/ossec/logs/ossec.log`), se detectaron errores continuos de `Connection refused` apuntando a una IP obsoleta del SIEM.
*   **Solución:** Reconfiguración de la etiqueta `<address>` en el archivo `ossec.conf` del agente hacia la IP actual del Manager y posterior reinicio del servicio (`wazuh-agentd`).

### 2. Discrepancia en la Correlación de Reglas
*   **Síntoma:** A pesar de tener conectividad y registrar el ataque, la orden de bloqueo no se ejecutaba.
*   **Diagnóstico:** Al analizar el flujo de alertas en tiempo real en el Manager (`/var/ossec/logs/alerts/alerts.log`), se comprobó que el sistema estaba clasificando los fallos de autenticación SSH locales bajo la regla `5760`, mientras que la automatización estaba configurada a la espera de la regla genérica `5716`.
*   **Solución:** Adaptación del bloque `<active-response>` en el Manager para reaccionar al `rules_id` exacto (5760).

## Resultado Final y Evidencia
Tras afinar la configuración, se ejecutó un ataque simulado contra el servicio SSH. El SIEM detectó la anomalía y ordenó instantáneamente al agente inyectar una regla de caída (`DROP`) en `iptables`, aislando al atacante y cortando todo el tráfico de red (incluyendo ICMP/Ping) durante el tiempo de *timeout* configurado.

![Evidencia del Bloqueo en iptables] <img width="518" height="231" alt="image" src="https://github.com/user-attachments/assets/a1bf51ae-648d-4d41-8582-ae3a1522617c" />

*Reglas DROP aplicadas dinámicamente por Wazuh Active Response sobre la IP atacante (10.0.2.4).*
