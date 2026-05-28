# Laboratorio SOC L1: Monitorización de Archivos Críticos con Auditd

## Objetivo
Desplegar una regla de monitorización activa sobre `/etc/shadow` para detectar intentos de modificación o escalada de privilegios, y analizar la telemetría resultante.

## Configuración de la Regla
Se implementó la siguiente regla temporal en memoria para vigilar accesos de escritura o adición:
`sudo auditctl -w /etc/shadow -p wa -k alerta_shadow`

## Simulación del Incidente
Se simuló un intento de inyección de un usuario falso con privilegios de root mediante una redirección desde un usuario sin privilegios:
`sudo echo "usuario_falso:x:0:0::/root:/bin/bash" >> /etc/shadow`

## Análisis del Log (Triage L1)
El sistema bloqueó preventivamente la acción (Permission denied), pero `auditd` generó la alerta correspondiente. Extracción de campos clave usando `ausearch -k alerta_shadow`:

* **Timestamp:** Thu May 28 12:08:13 2026
* **Target:** `name="/etc/shadow"`
* **Veredicto del SO:** Bloqueado (`success=no exit=-13`)
* **Vector de ejecución:** `exe="/usr/bin/bash"`
* **Atribución:** Origen del intento trazado al usuario con ID 1000 (`auid=1000`), a pesar del uso parcial de sudo en la cadena de comandos.

<img width="1112" height="595" alt="image" src="https://github.com/user-attachments/assets/44d0659e-b8e3-4c64-8b0a-4871ab4fc408" />
