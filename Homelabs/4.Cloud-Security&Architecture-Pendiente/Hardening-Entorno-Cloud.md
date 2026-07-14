# Laboratorio de Hardening en Servidor Linux (Ubuntu 24.04 LTS) sobre Azure

Este laboratorio práctico detalla el proceso de despliegue, configuración y endurecimiento (*hardening*) de un servidor Linux en entornos cloud (Microsoft Azure) bajo el principio de **Defensa en Profundidad**. El objetivo es reducir al mínimo la superficie de ataque del servidor frente a amenazas externas, aplicando controles de seguridad tanto a nivel perimetral como internos del sistema operativo.

---

## Arquitectura y Componentes del Laboratorio

*   **Proveedor de Cloud:** Microsoft Azure
*   **Región de Despliegue:** Spain Central
*   **Sistema Operativo:** Ubuntu Server 24.04 LTS
*   **Segmentación de Red:** 
    *   Red Virtual (VNet): `vnet-lab-hardening` (Rango: `10.0.0.0/16`)
    *   Subred: `snet-servers` (Rango: `10.0.0.0/24`)
*   **Filtrado Perimetral:** Grupo de Seguridad de Red (NSG) configurado para permitir tráfico entrante en el puerto SSH (22) **únicamente** desde la IP pública del administrador.
<img width="1917" height="874" alt="Captura de pantalla 2026-07-14 114941" src="https://github.com/user-attachments/assets/47605dcc-0188-42ce-9520-6304af0bf24b" />
<img width="1916" height="905" alt="Captura de pantalla 2026-07-14 115419" src="https://github.com/user-attachments/assets/47012b9d-fb63-40b0-9c32-ac998d4781c6" />


---

## Fase 1: Actualización y Gestión de Vulnerabilidades

El primer paso para mitigar el riesgo de explotación de vulnerabilidades conocidas (CVEs) es asegurar que el software del sistema operativo se encuentre en su última versión estable y con los parches de seguridad aplicados.

### 1. Actualización inicial de paquetes
```bash
sudo apt update && sudo apt upgrade -y
```

### 2. Automatización de parches de seguridad (Actualizaciones Desatendidas)
Para evitar que el servidor quede expuesto a nuevas vulnerabilidades sin parchear por falta de mantenimiento manual, se implementa el servicio de actualizaciones automáticas:

```bash
# Instalar los paquetes necesarios
sudo apt install unattended-upgrades apt-listchanges -y

# Activar y configurar el servicio interactivo
sudo dpkg-reconfigure -plow unattended-upgrades
```

---

## Fase 2: Robustecimiento del Servicio SSH (SSH Hardening)

Dado que SSH es el principal vector de administración remota, se securiza el demonio `sshd` aplicando directivas alineadas con los estándares de la industria (CIS Benchmarks). 

En lugar de modificar el archivo principal `/etc/ssh/sshd_config`, se genera un archivo de configuración específico en el directorio drop-in `.d` para garantizar modularidad y limpieza:

```bash
sudo nano /etc/ssh/sshd_config.d/99-hardening.conf
```

Se añaden las siguientes directivas de seguridad:

```text
# =============== SSH HARDENING ===============
# Desactivar el acceso directo al usuario administrador "root"
PermitRootLogin no

# Forzar el uso exclusivo de llaves criptográficas (prohibir autenticación por contraseña)
PasswordAuthentication no
PubkeyAuthentication yes

# Limitar los intentos fallidos de autenticación por conexión para mitigar fuerza bruta
MaxAuthTries 3

# Desactivar el reenvío de entorno gráfico (reduce la superficie de ataque de X11)
X11Forwarding no

# Desconexión automática por inactividad (5 minutos = 300 segundos)
ClientAliveInterval 300
ClientAliveCountMax 0
```

### Verificación de sintaxis previa al reinicio
Antes de aplicar la configuración, se comprueba la sintaxis para evitar pérdidas de acceso accidentales:

```bash
# Creación del directorio temporal de privilegios (requerido para tests manuales en Ubuntu moderno)
sudo mkdir -p /run/sshd

# Testeo de configuración (debe ejecutarse de manera silenciosa)
sudo sshd -t

# Reinicio seguro del servicio
sudo systemctl restart ssh
```

---

## Fase 3: Cortafuegos del Host (UFW - Uncomplicated Firewall)

Como segunda línea de defensa frente a fallos de configuración perimetrales en la nube, se despliega y activa el cortafuegos interno a nivel de sistema operativo, configurándolo bajo una política de **Denegación por Defecto** (*Default Deny*).

### 1. Establecer políticas globales
```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
```

### 2. Permitir acceso SSH exclusivo desde la IP pública autorizada
```bash
# Reemplazar por la IP pública real de administración
sudo ufw allow proto tcp from 88.148.42.123 to any port 22 comment 'SSH seguro desde casa'
```

### 3. Activar y verificar el estado del cortafuegos
```bash
sudo ufw enable
sudo ufw status verbose
```
<img width="1096" height="271" alt="Captura de pantalla 2026-07-14 133046" src="https://github.com/user-attachments/assets/658b7830-6a3c-413b-81e9-8264edc07301" />


---

## Fase 4: Endurecimiento del Kernel (Sysctl Hardening)

Se optimizan los parámetros del núcleo de Linux en tiempo de ejecución para blindar la pila de red (TCP/IP) del sistema operativo contra ataques de denegación de servicio (DoS), suplantación de identidad (IP Spoofing) y envenenamiento de tablas de enrutamiento.

Se crea un archivo de configuración de seguridad del kernel personalizado:

```bash
sudo nano /etc/sysctl.d/99-security.conf
```

Se introducen las siguientes directivas:

```text
# =============== KERNEL HARDENING ===============

# 1. Mitigar ataques de inundación de paquetes SYN (SYN Flood / DoS)
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_synack_retries = 5

# 2. Desactivar el reenvío de IP (impedir que el servidor actúe como router)
net.ipv4.ip_forward = 0

# 3. Ignorar redirecciones ICMP (prevenir ataques de redirección de tráfico o MitM)
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0

# 4. Habilitar filtro de ruta inversa (protección estricta contra IP Spoofing)
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1

# 5. Registrar en el log de sistema paquetes sospechosos o rutas imposibles (Martian Packets)
net.ipv4.conf.all.log_martians = 1
net.ipv4.conf.default.log_martians = 1
```

### Aplicación de cambios en caliente
Para aplicar los parámetros de seguridad directamente en la memoria del kernel sin necesidad de reiniciar el servidor:

```bash
sudo sysctl --system
```
<img width="1003" height="975" alt="Captura de pantalla 2026-07-14 133258" src="https://github.com/user-attachments/assets/217eceda-9f08-479c-8633-df2609f4810a" />


---

## Conclusiones del Laboratorio

La ejecución de este laboratorio demuestra la importancia de estructurar la seguridad de los sistemas de producción bajo la filosofía de **Defensa en Profundidad**. Incluso si el perímetro del proveedor de nube es vulnerado o sufre una mala configuración, las múltiples capas aplicadas internamente en el host (SSH robusto, Firewall local UFW y blindaje de la pila TCP/IP a nivel de Kernel) garantizan la resiliencia del activo y mitigan drásticamente los vectores de compromiso más comunes.
