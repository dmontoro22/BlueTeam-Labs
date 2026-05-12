# Hardening Inicial en Servidores Linux (Debian/Ubuntu)

## Descripción del Laboratorio
Documentación del procedimiento estándar para la securización inicial de un servidor Linux. El objetivo de este laboratorio es aplicar una línea base de seguridad en un entorno VPS para reducir la superficie de ataque, bloquear intentos de fuerza bruta y asegurar los accesos de administración.

## Entorno y Herramientas
* Sistema Operativo: Ubuntu Server 22.04 LTS
* Servicios: OpenSSH
* Seguridad perimetral: UFW (Uncomplicated Firewall)

---

## Procedimiento de Securización

### 1. Actualización base del sistema
Antes de configurar la red o exponer servicios, es imperativo aplicar los últimos parches de seguridad disponibles en los repositorios oficiales.

```bash
# Actualización del listado de repositorios y paquetes del sistema
sudo apt update && sudo apt upgrade -y

# Limpieza de paquetes obsoletos
sudo apt autoremove -y
```

### 2. Bastionado del servicio SSH
El puerto de administración es el objetivo principal de ataques automatizados. Para mitigarlo, editamos la configuración del demonio SSH (`/etc/ssh/sshd_config`) para aplicar el principio de mínimo privilegio y forzar una autenticación fuerte.

Los cambios clave en la configuración son:
* Deshabilitar el acceso directo al usuario root (`PermitRootLogin no`).
* Deshabilitar el acceso por contraseña (`PasswordAuthentication no`), forzando el uso de claves.

```bash
# Edición del archivo de configuración
sudo nano /etc/ssh/sshd_config

# Una vez modificados los parámetros, reiniciamos el servicio
sudo systemctl restart sshd
```

### 3. Configuración del Firewall (UFW)
Aplicamos una política de denegación por defecto (default deny). Esto asegura que cualquier puerto no autorizado explícitamente quede bloqueado a nivel de red.

```bash
# Bloquear todo el tráfico entrante por defecto
sudo ufw default deny incoming

# Permitir el tráfico saliente
sudo ufw default allow outgoing

# Autorizar únicamente el puerto de administración SSH
sudo ufw allow ssh

# Activar el firewall
sudo ufw enable
```

Para verificar que las reglas se han aplicado correctamente y persisten:

```bash
sudo ufw status verbose
```
