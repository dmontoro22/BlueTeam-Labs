# Laboratorio 2: Despliegue de un Sistema de Detección de Intrusiones (IDS) con Suricata

Este laboratorio documenta la implementación, optimización y personalización de un motor de detección de amenazas en red de alto rendimiento utilizando **Suricata 8.0.5** sobre un servidor espartano **Ubuntu Server (sin interfaz gráfica)**. El objetivo es actuar como un Sensor de Red dedicado (NIDS) capaz de analizar paquetes en tiempo real, diagnosticar fallos de visibilidad y generar firmas de telemetría a medida para la detección temprana de fases de reconocimiento.

---

## 1. Arquitectura y Requisitos del Entorno

* **Rol de la Máquina Virtual:** Sensor Dedicado de Red (NIDS - Network Intrusion Detection System).
* **Sistema Operativo:** Ubuntu Server LTS (Minimal install, sin entorno gráfico para reducir la superficie de ataque y optimizar el procesamiento multihilo).
* **Recursos Asignados:** 2 vCPUs (aprovechamiento nativo del motor multihilo af-packet), 2 GB RAM.
* **Modo de Red:** Adaptador Puente (Bridged) para interactuar directamente con el segmento de la red local física.
* **Segmento de Red Protegido (`HOME_NET`):** `192.168.1.0/24`
* **Dirección IP del Sensor:** `192.168.1.60`

---

## 2. Fase de Instalación y Aprovisionamiento

Para asegurar la adquisición de la última versión estable del motor de detección directa de la OISF (Open Information Security Foundation), se añade su repositorio oficial.

```bash
sudo apt update
sudo apt install software-properties-common -y
sudo add-apt-repository ppa:oisf/suricata-stable
sudo apt update
sudo apt install suricata jq -y
```

### Glosario Técnico de Comandos Ejecutados:
* `sudo`: Permite ejecutar instrucciones con los máximos privilegios del superusuario (`root`).
* `apt update`: Sincroniza los archivos de índices de paquetes desde sus fuentes. Descarga el catálogo más reciente de software disponible sin alterar los programas instalados.
* `apt install`: Descarga e instala los paquetes indicados junto con sus dependencias necesarias.
* `software-properties-common`: Proporciona abstracciones útiles para gestionar de forma segura los repositorios de software adicionales (PPAs).
* `-y`: Responde afirmativamente de forma automática a todas las solicitudes de confirmación de descarga de espacio en disco.
* `add-apt-repository ppa:...`: Introduce una nueva fuente de confianza en el gestor de paquetes para obtener software directamente de los desarrolladores originales.
* `jq`: Procesador ligero y flexible de línea de comandos diseñado para estructurar, filtrar y dar formato de lectura humana a datos JSON.

---

## 3. Configuración del Sensor e Interfaces de Red

Suricata requiere conocer con precisión el mapa de red local y el identificador de la tarjeta que operará en modo promiscuo para la captura de tramas.

### Paso 1: Identificación de la Interfaz del Sistema
```bash
ip a
```
* **Explicación:** Muestra el estado y las direcciones IP configuradas en todas las tarjetas de red lógicas del sistema operativo.
* **Resultado obtenido en este laboratorio:** Interfaz física real identificada como `enp0s3` con la dirección de host `192.168.1.60/24`.

### Paso 2: Edición del Archivo Principal `suricata.yaml`
Se accede al archivo maestro de configuración del motor:
```bash
sudo nano /etc/suricata/suricata.yaml
```
* `nano`: Editor de texto minimalista en consola.

Dentro del documento se modifican estrictamente los siguientes parámetros espaciales, cuidando la indentación de bloques YAML (sin usar tabuladores):

1.  **Definición de la Red Protegida:**
    ```yaml
    address-groups:
      HOME_NET: "[192.168.1.0/24]"
    ```
    * *Nota Teórica:* Acotar el rango a la subred real optimiza el uso de la memoria RAM y mitiga los falsos positivos en las reglas condicionales dirigidas hacia objetivos internos.
2.  **Asignación de la Interfaz de Captura de Alto Rendimiento:**
    ```yaml
    af-packet:
      - interface: enp0s3
    ```
    * *Nota Teórica:* `af-packet` (Address Family Packet) mapea los paquetes de la tarjeta de red al espacio de memoria del software saltándose llamadas del sistema intermedias.

### Paso 3: Validación Sintáctica del Archivo Configurado
Antes de inicializar procesos, se valida que el documento YAML no contenga errores gramaticales o de formato.
```bash
sudo suricata -T -c /etc/suricata/suricata.yaml -v
```
* `-T`: Modo Test. Examina y valida la configuración y aborta el arranque real.
* `-c`: Especifica la ruta absoluta del archivo de configuración a evaluar.
* `-v`: Modo Verbose. Incrementa el detalle de los mensajes informativos en pantalla.

---

## 4. Gestión de Inteligencia de Amenazas (Reglas Públicas)

Se descarga el conjunto completo de firmas de ciberseguridad *Emerging Threats (ET Open)* y se levanta el servicio por primera vez.

```bash
sudo suricata-update
sudo systemctl restart suricata
sudo systemctl status suricata
```

### Glosario Técnico de Comandos Ejecutados:
* `suricata-update`: Script oficial que unifica, limpia y compila las firmas de amenazas actualizadas de internet en el archivo unificado `/var/lib/suricata/rules/suricata.rules`.
* `systemctl restart`: Detiene por completo la instancia activa de un servicio del sistema y la arranca de nuevo desde cero para obligar al software a cargar la nueva configuración en memoria.
* `systemctl status`: Muestra el estado operativo de salud actual de un daemon (ej. *active (running)*).

---

## 5. Simulación de Ataques y Guía de Resolución de Problemas (Troubleshooting)

### Prueba de Concepto 1: True Positive (Ataque HTTP Conocido)
Se simula una respuesta comprometida del sistema operativo mediante una petición HTTP web que devuelve comandos en texto plano:
```bash
curl http://testmynids.org/uid/index.html
```
* `curl`: Cliente de línea de comandos para transferir datos mediante protocolos de red (como HTTP). Permite simular interacciones de red sin necesidad de un navegador web.

### Bitácora de Investigación ante un "Falso Negativo" (Logs Vacíos)

Durante la primera ejecución, el log clásico `fast.log` se encontraba vacío. Como analistas de SOC, implementamos la siguiente metodología de investigación:

1.  **Auditoría de Interfaz Errónea:** Al ejecutar `sudo grep -A 3 "af-packet:" /etc/suricata/suricata.yaml`, se detectó que el sensor venía preconfigurado para escuchar de manera predeterminada en `eth0` en lugar de en la tarjeta de red real `enp0s3`. Se corrigió el archivo yaml y se ejecutó un `systemctl restart`.
2.  **Transición de Formato (Suricata 8.x):** Las versiones modernas priorizan los registros estructurados JSON frente al texto plano. Se procedió a inspeccionar el archivo maestro moderno `eve.json` utilizando filtros selectivos:
    ```bash
    sudo cat /var/log/suricata/eve.json | jq 'select(.event_type=="alert")'
    ```
    * `cat`: Imprime el contenido completo de un archivo en el flujo de la terminal.
    * `|` (Tubería/Pipe): Redirige la salida estándar del comando izquierdo directamente hacia la entrada estándar del comando derecho.
    * `jq 'select(.event_type=="alert")'`: Filtra el JSON masivo extrayendo únicamente los objetos cuyos campos internos coincidan con una alerta de seguridad.

### Resultado de la Captura en `eve.json` (Ataque Exitoso)
```json
{
  "timestamp": "2026-05-22T12:20:50.821647+0000",
  "flow_id": 772799666000617,
  "in_iface": "enp0s3",
  "event_type": "alert",
  "src_ip": "52.222.132.10",
  "src_port": 80,
  "dest_ip": "192.168.1.60",
  "dest_port": 56980,
  "proto": "TCP",
  "alert": {
    "action": "allowed",
    "signature_id": 2100498,
    "signature": "GPL ATTACK_RESPONSE id check returned root",
    "category": "Potentially Bad Traffic",
    "severity": 2
  },
  "app_proto": "http",
  "direction": "to_client"
}
```
* **Análisis del Analista:** El motor identificó una severidad media (2). La alerta saltó en el flujo de vuelta (`direction: to_client`) porque la firma detectó la cadena de texto crítica `uid=0(root)` en la carga útil (payload) TCP proveniente de la IP externa por el puerto HTTP 80.

---

## 6. Creación e Implementación de Reglas Personalizadas (Custom Signatures)

Para detectar técnicas de reconocimiento temprano como un escaneo de descubrimiento de hosts activos (Ping Sweep), se desarrolla una firma local personalizada desde cero.

### Paso 1: Creación del Archivo de Reglas Locales
```bash
sudo nano /var/lib/suricata/rules/local.rules
```
Se inserta en una sola línea la siguiente firma de detección:
```text
alert icmp any any -> $HOME_NET any (msg:"LABORATORIO - Posible Host Discovery (Ping)"; sid:1000001; rev:1;)
```

### Desglose Estructural de la Firma:
* `alert`: Acción a tomar (IDS). Genera una entrada en los registros sin bloquear el tráfico.
* `icmp`: Protocolo de capa 3/4 a inspeccionar (Internet Control Message Protocol).
* `any any`: Cualquier IP de origen y cualquier puerto de origen.
* `->`: Operador direccional de origen a destino.
* `$HOME_NET any`: Destinado hacia nuestro segmento local en cualquier puerto.
* `msg:"..."`: Cadena descriptiva de texto plano que se presentará ante el analista en el SIEM.
* `sid:1000001`: Signature ID único. **Buenas Prácticas:** Las firmas locales personalizadas deben iniciar a partir del rango numérico `1000001` para prevenir colisiones con las bases de firmas internacionales.
* `rev:1`: Historial de revisión o versión de la firma creada.

### Paso 2: Vinculación en el Archivo Maestro
Se edita `/etc/suricata/suricata.yaml` para indicarle al motor que debe procesar este nuevo archivo junto al repositorio principal de firmas:
```yaml
rule-files:
  - suricata.rules
  - local.rules
```

### Lección Aprendida de Alerta en Logs de Inicialización
Durante el despliegue, el comando `sudo grep -i "rule" /var/log/suricata/suricata.log | tail -n 5` arrojó un error de lectura (*No rule files match the pattern*).
* *Análisis y Solución:* El archivo había sido guardado accidentalmente en una ruta errónea (`/var/lib/suricata/local.rules`). Se corrigió el posicionamiento del archivo moviéndolo al directorio interno adecuado mediante el comando de administración de sistemas:
    ```bash
    sudo mv /var/lib/suricata/local.rules /var/lib/suricata/rules/
    ```
    * `mv`: Desplaza o renombra archivos y directorios dentro de la jerarquía del sistema de archivos.

Tras corregir la ruta y reiniciar con `systemctl restart suricata`, el registro confirmó la carga exitosa de las firmas personalizadas:
`Info: detect: 2 rule files processed. 50224 rules successfully loaded`

---

## 7. Fase de Evidencia de Caza y Detección en Tiempo Real

Se ejecuta un ping clásico de 4 paquetes desde la dirección IP de auditoría física (`192.168.1.42`) apuntando al servidor sensor.

A continuación, se interroga al log refinando la consulta para omitir el ruido blanco e imprevisto de decodificación de tramas de red (`signature_id != 2200121`):
```bash
sudo cat /var/log/suricata/eve.json | jq 'select(.event_type=="alert" and .alert.signature_id != 2200121)'
```

### Telemetría Recogida en el SOC:
```json
{
  "timestamp": "2026-05-22T13:39:35.436556+0000",
  "flow_id": 2156469283656912,
  "in_iface": "enp0s3",
  "event_type": "alert",
  "src_ip": "192.168.1.42",
  "dest_ip": "192.168.1.60",
  "proto": "ICMP",
  "icmp_type": 8,
  "icmp_code": 0,
  "alert": {
    "action": "allowed",
    "signature_id": 1000001,
    "rev": 1,
    "signature": "LABORATORIO - Posible Host Discovery (Ping)",
    "severity": 3
  },
  "direction": "to_server"
}
```

### Conclusiones del Analista de Seguridad:
La regla personalizada se ejecutó con éxito detectando el paquete ICMP.
1.  La telemetría indica de forma explícita que la IP del origen de la sonda de escaneo es la `192.168.1.42`.
2.  El campo `"icmp_type": 8` confirma de forma inequívoca el análisis de redes: se trata de un *ICMP Echo Request* (solicitud de eco), validando que un atacante interno o una herramienta automatizada está realizando reconocimiento activo de hosts levantados en este segmento de red.

```
