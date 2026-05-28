# Laboratorio 3: Resiliencia, Continuidad de Negocio y Recuperación ante Desastres (DRP) con Restic

## 1. Introducción y Objetivos Técnicos
En las operaciones modernas de **Blue Team** y **SecOps**, se parte de la premisa de que la prevención perimetral absoluta no existe: tarde o temprano, se producirá un incidente de seguridad o un compromiso de credenciales o vectores avanzados como el ransomware. La verdadera resiliencia de una infraestructura radica en su capacidad para mitigar el impacto, preservar la integridad de los activos de información y restaurar los servicios críticos en el menor tiempo posible.

Este laboratorio documenta la implementación de una **estrategia de continuidad de negocio y recuperación ante desastres (Disaster Recovery Plan - DRP)** mediante una arquitectura descentralizada Cliente-Servidor utilizando **Restic**. 

### Objetivos del Laboratorio:
* **Separación de Funciones (Segregación de Roles):** Desplegar un servidor de almacenamiento aislado (`Backup-Server`) dedicado exclusivamente a la persistencia inmutable de copias de seguridad.
* **Seguridad Criptográfica de Extremo a Extremo:** Configurar cifrado asimétrico en tránsito mediante túneles **SFTP** alimentados por llaves de curva elíptica (`ED25519`) y cifrado en reposo simétrico estándar **AES-256 en modo CTR** combinado con autenticación **Poly1305**.
* **Eficiencia de Almacenamiento (Deduplicación):** Evaluar el impacto de la deduplicación de datos a bajo nivel para optimizar costes de almacenamiento y ancho de banda en la red corporativa.
* **Automatización Desatendida:** Desarrollar scripts en Bash parametrizados e integrarlos con el demonio de planificación del sistema (`cron`) bajo un estricto control de privilegios de acceso (permisos octales).
* **Validación del RTO (Recovery Time Objective):** Ejecutar un simulacro real de destrucción de datos simulando un ataque de denegación de disponibilidad (Ransomware/Borrado accidental) y cuantificar los tiempos de restauración.

---

## 2. Arquitectura del Entorno y Flujo de Datos
El entorno se compone de dos máquinas virtuales parametrizadas dentro de un segmento de red seguro y aislado:

1.  **Cliente de Producción (`hardening-labs`):** Servidor fortificado que ejecuta los servicios operativos de la empresa y contiene los directorios críticos de datos. Realiza todo el cómputo de compresión, deduplicación y cifrado.
2.  **Servidor de Almacenamiento (`Backup-Server`):** Máquina virtual minimalista (1 vCPU, 1-2 GB RAM) configurada sin servicios expuestos a internet, cuya única función es la de persistir los bloques de datos cifrados provistos por el cliente a través del protocolo seguro **SFTP**.

```text
[ Servidor Cliente: hardening-labs ] 
        (Datos Originales)
                │
                ▼ [Cómputo Local: Deduplicación + Cifrado AES-256-CTR]
                │
        (Bloques Cifrados)
                │
                ▼ Envíos mediante SFTP (Puerto SSH Personalizado)
                │
[ Servidor Remoto: Backup-Server ] -> Almacenamiento Inmutable (/mnt/backups/)
```

---

## 3. Fase 1: Preparación y Fortificación del Servidor de Backups
Para cumplir con las mejores prácticas de endurecimiento de sistemas, las copias de seguridad jamás deben guardarse bajo el usuario administrador `root` ni permitir que un compromiso del cliente comprometa el sistema operativo del servidor de copias.

### 1.1. Creación del usuario de servicio restringido
Ejecutado en el **`Backup-Server`**:
```bash
sudo adduser restic_user
```
* **Justificación:** `adduser` genera un entorno aislado para un usuario sin privilegios de sudo, asignándole un directorio de trabajo (`/home/restic_user`) y configurando su shell nativa.

### 1.2. Aislamiento y aprovisionamiento del espacio de almacenamiento
Ejecutado en el **`Backup-Server`**:
```bash
sudo mkdir -p /mnt/backups/restic_repo
sudo chown -R restic_user:restic_user /mnt/backups/
sudo chmod 700 /mnt/backups/restic_repo
```
* **`mkdir -p`**: Genera la ruta jerárquica completa de forma recursiva asegurando que si los directorios padres no existen, se inicialicen correctamente.
* **`chown -R`**: Traspasa de forma recursiva la propiedad y el grupo de control de `/mnt/backups/` al usuario sin privilegios `restic_user`.
* **`chmod 700`**: Aplica una máscara octal estricta. Otorga permisos de lectura, escritura y ejecución (`7`) únicamente al propietario (`restic_user`), y revoca de forma absoluta (`00`) cualquier visibilidad o interacción a grupos u otros usuarios locales de la máquina.

---

## 4. Fase 2: Establecimiento de Confianza e Interconexión SSH Segura
Para posibilitar la ejecución automática y desatendida de copias sin requerir intervención humana (introducción manual de contraseñas de red), se implementa una relación de confianza basada en criptografía asimétrica.

### 2.1. Generación del par de claves criptográficas
Ejecutado en el cliente **`hardening-labs`**:
```bash
ssh-keygen -t ed25519 -C "restic_automation"
```
* **`-t ed25519`**: Especifica el uso del algoritmo de firma digital sobre la Curva de Edwards de 25519. Ofrece un rendimiento de computación drásticamente superior y una mayor resistencia criptográfica contra ataques que las claves tradicionales RSA de 4096 bits.
* **`-C "restic_automation"`**: Inyecta un comentario metadato al final de la clave pública para facilitar la auditoría de accesos dentro del archivo del servidor de destino.

### 2.2. Transferencia de la identidad pública
Ejecutado en el cliente **`hardening-labs`**:
```bash
ssh-copy-id restic_user@192.168.1.78
```
* **`ssh-copy-id`**: Automatiza la conexión remota, autentica mediante contraseña por última vez e inyecta la clave pública del cliente dentro del archivo `~/.ssh/authorized_keys` de `restic_user` en el servidor de backups, ajustando automáticamente los permisos del sistema de archivos (`chmod 600`).

---

## 5. Fase 3: Inicialización y Ciclo de Copias de Seguridad Manuales
Con las dependencias de red resueltas, se despliega la herramienta cliente en el servidor de producción para interactuar con la bóveda criptográfica a través de SFTP.

### 3.1. Despliegue de la suite Restic en el Cliente
Ejecutado en el cliente **`hardening-labs`**:
```bash
sudo apt update && sudo apt install restic -y
```

### 3.2. Inicialización de la Bóveda Cifrada Remota
Ejecutado en el cliente **`hardening-labs`**:
```bash
restic -r sftp:restic_user@192.168.1.78:/mnt/backups/restic_repo init
```
* **`-r sftp:...`**: Define el repositorio remoto apuntando al protocolo SFTP, el usuario mapeado, la IP del servidor de copias y la ruta absoluta restringida creada en la Fase 1.
* **`init`**: Instrucción que genera la estructura lógica de Restic (carpetas de índices, claves, snapshots y datos) y solicita una frase de paso maestra. Esta clave deriva la clave de cifrado simétrico **AES-256** usada para encriptar los archivos y los metadatos de forma local antes de subirlos a la red.

### 3.3. Creación del entorno de datos y ejecución del primer backup
Ejecutado en el cliente **`hardening-labs`**:
```bash
# Generación de directorio corporativo y una base de datos ficticia de 50MB
mkdir -p ~/datos_empresa
dd if=/dev/urandom of=~/datos_empresa/base_datos_falsa.db bs=1M count=50

# Ejecución del snapshot inicial
restic -r sftp:restic_user@192.168.1.78:/mnt/backups/restic_repo backup ~/datos_empresa
```
* **`dd if=/dev/urandom ...`**: Herramienta de copia a nivel binario. Extrae entropía pura del kernel (`/dev/urandom`) para crear un archivo de 50 Megabytes no indexable ni comprimible estáticamente, garantizando un entorno de prueba realista.
* **`backup`**: Ordena a Restic procesar el contenido de la ruta indicada. El sistema devuelve el ID único inmutable asociado a la instantánea.

<img width="1061" height="883" alt="image" src="https://github.com/user-attachments/assets/5fe32c17-588f-4c5b-b2df-5e3292cde893" />


---

## 6. Fase 4: Simulación de Desastre y Plan de Recuperación (DRP)
Para auditar la viabilidad de la estrategia frente a incidentes de Ransomware o sabotaje interno, se realiza una prueba destructiva controlada en caliente.

### 4.1. Escenario de Compromiso (Borrado Absoluto)
Ejecutado en el cliente **`hardening-labs`**:
```bash
rm -rf ~/datos_empresa
ls -la ~/datos_empresa
```
* **`rm -rf`**: Comando que elimina de manera directa, forzada (`-f`) y recursiva (`-r`) el directorio completo. La consulta subsiguiente devuelve el código de error `No such file or directory`, confirmando la pérdida total de disponibilidad del activo.

### 4.2. Activación del Protocolo de Restauración ante Emergencias
Ejecutado en el cliente **`hardening-labs`**:
```bash
restic -r sftp:restic_user@192.168.1.78:/mnt/backups/restic_repo restore latest --target /
```
* **`restore latest`**: Ordena la recuperación inmediata utilizando la última captura temporal válida registrada en los índices cifrados del repositorio remoto, minimizando el RPO (Recovery Point Objective).
* **`--target /`**: Al utilizar la raíz del sistema operativo como destino, Restic lee los metadatos almacenados de la instantánea original y reconstruye el árbol completo de dependencias de archivos (`/home/dani-lab/datos_empresa`) manteniendo los permisos, propietarios y marcas temporales originales sin alterar otras carpetas del sistema.

### 4.3. Verificación de Integridad Técnica
```bash
ls -lh ~/datos_empresa
```
* **Métrica de RTO Lograda:** El sistema completa la descarga, descifrado bajo demanda y posicionamiento de los 50MB a través de la red local en un tiempo récord de **1 segundo** (`Restored 4 files/dirs (50.000 MiB) in 0:01`), validando la viabilidad de los SLAs corporativos.

---

## 7. Fase 5: Automatización del Ciclo de Vida y Persistencia Desatendida
Un modelo resiliente requiere eliminar el factor humano del mantenimiento diario para asegurar la consistencia temporal de los puntos de restauración.

### 5.1. Almacenamiento seguro del secreto criptográfico
Ejecutado en el cliente **`hardening-labs`**:
```bash
echo "TU_CONTRASEÑA_MAESTRA" > ~/.restic_pass
chmod 600 ~/.restic_pass
```
* **`~/.restic_pass`**: Archivo oculto que contiene la cadena string de la clave maestra del repositorio.
* **`chmod 600`**: Establece lectura y escritura exclusivamente para el propietario (`6`) y deniega cualquier interacción (`00`) al resto de entidades del sistema, blindando el secreto contra lecturas no deseadas por otros servicios.

### 5.2. Desarrollo del Script de Producción (`backup_diario.sh`)
Ubicación: `/home/dani-lab/backup_diario.sh`
```bash
#!/bin/bash

# ==============================================================================
# LAB 3: SCRIPT DE AUTOMATIZACIÓN DE BACKUPS CIFRADOS (RESTIC + SFTP)
# ==============================================================================

# Definición estricta de variables de entorno
REPO="sftp:restic_user@192.168.1.78:/mnt/backups/restic_repo"
PASS_FILE="/home/dani-lab/.restic_pass"
TARGET_DIR="/home/dani-lab/datos_empresa"

# Ejecución del Backup inyectando credenciales por archivo confidencial
restic -r $REPO --password-file $PASS_FILE backup $TARGET_DIR
```

Otorgar privilegios de ejecución binaria al script:
```bash
chmod +x ~/backup_diario.sh
```

### 5.3. Planificación en el Demonio del Sistema (Cron Engine)
Acceso al orquestador de tareas del kernel:
```bash
crontab -e
```

Inyección de la línea de persistencia en la base de datos de Cron:
```text
0 3 * * * /home/dani-lab/backup_diario.sh >> /home/dani-lab/backup_log.txt 2>&1
```
* **`0 3 * * *`**: Mapeo temporal estricto (Minuto 0, Hora 3, Todos los días, Todos los meses, Todos los días de la semana). Forza la ejecución de la política de copias a las **03:00 AM**, minimizando el impacto de rendimiento sobre el servidor de producción al ejecutarse fuera del horario laboral.
* **`>> /home/dani-lab/backup_log.txt`**: Redirecciona la salida estándar de ejecución (`stdout`) acumulando de forma cronológica los registros de Restic en un log histórico para su posterior auditoría.
* **`2>&1`**: Aplica redirección de flujos. Fusiona la salida de errores del sistema (`stderr` - canal 2) dentro del canal de salida de texto estándar (canal 1), asegurando que si ocurre un fallo de red o autenticación, quede plasmado detalladamente dentro del archivo log de auditoría de SecOps.

<img width="1156" height="1297" alt="image" src="https://github.com/user-attachments/assets/69adef13-a4da-44f5-b087-ee8c93ab94e0" />


---

## 8. Conclusiones y Métricas Blue Team
Tras la conclusión del Laboratorio 3, la infraestructura cuenta con una capa de resiliencia avanzada que complementa el endurecimiento del sistema (Lab 1) y el despliegue del IDS/SIEM (Lab 2). 

### Indicadores Clave obtenidos:
* **Cifrado Nativo:** Garantiza la confidencialidad de la información. Si un atacante compromete físicamente el `Backup-Server`, solo obtendrá bloques de datos ininteligibles cifrados bajo **AES-256-CTR**.
* **Eficiencia por Deduplicación:** Como se evidenció en la automatización, las tareas consecutivas analizan los cambios binarios y solo transmiten los diferenciales lógicos. En entornos reales, esto reduce los requerimientos de almacenamiento hasta en un 70%.
* **Control del RTO/RPO:** La configuración del cron asegura un RPO máximo de 24 horas, mientras que el motor de restauración de Restic garantiza un RTO mínimo gracias al ensamblado veloz de snapshots binarios directos.
