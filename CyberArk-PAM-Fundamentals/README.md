# Fundamentos de Privileged Access Management (PAM) y Arquitectura CyberArk

Este módulo contiene la documentación teórica sobre la gestión de accesos privilegiados y el funcionamiento interno de la arquitectura de CyberArk, basado en los fundamentos oficiales de CyberArk University.

---

## ¿Qué es PAM (Privileged Access Management)?
La gestión de accesos privilegiados es un pilar crítico en la ciberseguridad defensiva (Blue Team). Se centra en proteger, controlar, monitorizar y auditar las cuentas con permisos elevados (como `root` en Linux o `Administrator` en Windows), las cuales representan el objetivo principal de los atacantes para realizar movimientos laterales y comprometer una infraestructura.

---

## Componentes Críticos de la Arquitectura CyberArk

### 1. Digital Vault (La Bóveda)
Es el núcleo central de la infraestructura. Consiste en una base de datos altamente aislada, endurecida (hardened) y cifrada mediante algoritmos de grado militar. Su función exclusiva es almacenar de forma segura los secretos de la organización: contraseñas de alta jerarquía, llaves privadas SSH y tokens. El acceso al Vault está restringido y se gestiona únicamente a través de protocolos propietarios del software.

### 2. CPM (Central Password Manager)
Es el componente encargado del mantenimiento automático y el ciclo de vida de las credenciales. Ejecuta las políticas de rotación de contraseñas de la empresa (por ejemplo, cambiar la clave de un servidor de producción cada 30 días con una complejidad de 20 caracteres aleatorios). El CPM se conecta de forma desatendida a los sistemas de destino, modifica la credencial y la actualiza inmediatamente dentro del Vault.

### 3. PSM (Privileged Session Manager)
Actúa como un proxy de seguridad o intermediario para el establecimiento de conexiones críticas. Cuando un operador requiere acceder a un activo, la sesión (RDP, SSH, Web) se canaliza obligatoriamente a través del PSM. Este componente aísla el entorno del usuario, inyecta la credencial de forma transparente y registra la sesión completa (tanto en formato de vídeo como en registro de comandos de texto) para su posterior auditoría en el SOC.

### 4. PVWA (Password Vault Web Access)
Es la interfaz web centralizada a través de la cual los administradores, técnicos y analistas de seguridad interactúan con el sistema para solicitar accesos, visualizar las cuentas disponibles bajo su rol y gestionar las políticas de configuración del entorno.

---

## Flujo de Conexión Seguro (Aislamiento de Credenciales)

El despliegue de una solución PAM transforma el flujo tradicional de administración de sistemas, eliminando por completo la exposición de credenciales en los puntos finales (endpoints):

1. **Autenticación:** El técnico accede al portal web (PVWA) e inicia sesión con sus credenciales individuales corporativas.
2. **Selección del Activo:** El sistema presenta el catálogo de servidores a los que el usuario tiene autorización explícita, aplicando el principio de mínimo privilegio.
3. **Petición de Conexión:** Al seleccionar el servidor destino, el PVWA invoca al componente PSM para iniciar la sesión.
4. **Inyección de Credenciales:** El PSM recupera internamente la contraseña o llave SSH correspondiente desde el Vault y la inyecta de forma automatizada en la sesión remota.
5. **Establecimiento de Sesión:** El técnico accede a la consola de comandos del servidor crítico sin conocer en ningún momento la contraseña de la cuenta raíz (`root`), mitigando el riesgo de filtraciones por malware o técnicas de keylogging en su equipo local. Toda la actividad queda registrada en el PSM.
