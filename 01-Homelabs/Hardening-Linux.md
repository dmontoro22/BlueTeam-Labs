# Laboratorio de Hardening y Bastionado en Sistemas Linux

Este proyecto documenta el proceso de endurecimiento (hardening) aplicado a un servidor Linux para mitigar vectores de ataque comunes, asegurar la gestión de accesos e implementar directrices de seguridad sólidas en el sistema operativo.

---

## Índice

1. **Descripción del Entorno y Objetivos**
   * Especificaciones técnicas del sistema base.
   * Alcance del bastionado.

2. **Gestión de Accesos y Principio de Mínimo Privilegio**
   * Creación de usuarios con privilegios limitados.
   * Configuración de acceso seguro a privilegios elevados (`sudo`).
   * Desactivación del inicio de sesión directo como `root`.

3. **Securización del Servicio SSH (OpenSSH)**
   * Generación e implementación de pares de claves criptográficas.
   * Deshabilitación de la autenticación por contraseña.
   * Modificación del puerto por defecto y optimización del archivo `sshd_config`.

4. **Control de Tráfico de Red (Firewall con UFW)**
   * Configuración de la política por defecto (Denegar todo el tráfico entrante).
   * Definición de reglas específicas para servicios permitidos.
   * Activación y verificación del estado del firewall.

5. **Protección Activa contra Fuerza Bruta (Fail2Ban)**
   * Instalación y arquitectura del servicio.
   * Configuración de cárceles (`jails`) específicas para OpenSSH.
   * Verificación de la persistencia del bloqueo de IPs sospechosas.

6. **Auditoría y Verificación Final**
   * Pruebas de acceso cruzado para comprobar las restricciones.
   * Listado de comandos de comprobación del estado de los servicios.

---

## 1. Descripción del Entorno y Objetivos

El presente laboratorio se ejecuta sobre una máquina virtual aislada con el propósito de aplicar técnicas de bastionado defensivo.

* **Sistema Operativo Base:** Ubuntu Server 24.04 LTS (sin entorno gráfico).
* **Rol:** Servidor de administración (Infraestructura simulada).
* **Objetivo:** Reducir drásticamente la superficie de ataque mitigando vectores de entrada comunes, asegurar la gestión de credenciales e implementar segmentación de red a nivel de *host*.

## 2. Gestión de Accesos y Principio de Mínimo Privilegio

El primer paso del bastionado consiste en asegurar que el acceso administrativo al servidor esté estrictamente controlado y auditado. 

### 2.1. Creación de Usuarios Limitados
Se ha implementado el principio de mínimo privilegio creando un usuario estándar (`auditor`) dedicado exclusivamente a tareas de revisión o lectura, sin capacidad inherente para ejecutar comandos de sistema.

\`\`\`bash
sudo adduser auditor
\`\`\`
<img width="675" height="403" alt="image" src="https://github.com/user-attachments/assets/de5537fc-93ca-43c2-a283-3b81da819638" />

### 2.2. Bloqueo de la Cuenta Root
Para prevenir ataques de fuerza bruta dirigidos al superusuario y forzar la auditoría (obligando a los administradores a usar `sudo` con sus cuentas nominales), se ha bloqueado el acceso directo a la cuenta `root` en el sistema. El modificador `-l` bloquea (lock) la contraseña asignada.

\`\`\`bash
sudo passwd -l root
\`\`\`
*(Resultado esperado: El sistema confirma el bloqueo modificando la caducidad o el estado de la contraseña).*
<img width="427" height="46" alt="image" src="https://github.com/user-attachments/assets/05004429-9b49-4d49-b4c0-f204a353b819" />

## 3. Securización del Servicio SSH (OpenSSH)

El acceso remoto mediante SSH es uno de los vectores de ataque más críticos. Para mitigarlo, se ha reconfigurado el servicio `/etc/ssh/sshd_config` abandonando la autenticación basada en contraseñas en favor de la criptografía de curva elíptica.

### 3.1. Implementación de Claves Ed25519
Se ha generado e importado un par de claves criptográficas utilizando el algoritmo `Ed25519`, conocido por su alto rendimiento y resistencia a ataques de fuerza bruta. La clave pública se ha autorizado en el archivo `~/.ssh/authorized_keys` del usuario con permisos estrictos (`600` para el archivo, `700` para el directorio).
<img width="549" height="100" alt="image" src="https://github.com/user-attachments/assets/547b2f33-c6b9-482a-803e-27b9e48fe7c6" />


### 3.2. Optimización del archivo sshd_config
Se han aplicado las siguientes directivas de bastionado (Hardening) en la configuración del servicio:

* `Port 4422`: Modificación del puerto por defecto (22) para evadir el escaneo masivo automatizado y reducir el ruido en los logs.
* `PasswordAuthentication no`: Deshabilitación absoluta del acceso por contraseña (fallback), requiriendo obligatoriamente la presentación de la clave privada autorizada.
<img width="1281" height="860" alt="image" src="https://github.com/user-attachments/assets/15b9a8e7-54ad-496f-9dd8-f194c078a96e" />

## 4. Control de Tráfico de Red (Firewall con UFW)

Para reducir la exposición de red del servidor, se ha implementado el cortafuegos UFW (Uncomplicated Firewall) aplicando la política de seguridad de "Denegación por defecto" (Default Deny).

### 4.1. Configuración de Políticas Base
Se ha bloqueado de forma predeterminada todo el tráfico entrante no solicitado (`deny incoming`), permitiendo el tráfico saliente (`allow outgoing`) para mantener la capacidad del servidor de establecer conexiones legítimas (actualizaciones, resolución DNS, etc.).

### 4.2. Implementación de Reglas Estrictas
La única excepción configurada en la tabla de filtrado es la apertura del puerto SSH modificado, restringiendo el acceso exclusivamente al protocolo TCP.

Estado final de la configuración (`sudo ufw status numbered`):
```text
To                         Action      From
--                         ------      ----
[ 1] 4422/tcp              ALLOW IN    Anywhere
[ 2] 4422/tcp (v6)         ALLOW IN    Anywhere (v6) 
```
<img width="723" height="393" alt="image" src="https://github.com/user-attachments/assets/d27fc6ef-7e87-434a-99c9-5919de66868f" />

 ## 5. Protección Activa contra Fuerza Bruta (Fail2Ban)

Para complementar las defensas perimetrales estáticas, se ha desplegado Fail2Ban como medida de intrusión activa. Este servicio analiza los registros de autenticación en tiempo real y mitiga los ataques de fuerza bruta insertando dinámicamente reglas de denegación en el cortafuegos UFW.

### 5.1. Definición de Políticas (Jail)
Siguiendo las mejores prácticas de administración, se ha configurado una política local (`/etc/fail2ban/jail.local`) para evitar la sobrescritura durante las actualizaciones del paquete principal.
<img width="712" height="193" alt="image" src="https://github.com/user-attachments/assets/4cb89c37-2d2e-41b8-83e9-08c9b9603acc" />


Se han definido los siguientes parámetros estrictos para el servicio SSH:
* `port = 4422`: Sincronizado con el puerto modificado en la fase anterior.
* `maxretry = 3`: Umbral máximo de intentos de autenticación fallidos.
* `bantime = 3600`: Tiempo de penalización (1 hora) durante el cual el tráfico de la IP atacante será descartado (DROP).

Estado operativo verificado mediante la instrucción: `sudo fail2ban-client status sshd`.
<img width="586" height="271" alt="image" src="https://github.com/user-attachments/assets/0c38b6cd-3df2-4fa1-b570-9d3de5e89639" />

## 6. Auditoría y Verificación Final

Para validar la efectividad de las políticas de bastionado implementadas, se define el siguiente protocolo de auditoría, diseñado para comprobar tanto el control de accesos como el estado de los servicios defensivos.

### 6.1. Pruebas de Acceso Cruzado y Restricciones
* **Auditoría de Mínimo Privilegio:** Al cambiar la sesión al usuario estándar (`su - auditor`) y ejecutar acciones de escalada de privilegios (ej. `sudo cat /etc/shadow`), el sistema bloquea la acción y registra un evento de seguridad (`not in the sudoers file`), confirmando la restricción.
  <img width="429" height="161" alt="image" src="https://github.com/user-attachments/assets/5eb7d820-fac1-404a-9d56-fd615ff82e21" />
* **Auditoría de Autenticación SSH:** Cualquier intento de conexión sin la correspondiente clave privada Ed25519 es rechazado de forma silenciosa por el servidor, impidiendo el fallback a la autenticación por contraseña.
* **Auditoría IPS (Fail2Ban):** La simulación de intentos fallidos reiterados desencadena la inserción de una regla `REJECT` en UFW para la IP origen, validando la defensa activa.
  <img width="748" height="141" alt="image" src="https://github.com/user-attachments/assets/711a722e-9b99-4744-ae3a-86ea12878f24" />

### 6.2. Listado de Comandos de Comprobación de Estado
Estos comandos permiten al administrador verificar el estado de la infraestructura de seguridad en cualquier momento:

1. **Estado del servicio SSH y puerto de escucha:**
   \`\`\`bash
   sudo systemctl status ssh
   ss -tulpn | grep sshd
   \`\`\`
   *(Verifica que el servicio esté `active (running)` y escuchando en el puerto modificado `*:4422`).*

2. **Estado de las reglas de filtrado de red (UFW):**
   \`\`\`bash
   sudo ufw status numbered
   \`\`\`
   *(Audita que solo existan reglas `ALLOW IN` explícitas para los puertos autorizados).*

3. **Estado de la monitorización contra intrusiones (Fail2Ban):**
   \`\`\`bash
   sudo fail2ban-client status sshd
   \`\`\`
   *(Muestra el número de IPs actualmente bloqueadas y el total de intentos fallidos).*







