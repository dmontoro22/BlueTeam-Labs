# Laboratorio Antiforensics Detection: Despliegue de SIEM Splunk y Detección de Amenazas

Este repositorio contiene la documentación y configuración de un entorno de laboratorio enfocado en operaciones de **Blue Team**, diseñado para centralizar la telemetría de un sistema operativo y detectar técnicas de evasión de defensas (evasión de logs).

---

## Descripción del Entorno
* **SIEM:** Splunk Enterprise (Despliegue local en entorno de pruebas).
* **Sistema Operativo Auditado:** Windows 11 (Máquina Virtual en entorno controlado).
* **Objetivo:** Capturar e indexar eventos críticos del sistema y diseñar paneles de monitorización para el analista de seguridad (SOC).

---

## Desarrollo del Laboratorio

### Paso 1: Configuración de la Ingesta de Datos (Data Input)
Para alimentar el SIEM con telemetría relevante de ciberseguridad sin saturar las capacidades del indexador, se configuró la recolección de eventos locales de Windows (`Local event log collection`) seleccionando exclusivamente los tres canales principales:
* **Application**
* **Security**
* **System**

### Paso 2: Simulación del Escenario de Ataque (Anti-Forensics)
Para validar la efectividad de la monitorización, se simuló una técnica habitual de los atacantes para ocultar huellas tras comprometer un sistema: **el vaciado absoluto del registro de auditoría de seguridad** mediante el Visor de Eventos de Windows (*Clear Log*).

### Paso 3: Caza de Amenazas con SPL (Search Processing Language)
Una vez ejecutado el sabotaje, se desarrolló una query en la interfaz de búsqueda de Splunk para localizar de manera inequívoca la firma del borrado. 

Para ello, filtramos específicamente por el **Event ID 1102**, que es el código estandarizado por Microsoft para registrar que el log de auditoría de seguridad fue vaciado:

```splunk
source="WinEventLog:Security" EventCode=1102
```
<img width="1600" height="662" alt="image" src="https://github.com/user-attachments/assets/e8440eb7-30df-4360-85b6-e4d9da08bcb7" />


---

## Optimización de la Visualización (Filtrado de Columnas)
Splunk añade por defecto campos técnicos internos que saturan la visualización de la tabla para el analista. Para limpiar la vista y presentar únicamente la información útil, aplicamos una canalización (pipe) utilizando el comando `fields -` para eliminar las columnas innecesarias (como `_bkt` o `_cd`)

Fragmento de código

```splunk
source="WinEventLog:Security" EventCode=1102 | fields - bkt, _cd, _raw
```

---

# Dashboard de Seguridad en Tiempo Real
A partir de la consulta optimizada en SPL, se diseñó un panel analítico (Dashboard Studio) en formato cuadrícula (Grid). Este panel actúa como un monitor crítico en el SOC: si un atacante intenta mitigar las evidencias borrando registros, el evento se refleja de manera permanente y visual en la pantalla del analista para agilizar la respuesta ante incidentes.
<img width="1600" height="664" alt="image" src="https://github.com/user-attachments/assets/e2c92518-91fc-429f-88ad-53ecab55133f" />
