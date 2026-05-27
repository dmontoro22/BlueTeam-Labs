# Auditoría y Hardening de Servidor Linux (Kenobi)

## Resumen Ejecutivo
Este documento detalla la auditoría de seguridad realizada sobre un servidor Linux Ubuntu (Kenobi). Durante el ejercicio, se identificaron múltiples vulnerabilidades de configuración y software obsoleto que permitieron comprometer el sistema en su totalidad (Escalada a `root`). A continuación, se detalla la cadena de ataque (Kill Chain) y las contramedidas de bastionado (Hardening) necesarias para asegurar la infraestructura.

---

## Fase 1: Reconocimiento y Exposición de Servicios

Se realizó un escaneo exhaustivo de puertos utilizando Nmap (`nmap -sC -sV -p- -T4 <IP>`), revelando una superficie de ataque excesivamente amplia:

* **21/tcp (FTP):** ProFTPD 1.3.5
* **22/tcp (SSH):** OpenSSH 8.2p1
* **80/tcp (HTTP):** Apache httpd 2.4.41
* **111/tcp & 2049/tcp (RPC/NFS):** Network File System expuesto.
* **139/tcp & 445/tcp (SMB):** Samba smbd 4.6.2

### Hallazgo Crítico 1: Enumeración SMB
Mediante la herramienta `smbclient`, se descubrió que el servidor permitía **Null Sessions** (sesiones anónimas sin credenciales). Se identificó un recurso compartido llamado `anonymous` que contenía un archivo de registro (log) exponiendo que el servicio FTP se ejecutaba bajo el usuario `kenobi` y que se había generado una clave SSH.

### Hallazgo Crítico 2: Exposición NFS
La configuración de `/etc/exports` carecía de listas blancas de direcciones IP, permitiendo a cualquier equipo externo listar y montar directorios internos. Se identificó que el directorio `/var` estaba disponible para montaje remoto.

---

## Fase 2: Explotación y Exfiltración

Se identificó que la versión de ProFTPD (1.3.5) era vulnerable al exploit del módulo `mod_copy`.

1.  **Abuso de comandos SITE:** Utilizando una conexión cruda mediante `netcat` al puerto 21, se emplearon los comandos `SITE CPFR` y `SITE CPTO`.
2.  **Movimiento lateral de archivos:** Se copió la clave privada SSH del administrador desde un directorio protegido (`/home/kenobi/.ssh/id_rsa`) hacia el directorio expuesto en red (`/var/tmp/id_rsa`).
3.  **Exfiltración:** Se montó el directorio NFS en la máquina atacante local (`sudo mount <IP>:/var /mnt/kenobiNFS`) y se exfiltró la clave privada para iniciar sesión por SSH como el usuario `kenobi`.

---

## Fase 3: Escalada de Privilegios (Path Hijacking)

Una vez dentro del sistema como usuario sin privilegios, se realizó una auditoría de binarios con permisos SUID utilizando el comando:
`find / -perm -u=s -type f 2>/dev/null`

Se identificó un binario personalizado inusual (`/usr/bin/menu`). Al analizarlo con el comando `strings`, se descubrió que el programa invocaba herramientas del sistema (como `curl`) utilizando rutas relativas en lugar de rutas absolutas.

**Vector de ataque (Path Hijacking):**
1. Se creó un archivo ejecutable falso llamado `curl` en `/tmp` que contenía una llamada a `/bin/sh`.
2. Se modificó la variable de entorno PATH (`export PATH=/tmp:$PATH`) para priorizar el directorio temporal.
3. Al ejecutar el menú (que corría como `root` por el bit SUID), el sistema ejecutó el binario falso, otorgando una *shell* interactiva con privilegios de administrador supremo (root).

---

## Medidas de Mitigación y Hardening (Blue Team)

Para securizar este servidor y evitar la cadena de ataque descrita, se deben aplicar las siguientes configuraciones de bastionado en Linux:

1.  **Securización de Samba (SMB):**
    * Deshabilitar el acceso de invitados en `smb.conf` estableciendo `guest ok = no` y `map to guest = never`.
    * Deshabilitar SMBv1 y forzar el cifrado y firmado de paquetes.
2.  **Bastionado de NFS (`/etc/exports`):**
    * Restringir el montaje de directorios únicamente a IPs específicas de la subred interna autorizada.
    * Asegurar que la opción `root_squash` esté activada para degradar los permisos de cualquier usuario root remoto.
3.  **Gestión de Vulnerabilidades y Parcheo:**
    * Actualizar inmediatamente ProFTPD a una versión segura o, preferiblemente, migrar el servicio a SFTP (sobre SSH) para unificar la superficie de ataque.
4.  **Auditoría de Permisos SUID:**
    * Implementar el uso exclusivo de **rutas absolutas** (ej. `/usr/bin/curl`) en el código fuente de cualquier script o binario personalizado.
    * Retirar el bit SUID de `/usr/bin/menu` (`chmod u-s /usr/bin/menu`) o restringir su ejecución a un grupo de administradores específico.
