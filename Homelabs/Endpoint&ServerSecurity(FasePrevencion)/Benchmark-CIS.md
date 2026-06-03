# Hardening de Servidor: Ubuntu 24.04 (CIS Benchmark)

## Objetivo
Fortalecimiento de la seguridad de un servidor Ubuntu 24.04 LTS siguiendo las directrices del **CIS (Center for Internet Security) Benchmark Level 1**.

## Herramientas utilizadas
- **Sistema Operativo:** Ubuntu 24.04.4 LTS.
- **Auditoría:** `OpenSCAP` con el perfil `cis_level1_server`.
- **Entorno:** Máquina virtual en VirtualBox.

## Implementación
Se realizaron las siguientes configuraciones de seguridad:
- [x] **Gestión de Autenticación:** Configuración de `pam_faillock` para prevenir ataques de fuerza bruta.
- [x] **Permisos Sensibles:** Restricción de acceso al archivo `/etc/shadow`.
- [x] **Gestión de Servicios:** Deshabilitación de servicios innecesarios (ej. `avahi-daemon`).

## Incidentes y Resolución
Durante la fase de bastionado de los módulos PAM (`/etc/pam.d/`), se produjo un bloqueo de acceso al sistema debido a un error de sintaxis en los archivos de configuración.

**Proceso de recuperación:**
1. Acceso mediante `root shell` en modo `recovery`.
2. Reversión de cambios en `common-auth` y `common-account`.
3. Regeneración de perfiles de autenticación mediante `pam-auth-update --force`.
*Lección aprendida:* La importancia de validar configuraciones de autenticación en entornos de laboratorio antes de aplicar cambios masivos y mantener siempre una vía de acceso de emergencia.

## Resultados de la Auditoría

El servidor presenta un alto nivel de cumplimiento tras la remediación manual, habiéndose validado la integridad de los servicios críticos y las políticas de acceso.
- Se adjunta el reporte final en formato HTML: `cis-final.html`.
  <img width="2282" height="1232" alt="image" src="https://github.com/user-attachments/assets/b33fa288-34e0-4764-ad14-6006e0e63ead" />
