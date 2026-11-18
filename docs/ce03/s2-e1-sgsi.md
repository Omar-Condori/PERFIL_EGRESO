# CE0321-CE0324 - S2 - E1: Planificación e Implementación del SGSI

**Equipo:** Omar Condori Pachauri (201810211) · Merry Elizabeth Flores Llanque (202211953) · Jaqueline Huahuaccapa Ccama (202210712) — Ciclo 10.

## 1. Resumen ejecutivo del SGSI

**Alcance.** El Sistema de Gestión de Seguridad de la Información (SGSI) protege la plataforma Finanzas EPE tal como está desplegada: un Web Service Docker y una base PostgreSQL en Render, el código en GitHub, las cuentas que los administran y los equipos de desarrollo del equipo. La plantilla de la línea supone VMware ESXi, AWS y VPN; el proyecto real no los usa, de modo que se aplican los mismos criterios ISO/IEC 27001 a los activos que sí existen.

**Riesgos principales.** Se identificaron 16 riesgos con el método de ISO/IEC 27005: 2 críticos, 8 altos y 6 medios. Los dos críticos son la pérdida de la base de datos (el panel de Render la elimina el 17/12/2026 si no se actualiza el plan y no hay respaldos) y el acceso de administrador con una credencial de demostración simple publicada en un informe. Tras el tratamiento planificado, el riesgo residual queda en 11 bajos y 5 medios.

**Controles y SoA.** La Declaración de Aplicabilidad evalúa los 93 controles de ISO/IEC 27002:2022: 71 aplican y 22 se excluyen con justificación (casi todos físicos, delegados al proveedor). Del total aplicable, 4 están implementados, 40 parciales y 27 planificados.

**Ciclo PHVA.** Planificar: secciones 2 a 6. Hacer: secciones 7 a 12. Verificar: auditoría interna y métricas (sección 13). Actuar: acciones correctivas y revisión de la dirección.

<img src="../../img_s2-e1-sgsi/media/b8e2dde877adabe5174eebee93ecd44f34dedd65.png" title="Figura 1.1 — Ciclo PHVA del SGSI y secciones de este documento." style="width:6.25in;height:0.91667in" alt="Figura 1.1 — Ciclo PHVA del SGSI y secciones de este documento." />

*Figura 1.1 — Ciclo PHVA del SGSI y secciones de este documento.*

**Conclusión.** El SGSI es viable para el tamaño del proyecto: la mayoría de los controles ya tiene un respaldo parcial en la arquitectura, y las acciones críticas son pocas y de bajo costo (cambiar una credencial, activar MFA, hacer un respaldo externo).

|                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Estado de la evidencia.** Los activos, riesgos y controles se basan en el informe de despliegue (v1.0.4), en las capturas del panel de Render, en verificaciones externas a la URL pública el 18/11/2026 y en los Entregables 1 y 2. Dos riesgos (R-10 y la presencia de llaves JWT en el repositorio) provienen del repositorio base del proyecto y deben verificarse en FinanzasEPE. La contraseña del administrador no se reproduce en este documento, conforme a la regla de la línea. |

## 2. Contexto de la organización (ISO/IEC 27001, cláusula 4)

**Tabla 1. Cuestiones internas y externas.**

| **Tipo** | **Cuestión**                                                              | **Efecto en el SGSI**                            |
|----------|---------------------------------------------------------------------------|--------------------------------------------------|
| Externa  | Dependencia de Render (PaaS) y de las condiciones del plan gratuito \[1\] | Riesgos R-01, R-05, R-15; controles de nube 5.23 |
| Externa  | Ley 29733 de Protección de Datos Personales (Perú) \[4\]                  | Riesgo R-11; control 5.34                        |
| Externa  | Amenazas comunes de aplicaciones web expuestas a Internet                 | Controles 8.3, 8.5, 8.26                         |
| Interna  | Equipo de 3 integrantes con roles múltiples y presupuesto cero            | Alcance reducido; roles RD, RI y RS              |
| Interna  | Datos de demostración sembrados automáticamente al arrancar               | Riesgo R-04; control 8.33                        |
| Interna  | Aplicación con roles y permisos granulares y bitácora de auditoría        | Base para 5.15, 5.18 y 8.15                      |

**Tabla 2. Partes interesadas y sus requisitos.**

| **Parte interesada**                    | **Requisito de seguridad**                                            |
|-----------------------------------------|-----------------------------------------------------------------------|
| Dirección / cliente (laboratorio)       | Confidencialidad de ventas, caja y planilla; continuidad del servicio |
| Usuarios (cajeros, finanzas, logística) | Acceso confiable y datos correctos                                    |
| Equipo del proyecto                     | Operación segura y reproducible; evidencia verificable                |
| Render y GitHub (proveedores)           | Uso conforme a sus términos; cuentas protegidas                       |
| Docente y evaluadores                   | Cumplimiento de la línea y conducta ética                             |
| Autoridad de protección de datos        | Tratamiento lícito de datos personales cuando se usen datos reales    |

## 3. Alcance del SGSI

| **Incluido**                                                                          | **Excluido (con justificación)**                                                                |
|---------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Web Service FinanzasEPE (Docker, Spring Boot, SPA) y su configuración                 | Infraestructura física, hipervisor y red interna de Render: responsabilidad del proveedor \[1\] |
| Base PostgreSQL finanzas-db y sus respaldos                                           | VMware ESXi, AWS, VPN y servidores on-premise: no existen en el proyecto                        |
| Repositorio GitHub, pipeline de build y despliegue                                    | Dispositivos y redes de los usuarios finales                                                    |
| Cuentas de Render y GitHub, credenciales, llaves JWT y variables de entorno           | Datos personales reales: no se cargan; si se cargaran, el alcance se amplía (R-11)              |
| Procesos: desarrollo, despliegue, respaldo, monitoreo, incidentes, control de accesos | Sistemas de terceros no usados por el proyecto                                                  |
| Equipos de desarrollo de los tres integrantes                                         |                                                                                                 |

## 4. Liderazgo y objetivos de seguridad

**Política de seguridad (resumen).** El equipo se compromete a proteger la confidencialidad, integridad y disponibilidad de la información de Finanzas EPE, a cumplir la ley y el código de ética ACM, a gestionar los riesgos con criterios medibles y a mejorar de forma continua. Será aprobada y firmada por los tres integrantes y el docente: \[firmas pendientes\].

**Tabla 3. Roles y responsabilidades.**

| **Rol**                             | **Responsabilidades**                                        | **Asignación nominal**                            |
|-------------------------------------|--------------------------------------------------------------|---------------------------------------------------|
| Responsable de Seguridad (RS)       | Políticas, cuentas, credenciales, riesgos, auditoría interna | \[a asignar\]                                     |
| Responsable de Infraestructura (RI) | Render, despliegue, monitoreo, configuración, cambios        | \[a asignar\]                                     |
| Responsable de Datos (RD)           | Base de datos, respaldos, datos personales                   | \[a asignar\]                                     |
| Comité del SGSI (los tres)          | Aprueba riesgos residuales y revisa el SGSI                  | Omar Condori, Merry Flores, Jaqueline Huahuaccapa |

**Tabla 4. Objetivos medibles.**

| **N.º** | **Objetivo**                                       | **Indicador**                                  | **Meta** | **Plazo**  |
|---------|----------------------------------------------------|------------------------------------------------|----------|------------|
| O1      | Eliminar riesgos altos y críticos residuales       | Riesgos residuales Alto o Crítico              | 0        | 15/12/2026 |
| O2      | Proteger las cuentas administrativas               | Cuentas de Render y GitHub con MFA             | 100 %    | 20/11/2026 |
| O3      | Garantizar la recuperación de los datos            | Antigüedad del último respaldo verificado      | ≤ 24 h   | 10/12/2026 |
| O4      | Mantener el servicio disponible en horario laboral | Disponibilidad medida por sonda                | ≥ 99 %   | 30/11/2026 |
| O5      | Cobertura de controles                             | Controles aplicables implementados o parciales | ≥ 80 %   | 15/12/2026 |

## 5. Inventario de activos y clasificación de la información

Escala C-I-D de 1 a 3 (1 bajo, 3 alto). Criticidad = valor máximo. El inventario completo, con fórmulas, está en la hoja «Activos» del libro Infra_S2_Matrices.xlsx.

| **ID** | **Activo**                                               | **Tipo**     | **Propietario**    | **C** | **I** | **D** | **Crit.** |
|--------|----------------------------------------------------------|--------------|--------------------|-------|-------|-------|-----------|
| A-01   | Base de datos finanzas-db (datos de negocio)             | Información  | R. Datos           | 3     | 3     | 3     | Alta      |
| A-02   | Datos personales y de planilla (personal y pacientes)    | Información  | RD                 | 3     | 3     | 2     | Alta      |
| A-03   | Bitácora de auditoría (lab_auditoria)                    | Información  | R. Seguridad       | 2     | 3     | 2     | Alta      |
| A-04   | Web Service FinanzasEPE (contenedor Docker)              | Servicio     | R. Infraestructura | 2     | 3     | 3     | Alta      |
| A-05   | Código fuente (repositorio Jaqueline2101/FinanzasEPE)    | Software     | RI                 | 2     | 3     | 2     | Alta      |
| A-06   | Imagen Docker y Dockerfile                               | Software     | RI                 | 1     | 3     | 2     | Alta      |
| A-07   | Variables de entorno (SPRING_DATASOURCE\_\*, PORT)       | Credenciales | RS                 | 3     | 2     | 2     | Alta      |
| A-08   | Credenciales de la aplicación (administrador y usuarios) | Credenciales | RS                 | 3     | 3     | 2     | Alta      |
| A-09   | Llaves JWT (par RSA)                                     | Credenciales | RS                 | 3     | 3     | 2     | Alta      |
| A-10   | Cuenta y espacio de trabajo de Render                    | Cuenta       | RI                 | 3     | 3     | 3     | Alta      |
| A-11   | Cuenta de GitHub del repositorio                         | Cuenta       | RI                 | 2     | 3     | 2     | Alta      |
| A-12   | Dominio y certificado TLS (onrender.com)                 | Red          | RI                 | 1     | 3     | 3     | Alta      |
| A-13   | Equipos de desarrollo del equipo                         | Hardware     | Cada integrante    | 2     | 2     | 2     | Media     |
| A-14   | Documentación técnica (informes y diagramas)             | Información  | RS                 | 1     | 2     | 1     | Media     |
| A-15   | Respaldos de la base de datos (planificado)              | Información  | RD                 | 3     | 3     | 3     | Alta      |
| A-16   | Registros y métricas de Render                           | Información  | RI                 | 2     | 2     | 1     | Media     |
| A-17   | Conocimiento del equipo (3 integrantes)                  | Personas     | RS                 | 1     | 2     | 2     | Media     |

Clasificación de la información: Pública (documentación sin datos sensibles), Interna (informes, código), Confidencial (datos de negocio, planilla, credenciales, llaves). El etiquetado se aplicará en el encabezado de cada documento.

## 6. Evaluación de riesgos (ISO/IEC 27005)

Método: riesgo = probabilidad (1 a 5) × impacto (1 a 5). Bajo 1–4, Medio 5–9, Alto 10–15, Crítico 16–25. Se aceptan riesgos Bajos; los Medios requieren plan; los Altos y Críticos requieren tratamiento antes del plazo. Probabilidad e impacto son juicios del equipo basados en la evidencia citada; deben revisarse con el docente.

| **ID** | **Amenaza**                                                | **Vulnerabilidad**                                                                                    | **P** | **I** | **Nivel**  |
|--------|------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|-------|-------|------------|
| R-01   | Pérdida total de la base de datos                          | Plan gratuito con vencimiento el 17/12/2026 y sin respaldos administrados                             | 5     | 5     | 25 Crítico |
| R-02   | Acceso no autorizado como administrador                    | Credencial de demostración simple, publicada en un informe                                            | 4     | 5     | 20 Crítico |
| R-03   | Abuso de privilegios                                       | El filtro JWT asigna todas las autoridades al administrador (sin mínimo privilegio)                   | 3     | 4     | 12 Alto    |
| R-04   | Mezcla de datos de demostración con datos reales           | DemoDataInitializer se ejecuta al arrancar en producción                                              | 3     | 3     | 9 Medio    |
| R-05   | Indisponibilidad del servicio                              | Instancia única que se suspende a los 15 min sin tráfico (plan Free)                                  | 4     | 3     | 12 Alto    |
| R-06   | Caída por falta de memoria (error 502)                     | Topes de JVM suman 508 de 512 MB                                                                      | 3     | 4     | 12 Alto    |
| R-07   | Fuga de secretos                                           | Llaves o credenciales en repositorios, capturas o informes                                            | 3     | 5     | 15 Alto    |
| R-08   | Reconocimiento de la API por un atacante                   | Página Swagger UI accesible sin autenticación en producción                                           | 3     | 2     | 6 Medio    |
| R-09   | Toma de control de las cuentas de Render o GitHub          | MFA no verificado en las cuentas                                                                      | 3     | 5     | 15 Alto    |
| R-10   | Creación de usuarios con roles elegidos por el solicitante | Endpoint /api/auth/register sin restricción (hallazgo del repositorio base; verificar en FinanzasEPE) | 3     | 5     | 15 Alto    |
| R-11   | Tratamiento ilícito de datos personales                    | Uso de datos reales sin base legal ni medidas (Ley 29733)                                             | 2     | 5     | 10 Alto    |
| R-12   | Incidentes no detectados                                   | Sin alertas ni monitoreo externo                                                                      | 4     | 3     | 12 Alto    |
| R-13   | Regresiones o despliegues defectuosos                      | Sin pruebas en CI; 1 despliegue fallido observado (33,7 s)                                            | 3     | 3     | 9 Medio    |
| R-14   | Pérdida de conocimiento                                    | Pocas personas conocen el despliegue                                                                  | 3     | 3     | 9 Medio    |
| R-15   | Cambio o interrupción del proveedor                        | Dependencia de las condiciones del plan gratuito de Render                                            | 2     | 4     | 8 Medio    |
| R-16   | Robo o pérdida de un equipo con credenciales               | Equipos personales sin cifrado verificado                                                             | 2     | 4     | 8 Medio    |

<img src="../../img_s2-e1-sgsi/media/38ad1f6063d9f696c4c1c9174d3b498ba66d4f6e.png" title="Figura 6.1 — Matriz de riesgos inherente y residual." style="width:5.41667in;height:2.57292in" alt="Figura 6.1 — Matriz de riesgos inherente y residual." />

*Figura 6.1 — Matriz de riesgos inherente y residual.*

## 7. Tratamiento de riesgos

| **ID** | **Opción** | **Acción**                                                                       | **Resp.** | **Plazo**  | **Residual** |
|--------|------------|----------------------------------------------------------------------------------|-----------|------------|--------------|
| R-01   | Mitigar    | Respaldo externo diario y migración a plan de pago antes de la fecha             | RD        | 10/12/2026 | 6 Medio      |
| R-02   | Mitigar    | Cambiar la contraseña, usar una no trivial y retirarla de todos los documentos   | RS        | 20/11/2026 | 4 Bajo       |
| R-03   | Mitigar    | Cargar permisos desde la base y limitar ACCESO_TOTAL                             | RI        | 15/12/2026 | 6 Medio      |
| R-04   | Mitigar    | Activarlo solo con un perfil de demostración                                     | RI        | 15/12/2026 | 2 Bajo       |
| R-05   | Mitigar    | Sonda externa de disponibilidad; evaluar plan de pago                            | RI        | 30/11/2026 | 6 Medio      |
| R-06   | Mitigar    | Medir con Metrics y reducir el tope de metaspace                                 | RI        | 30/11/2026 | 6 Medio      |
| R-07   | Mitigar    | Secretos solo en variables de entorno; rotar llaves; revisar el historial de git | RS        | 30/11/2026 | 4 Bajo       |
| R-08   | Mitigar    | Deshabilitar Swagger en el perfil de producción                                  | RI        | 30/11/2026 | 1 Bajo       |
| R-09   | Mitigar    | Activar MFA en ambas cuentas y guardar códigos de recuperación                   | RS        | 20/11/2026 | 4 Bajo       |
| R-10   | Mitigar    | Restringir el registro a administradores y ignorar roles enviados                | RI        | 30/11/2026 | 3 Bajo       |
| R-11   | Evitar     | No cargar datos reales hasta cumplir; mantener datos sintéticos                  | RD        | Permanente | 3 Bajo       |
| R-12   | Mitigar    | Sonda externa y revisión periódica de logs                                       | RI        | 30/11/2026 | 4 Bajo       |
| R-13   | Mitigar    | Pruebas automáticas previas al despliegue y uso de Rollback                      | RI        | 15/12/2026 | 4 Bajo       |
| R-14   | Mitigar    | Documentación (Entregables 1 a 3) y rotación de tareas                           | RS        | 15/12/2026 | 4 Bajo       |
| R-15   | Transferir | Mantener la aplicación portable en Docker; plan alterno documentado              | RI        | 15/12/2026 | 6 Medio      |
| R-16   | Mitigar    | Cifrado de disco y bloqueo de sesión                                             | Todos     | 30/11/2026 | 3 Bajo       |

Riesgo aceptado: ninguno de nivel Alto o Crítico. Los residuales Medios (R-01, R-03, R-05, R-13, R-15) serán revisados en la revisión por la dirección; la aceptación formal queda pendiente de la firma del comité.

## 8. Declaración de Aplicabilidad (SoA)

Se evaluaron los 93 controles de ISO/IEC 27002:2022 \[3\] en sus cuatro temas. La SoA completa, con justificación y evidencia por control, está en la hoja «SoA» del libro Excel. Resumen:

| **Tema**        | **Controles** | **Implementado** | **Parcial** | **Planificado** | **No aplica** |
|-----------------|---------------|------------------|-------------|-----------------|---------------|
| 5 Organizativos | 37            | 3                | 18          | 14              | 2             |
| 6 De personas   | 8             | 0                | 1           | 3               | 4             |
| 7 Físicos       | 14            | 0                | 0           | 4               | 10            |
| 8 Tecnológicos  | 34            | 1                | 21          | 6               | 6             |
| Total           | 93            | 4                | 40          | 27              | 22            |

**Tabla 5. Controles clave con evidencia (extracto de la SoA).**

| **Control**                                                          | **Estado**   | **Justificación / evidencia**                                               |
|----------------------------------------------------------------------|--------------|-----------------------------------------------------------------------------|
| 5.9 Inventario de información y otros activos asociados              | Implementado | Inventario de 17 activos (hoja Activos)                                     |
| 5.14 Transferencia de información                                    | Implementado | Tráfico de usuarios por HTTPS (verificado)                                  |
| 5.17 Información de autenticación                                    | Parcial      | Variables de entorno para secretos; credencial de demostración débil (R-02) |
| 5.23 Seguridad de la información para el uso de servicios en la nube | Parcial      | Despliegue en Render; modelo de responsabilidad compartida documentado      |
| 5.30 Preparación de las TIC para la continuidad del negocio          | Parcial      | Rollback disponible; respaldo de BD no operativo (R-01)                     |
| 8.2 Derechos de acceso privilegiado                                  | Parcial      | Administrador con todas las autoridades (R-03)                              |
| 8.5 Autenticación segura                                             | Parcial      | JWT firmado con RSA; MFA de cuentas pendiente (R-09)                        |
| 8.13 Respaldo de la información                                      | Parcial      | Scripts probados en laboratorio; sin operación en Render (R-01)             |
| 8.16 Actividades de monitoreo                                        | Parcial      | Endpoint /actuator/health público y Metrics; sonda externa planificada      |
| 8.19 Instalación de software en sistemas operativos                  | Implementado | Imagen Docker inmutable; sin instalación manual                             |
| 8.20 Seguridad de las redes                                          | Parcial      | TLS en el borde; reglas de acceso de la base por verificar                  |
| 8.24 Uso de criptografía                                             | Parcial      | TLS administrado y JWT con RSA; gestión de llaves pendiente (R-07)          |

Exclusiones justificadas: 22 controles. Los controles 7.1 a 7.6, 7.8 y 7.11 a 7.13 son físicos y los ejecuta el proveedor; 6.1, 6.2, 6.4 y 6.5 no aplican porque el proyecto no contrata personal; 8.12, 8.14, 8.17, 8.18, 8.23 y 8.30 no aplican por el alcance reducido o porque la plataforma los gestiona.

## 9. Políticas y procedimientos

Cada política está adaptada al despliegue real y a los riesgos de la sección 6, y no es una plantilla genérica.

| **Política / procedimiento** | **Reglas específicas del proyecto**                                                                                                                                                      |
|------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Seguridad de la información  | Alcance de la sección 3; los activos se clasifican C-I-D; los riesgos Altos y Críticos se tratan antes de su plazo; cada cambio relevante se evalúa contra la matriz de riesgos          |
| Control de accesos           | Mínimo privilegio; ninguna cuenta compartida; MFA obligatorio en Render y GitHub; revisión de permisos de la aplicación cada trimestre; el administrador no usa credenciales por defecto |
| Credenciales y secretos      | Los secretos solo viven en variables de entorno de Render; prohibido incluirlos en el repositorio, capturas o informes; rotación de llaves JWT y claves de base ante sospecha de fuga    |
| Gestión de incidentes        | Procedimiento de la sección 11; todo incidente se registra con fecha, impacto, causa y acción                                                                                            |
| Proveedores y nube           | Render y GitHub son los únicos proveedores; se revisan los cambios de plan y de condiciones; la aplicación debe poder reconstruirse desde el repositorio (portabilidad Docker)           |
| Respaldo y recuperación      | Respaldo diario de la base con verificación y copia fuera de Render; prueba de restauración mensual; objetivos RPO 24 h y RTO 2 h                                                        |
| Uso aceptable                | Los activos del proyecto se usan solo para fines del proyecto; pruebas de seguridad solo sobre el sistema propio; no se cargan datos personales reales                                   |

## 10. Gestión de identidades y accesos

Matriz objetivo de acceso a la aplicación por rol, según las historias de usuario del Entregable 1 de Ingeniería de Software. Hoy el administrador recibe todas las autoridades por filtro (R-03); la matriz es el estado al que debe converger.

| **Función**                      | **Administrador** | **Cajero** | **Aprobador** | **Logística** | **Finanzas** |
|----------------------------------|-------------------|------------|---------------|---------------|--------------|
| Registrar ventas y cobrar        | ✔                 | ✔          | ✖             | ✖             | ✖            |
| Solicitar descuento / anulación  | ✔                 | ✔          | ✖             | ✖             | ✖            |
| Aprobar descuentos y anulaciones | ✔                 | ✖          | ✔             | ✖             | ✖            |
| Caja chica y arqueo              | ✔                 | ✔          | ✖             | ✖             | ✔            |
| Compras, inventario y kardex     | ✔                 | ✖          | ✖             | ✔             | ✖            |
| Honorarios y planilla            | ✔                 | ✖          | ✖             | ✖             | ✔            |
| Usuarios, roles y permisos       | ✔                 | ✖          | ✖             | ✖             | ✖            |
| Bitácora de auditoría            | ✔                 | ✖          | ✖             | ✖             | ✖            |
| Reportes                         | ✔                 | ✖          | ✖             | ✔             | ✔            |

| **Cuenta de infraestructura** | **Titular** | **Control**                                                       | **Estado**               |
|-------------------------------|-------------|-------------------------------------------------------------------|--------------------------|
| Render (workspace)            | Equipo (RI) | MFA, mínimo de colaboradores                                      | MFA sin verificar (R-09) |
| GitHub (repositorio)          | Equipo (RI) | MFA, rama main protegida, revisión de colaboradores               | Sin verificar (R-09)     |
| Base de datos                 | RD          | Credenciales solo en variables de entorno; sin usuario compartido | Parcial                  |
| Aplicación: administrador     | RS          | Contraseña no trivial; permisos desde la base                     | No cumple (R-02, R-03)   |

Manejo de credenciales: ninguna se escribe en documentos o capturas; se usan variables de entorno y un gestor de contraseñas; rotación inmediata ante exposición. Como el informe de despliegue original contiene una credencial de administrador, debe retirarse de ese archivo y cambiarse la contraseña (acción de R-02).

## 11. Gestión de incidentes, continuidad y recuperación

**Tabla 6. Procedimiento de incidentes.**

| **Etapa**                       | **Qué se hace**                                                                                    | **Responsable** |
|---------------------------------|----------------------------------------------------------------------------------------------------|-----------------|
| 1\. Detección y reporte         | Cualquier integrante reporta por el canal del equipo con fecha, síntoma y evidencia (captura, log) | Todos           |
| 2\. Clasificación               | Alta (pérdida de datos o acceso indebido), Media (servicio caído), Baja (error funcional)          | RS              |
| 3\. Contención                  | Aislar: cambiar credenciales, suspender el servicio o hacer Rollback                               | RI              |
| 4\. Erradicación y recuperación | Corregir la causa; restaurar desde respaldo si hubo pérdida de datos                               | RI / RD         |
| 5\. Registro y aprendizaje      | Registro del incidente y acción preventiva; revisión en el comité                                  | RS              |

**Caso real.** El informe de despliegue documenta un incidente de disponibilidad: error 502 persistente por límites de memoria de la JVM (-Xmx160m y pila de 256 KB) y una transacción global en el arranque. Se contuvo y resolvió reajustando la JVM y quitando la transacción. Es el primer registro de lecciones aprendidas (control 5.27).

**Tabla 7. Servicios críticos, RTO y RPO.**

| **Servicio**         | **RTO**  | **RPO**   | **Mecanismo**                                          |
|----------------------|----------|-----------|--------------------------------------------------------|
| Aplicación web y API | ≤ 15 min | No aplica | Rollback o redespliegue desde GitHub (≈ 6 min medidos) |
| Base de datos        | ≤ 2 h    | ≤ 24 h    | Restauración desde el respaldo externo                 |
| Código fuente        | ≤ 1 h    | ≈ 0       | Repositorio GitHub                                     |

## 12. Ética de la seguridad de la información (ACM)

El equipo adopta el Código de Ética y Conducta Profesional de la ACM \[5\]. Su aplicación concreta es:

| **Principio ACM**                            | **Aplicación en el proyecto**                                                                                                                      |
|----------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------|
| 1.2 Evitar daño                              | Las pruebas de seguridad se limitan al sistema propio, sin interrumpir a otros usuarios de Render ni degradar el servicio compartido               |
| 1.3 Ser honesto y digno de confianza         | Los informes rotulan lo verificado y lo pendiente; no se presentan como probados controles que no lo están                                         |
| 1.6 Respetar la privacidad                   | No se usan datos personales reales; los datos de demostración son sintéticos                                                                       |
| 1.7 Honrar la confidencialidad               | Las credenciales y llaves no se escriben en documentos; se comparten solo por canales seguros                                                      |
| 2.8 Acceder a recursos solo con autorización | Alcance ético de pruebas: solo finanzasepe.onrender.com y la base finanzas-db del equipo; prohibido atacar infraestructura de Render o de terceros |

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<tbody>
<tr class="odd">
<td><p><strong>[ Declaración de ética de seguridad — firmas ]</strong></p>
<p><em>Omar Condori Pachauri · Merry Elizabeth Flores Llanque · Jaqueline Huahuaccapa Ccama (firma y fecha, a completar)</em></p></td>
</tr>
</tbody>
</table>

## 13. Evaluación del desempeño del SGSI

**Tabla 8. Métricas.**

| **ID** | **Métrica**                                    | **Meta** | **Valor actual**                            |
|--------|------------------------------------------------|----------|---------------------------------------------|
| M1     | Controles aplicables implementados o parciales | ≥ 80 %   | 62 % (44 de 71)                             |
| M2     | Riesgos Altos o Críticos abiertos              | 0        | 10 (inherentes); 0 tras tratar              |
| M3     | Antigüedad del último respaldo verificado      | ≤ 24 h   | Sin respaldo en Render; laboratorio probado |
| M4     | Disponibilidad medida                          | ≥ 99 %   | Sin medir                                   |
| M5     | Cuentas administrativas con MFA                | 100 %    | Sin verificar                               |
| M6     | No conformidades cerradas a tiempo             | 100 %    | 0 de 6 (recién abiertas)                    |

**Auditoría interna n.º 1 (autoauditoría).** Criterio: ISO/IEC 27001 y la SoA. Método: revisión documental, capturas del panel de Render y pruebas externas del 18/11/2026. No la realizó un auditor independiente.

| **ID** | **Hallazgo**                                                                       | **Tipo**    | **Acción correctiva**                             | **Plazo**  |
|--------|------------------------------------------------------------------------------------|-------------|---------------------------------------------------|------------|
| NC-01  | Credencial de administrador simple y publicada en un informe                       | Mayor       | Cambiar contraseña y retirar del documento (R-02) | 20/11/2026 |
| NC-02  | No existe respaldo de la base en Render y la base vence el 17/12/2026              | Mayor       | Respaldo externo y plan de pago (R-01)            | 10/12/2026 |
| NC-03  | El administrador recibe todas las autoridades por filtro                           | Mayor       | Cargar permisos desde la base (R-03)              | 15/12/2026 |
| NC-04  | Swagger UI visible en producción                                                   | Menor       | Deshabilitar en producción (R-08)                 | 30/11/2026 |
| NC-05  | MFA de las cuentas no verificado                                                   | Menor       | Activar y registrar evidencia (R-09)              | 20/11/2026 |
| NC-06  | Sin monitoreo externo de disponibilidad                                            | Menor       | Sonda cada 5 min (R-12)                           | 30/11/2026 |
| OBS-01 | /actuator/health es público y expone solo el estado UP; no se expone /actuator/env | Observación | Aceptable; mantener solo health                   | —          |

Revisión por la dirección: se realizará con el docente y el comité al cierre de la acción correctiva del 15/12/2026 y revisará objetivos, métricas, hallazgos y riesgos residuales.

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<tbody>
<tr class="odd">
<td><p><strong>[ Acta de revisión por la dirección ]</strong></p>
<p><em>Fecha, asistentes, decisiones y firmas (a completar)</em></p></td>
</tr>
</tbody>
</table>

## 14. Conclusiones y matriz maestra de trazabilidad

El SGSI cubre los activos reales del despliegue, tiene 16 riesgos evaluados, una SoA de 93 controles y un plan con responsables y fechas. Lo más urgente es proteger la credencial de administrador y salvar la base de datos antes del 17/12/2026.

| **Riesgo** | **Control(es)** | **Documento / política**  | **Evidencia**                               | **Estado**  |
|------------|-----------------|---------------------------|---------------------------------------------|-------------|
| R-01       | 8.13, 5.30      | Tratamiento (sec. 7)      | Scripts de respaldo y prueba en laboratorio | Parcial     |
| R-02       | 5.17, 8.5       | Tratamiento (sec. 7)      | Sec. 6 y 13                                 | Planificado |
| R-03       | 8.2, 5.18       | Tratamiento (sec. 7)      | Sec. 6 y 13                                 | Planificado |
| R-04       | 8.31, 8.33      | Tratamiento (sec. 7)      | Sec. 6 y 13                                 | Planificado |
| R-05       | 8.14, 8.16      | Tratamiento (sec. 7)      | Verificación de /actuator/health            | Planificado |
| R-06       | 8.6             | Tratamiento (sec. 7)      | Sec. 6 y 13                                 | Planificado |
| R-07       | 5.17, 8.4, 8.24 | Tratamiento (sec. 7)      | Sec. 6 y 13                                 | Planificado |
| R-08       | 8.9, 8.26       | Tratamiento (sec. 7)      | Prueba PT-06                                | Planificado |
| R-09       | 5.17, 8.5       | Tratamiento (sec. 7)      | Sec. 6 y 13                                 | Planificado |
| R-10       | 8.3, 5.15       | Tratamiento (sec. 7)      | Sec. 6 y 13                                 | Planificado |
| R-11       | 5.34, 5.31      | Política de uso aceptable | Sec. 6 y 13                                 | Planificado |
| R-12       | 8.15, 8.16      | Tratamiento (sec. 7)      | Verificación de /actuator/health            | Planificado |
| R-13       | 8.29, 8.32      | Tratamiento (sec. 7)      | Sec. 6 y 13                                 | Planificado |
| R-14       | 5.2, 6.3        | Tratamiento (sec. 7)      | Sec. 6 y 13                                 | Planificado |
| R-15       | 5.19, 5.23      | Tratamiento (sec. 7)      | Sec. 6 y 13                                 | Planificado |
| R-16       | 8.1, 7.9        | Tratamiento (sec. 7)      | Sec. 6 y 13                                 | Planificado |

## 15. Referencias

**\[1\]** Render, «Deploy for Free», Render Docs. \[En línea\]. Disponible: https://render.com/docs/free (consultado el 17/11/2026).

**\[2\]** ISO/IEC, «ISO/IEC 27001:2022 — Information security management systems — Requirements», 2022.

**\[3\]** ISO/IEC, «ISO/IEC 27002:2022 — Information security controls», 2022.

**\[4\]** Congreso de la República del Perú, «Ley N.º 29733, Ley de Protección de Datos Personales», 2011.

**\[5\]** ACM, «ACM Code of Ethics and Professional Conduct», 2018. \[En línea\]. Disponible: https://www.acm.org/code-of-ethics

**\[6\]** ISO/IEC, «ISO/IEC 27005:2022 — Guidance on managing information security risks», 2022.

**\[7\]** Equipo del proyecto, «Informe Técnico Integral … Despliegue en Render», Finanzas EPE, v1.0.4, 2026.

---

## Material complementario

- [Matrices completas: activos, riesgos, SoA de 93 controles y plan de pruebas (XLSX)](../descargas/ce03/S2_Matrices_Activos_Riesgos_SoA_Pruebas.xlsx)
