# Laboratorio 5: Campaña de Phishing Ético y Programa de Concienciación

## 1. Introducción y Objetivos
El eslabón más débil en la cadena de seguridad suele ser el factor humano. Este laboratorio tiene como objetivo diseñar, ejecutar y analizar una campaña de phishing ético controlada, seguida del diseño de un programa de concienciación corporativo para mitigar los riesgos asociados a la ingeniería social.

### Objetivos del laboratorio:
*   Desplegar una infraestructura de simulación aislada y 100% segura.
*   Diseñar un escenario de phishing realista basado en urgencia corporativa (pretexto).
*   Analizar las métricas de interacción de los usuarios.
*   Elaborar una propuesta de plan formativo de concienciación (Security Awareness).

---

## 2. Arquitectura del Laboratorio (Entorno Local Seguro)
Para garantizar la privacidad y la seguridad, evitando interactuar con servidores de correo reales o disparar filtros de spam comerciales, se ha optado por un entorno contenedorizado en **Docker Desktop** que corre de forma local en el host del analista:

                 [ Máquina del Analista ]
                             │
         ┌───────────────────┴───────────────────┐
         ▼                                       ▼
    [ GoPhish ] ───(Envía correo simulado)───► [ MailHog ]
    (Port 3333/8080)                            (Port 8025/1025)

*   **GoPhish (v0.12.1):** Servidor y motor de campañas de phishing.
*   **MailHog:** Servidor SMTP ficticio que intercepta y almacena localmente todos los correos entrantes, permitiendo visualizar los emails sin que salgan a internet.

---

## 3. Diseño de la Campaña (Ingeniería Social)

### El Pretexto (Escenario)
Se simuló un comunicado crítico y urgente del **Departamento de Soporte Técnico y Ciberseguridad** que solicitaba la migración obligatoria del portal de empleado en un plazo de 24 horas bajo amenaza de suspensión de cuenta.

*   **Plantilla de correo (Email Template):** Utiliza variables personalizadas (`{{.FirstName}}`) para aumentar la credibilidad del mensaje y un botón de llamada a la acción enlazado a la Landing Page.
*   **Página de aterrizaje (Landing Page):** Un clon simplificado de un formulario de inicio de sesión de portal interno corporativo que registra la acción de enviar datos sin almacenar credenciales en texto plano.

---

## 4. Análisis de Resultados (Métricas reales)

La campaña se lanzó sobre un grupo objetivo de **6 usuarios simulados** del Departamento de Administración. Los resultados obtenidos en la consola de GoPhish son los siguientes:

### Resumen de Métricas

| Métrica | Total | Porcentaje | Estado de Riesgo |
| :--- | :---: | :---: | :--- |
| **Correos Enviados** | 6 | 100% | - |
| **Correos Abiertos** | 1 | 16.67% | 🟡 Bajo control |
| **Clics en el Enlace** | 1 | 16.67% | 🔴 Requiere formación |
| **Datos Enviados (Credenciales)** | 1 | 16.67% | 🔴 Alto Riesgo |
| **Correos Reportados** | 0 | 0.00% | 🔴 Muy Alto Riesgo |

<img width="1901" height="907" alt="image" src="https://github.com/user-attachments/assets/93aa4f8b-6495-4f9a-933a-006b662cdae7" />


---

## 5. Programa de Concienciación Propuesto (Plan de Acción)

A raíz de los resultados de la simulación, se propone el siguiente programa de capacitación para fortalecer la cultura de ciberseguridad en la empresa:

### Fase 1: Feedback Inmediato (Teachable Moments)
*   **Acción:** Implementar redirección automática en la landing page del simulacro hacia un portal interactivo breve que explique al empleado qué pistas del correo (remitente sospechoso, sentido de urgencia, enlaces externos) delataban que era un phishing.

### Fase 2: Formación e Implantación del Botón de Reporte
*   **Capacitación:** Sesión práctica de 15 minutos enfocada en el reconocimiento de técnicas de ingeniería social.
*   **Herramienta:** Implementar un complemento en el gestor de correo (ej. "Reportar Phishing" en Outlook/Gmail) que automatice el envío de correos sospechosos al buzón del equipo de ciberseguridad (SOC).

### Fase 3: Refuerzo y Simulación Continua
*   **Frecuencia:** Lanzamiento de simulacros trimestrales con pretextos variados (falsas facturas, ofertas del departamento de RRHH, alertas de MFA).
*   **Gamificación:** Premiar o destacar mensualmente a los departamentos con mayor tasa de reporte de correos sospechosos.
