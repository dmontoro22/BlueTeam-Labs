# Laboratorio SOC L1: Análisis de Tráfico y Extracción de Credenciales en Claro

## Objetivo
Capturar e inspeccionar tráfico de red para identificar y extraer credenciales expuestas debido al uso de protocolos de autenticación no seguros (HTTP Basic Auth).

## Preparación del Entorno
Para llevar a cabo la captura y el análisis posterior, se instalaron las herramientas de inspección de red en el entorno de pruebas aislado:

```bash
sudo apt-get update && sudo apt-get install tcpdump tshark
```

## Captura de Red y Simulación
Se configuró un sniffer para capturar exclusivamente el tráfico web (puerto 80) y almacenarlo en un archivo PCAP. Simultáneamente, se simuló un inicio de sesión de un usuario hacia un portal sin cifrado SSL/TLS.

```bash
# Terminal 1 (Escucha)
sudo tcpdump -i any port 80 -w captura_credenciales.pcap

# Terminal 2 (Simulación del login)
curl -u analista_soc:ClaveComprometida2026 [http://neverssl.com](http://neverssl.com)
```

<img width="973" height="131" alt="image" src="https://github.com/user-attachments/assets/7218f394-3096-4e33-8a0f-c73693c6b2d0" />


## Análisis Forense (Triage L1)
Tras detener la captura, se procedió a analizar el archivo PCAP aislando las peticiones HTTP y buscando cabeceras de autorización. 

```bash
tshark -r captura_credenciales.pcap -Y "http.request" -V | grep -i "Authorization"
```

El análisis reveló la exposición de la cabecera `Authorization: Basic YW5hbGlzdGFfc29jOkNsYXZlQ29tcHJvbWV0aWRhMjAyNg==`. 

## Confirmación del Incidente
Al identificar que la autenticación se transmitió codificada en Base64 en lugar de estar cifrada, se procedió a su decodificación para confirmar la fuga de información:

```bash
echo "YW5hbGlzdGFfc29jOkNsYXZlQ29tcHJvbWV0aWRhMjAyNg==" | base64 -d
```
**Resultado:** Se extrajeron con éxito las credenciales comprometidas (`analista_soc:ClaveComprometida2026`), confirmando la vulnerabilidad en la transmisión.

<img width="1080" height="93" alt="image" src="https://github.com/user-attachments/assets/940a9771-d729-4143-8e45-949f8d497f03" />
