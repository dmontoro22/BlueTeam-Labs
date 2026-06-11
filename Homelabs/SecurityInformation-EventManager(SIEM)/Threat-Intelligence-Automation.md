# Proyecto de Laboratorio Blue Team: Integración de Cyber Threat Intelligence (CTI) y Troubleshooting en Wazuh SIEM

---

## 1. Resumen
Este proyecto documenta el despliegue, la resolución de problemas estructurales (Troubleshooting) y la validación táctica de un entorno de monitorización unificado **SIEM (Security Information and Event Management)** basado en **Wazuh (v4.7.5)**. 

El objetivo principal del laboratorio es centralizar la telemetría de un servidor corporativo expuesto (**Ubuntu-CIS**) y enriquecer los eventos detectados mediante la ingesta automática de **Indicadores de Compromiso (IoCs)** procedentes de bases de datos de Inteligencia de Amenazas (**Cyber Threat Intelligence - CTI**), específicamente de **AlienVault OTX (Open Threat Exchange)**.

A lo largo del proyecto se simula un escenario real de administración de sistemas y respuesta a incidentes, solventando errores críticos de desincronización de red por DHCP, desconfiguración de rutas de certificados TLS/SSL, y cuellos de botella de memoria RAM (Java Virtual Machine). Finalmente, se valida el funcionamiento del pipeline de detección mediante la simulación de un ataque de fuerza bruta SSH utilizando una dirección IP catalogada como maliciosa a nivel global.

---

## 2. Arquitectura del Laboratorio y Flujo de Datos
El entorno se ha virtualizado utilizando entornos aislados con las siguientes especificaciones técnicas:

* **Wazuh Manager (All-in-One Deployment):**
    * **S.O:** Linux Debian/Ubuntu Server.
    * **Recursos:** 4GB RAM (Límite operativo optimizado mediante secuenciación de hilos).
    * **Red:** Adaptador Puente (IP dinámica asignada por router doméstico: `192.168.1.174`).
    * **Componentes:** Wazuh Indexer (Motor de búsqueda basado en OpenSearch en puerto `9200`), Wazuh Manager (Cerebro de análisis y API en puerto `55000`), Filebeat (Transporte seguro de eventos), y Wazuh Dashboard (Interfaz de usuario en puerto `443` HTTPS).
* **Endpoint Supervisado (Ubuntu-CIS):**
    * **S.O:** Ubuntu 24.04 LTS.
    * **Red:** Configurada en la misma subred interna corporativa.
    * **Componente:** `wazuh-agent` en ejecución continua, monitorizando archivos críticos del sistema.

### Flujo del Pipeline de Eventos
1. **Generación:** Un evento de seguridad ocurre en el Endpoint (Ej: intento de login SSH defectuoso).
2. **Recolección:** El `wazuh-agent` lee la línea agregada al archivo `/var/log/auth.log` en tiempo real.
3. **Transporte:** El agente cifra el paquete de telemetría y lo envía al Manager a través del puerto `1514 TCP`.
4. **Análisis y Correlación:** El motor de reglas del Manager analiza el log y cruza los datos con las listas de reputación de IP (IoCs de AlienVault OTX).
5. **Indexación:** Filebeat recoge la alerta generada y la inyecta de forma segura en el `wazuh-indexer` (puerto `9200`).
6. **Visualización:** El analista visualiza e investiga la alerta crítica de Nivel 12 (`threat_intel`) en el `wazuh-dashboard` (puerto `443`).

---

## 3. Incidentes y Troubleshooting Resueltos

### 🔹 Incidente A: Saturación de Memoria Virtual (RAM) en el arranque del Indexer
* **Síntoma:** Al levantar el laboratorio tras un estado de hibernación, la terminal se congelaba al ejecutar `sudo systemctl restart wazuh-indexer`. El servicio web (`wazuh-dashboard`) arrojaba un error de conexión rechazada (`ERR_CONNECTION_REFUSED`).
* **Análisis Técnico:** El motor de base de datos de Wazuh está programado en Java (OpenSearch/Elasticsearch), exigiendo un consumo masivo de memoria RAM. En entornos simulados de 4GB, al restaurar el estado desde el disco duro, el sistema entra en un estado transitorio de saturación donde la memoria se vuelca a la partición *Swap*, ralentizando el arranque del puerto de escucha `9200`.
* **Resolución Aplicada:** Se implementó una técnica de auditoría de red de bajo nivel mediante el comando de estadísticas de sockets para monitorizar el núcleo (kernel) de Linux en tiempo real:
    ```bash
    sudo ss -tulpn | grep 9200
    ```
    *Desglose del comando de diagnóstico:*
    * `-t` / `-u`: Filtra únicamente sockets TCP y UDP.
    * `-l`: Muestra en exclusiva servicios en estado `LISTEN` (puertos abiertos esperando conexiones).
    * `-p`: Despliega el Identificador de Proceso (PID) y nombre del binario que lo acapara (ej: `java`).
    * `-n`: Desactiva la resolución de nombres DNS para agilizar la respuesta numérica de la consola.
    * `grep 9200`: Aísla el puerto de la base de datos.
    
    *Resultado:* Se comprobó que el proceso Java se levantaba de forma limpia una vez que la memoria RAM liberaba páginas en caché (`available: 1.7Gi`), permitiendo secuenciar el encendido del Dashboard con total estabilidad.

### 🔹 Incidente B: Bucle de Reinicios en Dashboard por Disparidad de Nombres en Certificados TLS
* **Síntoma:** El servicio `wazuh-dashboard` entraba en un bucle infinito de caídas consecutivas. Al auditar los registros con el comando `sudo systemctl status wazuh-dashboard`, el contador de reinicios automáticos del sistema operativo se situaba críticamente en `122`.
* **Análisis Técnico:** Inspeccionando las últimas 50 líneas del registro del servicio mediante las herramientas nativas del diario de Linux:
    ```bash
    sudo journalctl -u wazuh-dashboard -n 50 --no-pager | grep -i "error"
    ```
    Se identificó un fallo de tipo `ENOENT` (Error No Entry / Archivo no encontrado):
    `Error: ENOENT: no such file or directory, open '/etc/wazuh-dashboard/certs/dashboard-key.pem'`
    
    Cruzando este hallazgo con el listado físico del directorio cifrado mediante `sudo ls -l /etc/wazuh-dashboard/certs/`, se descubrió que las claves criptográficas reales generadas por el instalador automatizado poseían un prefijo diferente: `wazuh-dashboard-key.pem`. La aplicación buscaba una llave inexistente para encriptar el canal HTTPS del puerto `443`.
* **Resolución Aplicada:** Se accedió a la edición del archivo de configuración maestro YAML del servidor web para sincronizar la ruta de las claves privadas:
    ```bash
    sudo nano /etc/wazuh-dashboard/opensearch_dashboards.yml
    ```
    Se modificaron las líneas conflictivas con las directrices correctas del almacén:
    ```yaml
    server.ssl.key: "/etc/wazuh-dashboard/certs/wazuh-dashboard-key.pem"
    server.ssl.certificate: "/etc/wazuh-dashboard/certs/wazuh-dashboard.pem"
    ```
    Tras guardar los cambios (`Ctrl+O`) y aplicar un reinicio forzado del servicio, la interfaz web levantó con éxito abriendo de forma inmediata el puerto SSL seguro.

### 🔹 Incidente C: Desconexión de Red del Agente debido a Reasignación Dinámica por DHCP
* **Síntoma:** Al entrar en la interfaz gráfica, el cuadro resumen de monitorización (*Agents Summary*) indicaba que el endpoint remoto se encontraba en estado **Disconnected (4)** de forma permanente.
* **Análisis Técnico:** Se auditó el archivo interno de trazas de eventos del agente en la máquina cliente (`cis-ubuntu`) mediante el comando de volcado:
    ```bash
    sudo tail -n 20 /var/ossec/logs/ossec.log
    ```
    El log destapó reintentos fallidos de conexión de red sistemáticos:
    `ERROR: (1216): Unable to connect to '192.168.1.54':1514/tcp': 'Connection refused'.`
    
    Al haber reiniciado el enrutador físico del laboratorio, el protocolo DHCP reasignó dinámicamente la dirección IP del servidor Wazuh Manager de la antigua `.54` a la nueva `.174`. El agente remoto estaba enviando toda la telemetría de seguridad a un destino vacío en la red.
* **Resolución Aplicada:** Se accedió a la configuración del agente local (`sudo nano /var/ossec/etc/ossec.conf`) y se reconfiguró el bloque de red directriz:
    ```xml
    <server>
      <address>192.168.1.174</address>
      <port>1514</port>
    </server>
    ```
    Se ejecutó un `sudo systemctl restart wazuh-agent`, logrando que la máquina pasara instantáneamente al estado **Active (1)** en la consola del analista.

---

## 4. Simulación de Ataque e Inyección de Telemetría (Explotación)
Con el pipeline de detección completamente operativo, se procedió a validar de manera táctica la efectividad de las directivas de Inteligencia de Amenazas (CTI) cargadas en el SIEM.

### Paso 1: Inyección directa en el Vector de Autenticación
Dado que los sistemas operativos modernos con `systemd-journald` tienden a canalizar los comandos `logger` estándar fuera de los ficheros planos vigilados por Wazuh, se utilizó una canalización directa forzada (`echo` + `tee`) para añadir una traza de ataque de fuerza bruta SSH idéntica a una real al final del archivo de auditoría del sistema operativo:

```bash
echo "$(date '+%b %d %H:%M:%S') cis-ubuntu sshd[1234]: Failed password for root from 23.137.105.75 port 22 ssh2" | sudo tee -a /var/log/auth.log

<img width="1600" height="793" alt="image" src="https://github.com/user-attachments/assets/4718301f-8609-424f-89fd-54cfc00665b7" />
