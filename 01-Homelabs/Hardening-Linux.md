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
