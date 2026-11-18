# CE0311 - S1 - E1: Análisis y Diseño de Red

**Equipo:** Omar Condori Pachauri (201810211) · Merry Elizabeth Flores Llanque (202211953) · Jaqueline Huahuaccapa Ccama (202210712) — Ciclo 10.

## 1. Resumen ejecutivo del diseño del despliegue

**Proyecto y organización.** Finanzas EPE es una plataforma de gestión financiera y administrativa (ventas y correlativos, compras, caja chica y general, kardex, planilla y honorarios, auditoría) construida para un laboratorio clínico. Este entregable analiza y diseña la infraestructura sobre la que el sistema ya está en producción: **https://finanzasepe.onrender.com** (versión de entrega 1.0.4).

**Adaptación de la plantilla.** La plantilla de la línea describe una red on-premise con VLAN, VPN y firewall. El proyecto real no tiene red propia: se despliega en Render, una plataforma como servicio (PaaS) con dos componentes administrados: un Web Service Docker que entrega la aplicación y un PostgreSQL gestionado. Por eso, cada elemento de la plantilla se documenta en su equivalente real (por ejemplo, zonas lógicas en lugar de VLAN y conectividad entre servicios en lugar de VPN) y se declara de forma explícita lo que no existe.

**Problema de conectividad y seguridad.** Publicar un sistema que maneja dinero y datos de personal con recursos mínimos (512 MB de RAM en el plan gratuito) exige: cifrar el tráfico de los usuarios, aislar la base de datos de Internet, controlar el acceso a la API y poder reconstruir el entorno de forma reproducible. Los riesgos preliminares detectados son la suspensión por inactividad del servicio, el vencimiento de la base de datos gratuita, la ausencia de copias de seguridad automáticas y un perfil de administrador con permisos totales.

**Topología propuesta.** Un único servicio contenedorizado sirve el frontend Angular (archivos estáticos) y la API REST de Spring Boot bajo el mismo dominio, y se conecta por JDBC a PostgreSQL. GitHub dispara cada despliegue por webhook. No hay VPN ni túnel híbrido: todos los componentes viven dentro de la plataforma y el único punto expuesto a Internet es el Web Service sobre HTTPS.

**Resultados esperados y siguientes pasos.** Un diseño documentado, medible y auditable del despliegue actual, con una matriz de componentes y de pruebas de conectividad. La implementación de los controles pendientes (respaldos, monitoreo, endurecimiento de accesos) y su validación corresponden a los entregables siguientes de la línea.

<img src="../../img_s1-e1-analisis-y-diseno-de-red/media/e5a0a6e81466e9f2042fef900c7ba2c317a2dbeb.png" title="Figura 1.1 — Topología ejecutiva del despliegue (vista simplificada)." style="width:6.25in;height:1.02083in" alt="Figura 1.1 — Topología ejecutiva del despliegue (vista simplificada)." />

*Figura 1.1 — Topología ejecutiva del despliegue (vista simplificada).*

|                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Estado de la evidencia.** Los datos de despliegue provienen del informe técnico de despliegue del equipo (producción v1.0.4). Se verificó desde fuera, el 17/11/2026, que la URL pública responde con la aplicación «Finanzas». Los límites de la plataforma se citan de la documentación oficial de Render \[1\]. También se usaron capturas del panel de Render del 17/11/2026 (figuras 5.5 y 5.6): base de datos en la región Oregon (US West), plan gratuito, PostgreSQL 18, con vencimiento el 17/12/2026. Lo que no se pudo verificar (región del Web Service, reglas de acceso a la base) aparece como «completar desde el panel de Render» y no se asume. |

## 2. Descripción de la organización y necesidades de infraestructura

### 2.1 Descripción general

| **Aspecto**                              | **Descripción**                                                                                                                                                                                            |
|------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Organización y rubro                     | Laboratorio clínico (servicios de laboratorio, imágenes, consultas y farmacia). Sistema: Plataforma Integral de Gestión Financiera y Administrativa (Finanzas EPE)                                         |
| Usuarios y sedes                         | Cajeros, aprobadores, logística, finanzas y administradores; modelo multi-sucursal. Los datos actuales son de demostración: una sede principal sembrada automáticamente (informe de despliegue, sección 4) |
| Servicios TI que dependen del despliegue | Aplicación web (SPA), API REST, base de datos PostgreSQL, tarea de poblado inicial de datos (DemoDataInitializer)                                                                                          |
| Uso de cloud y local                     | Todo en la nube (Render). No hay infraestructura local en producción; el desarrollo se hace en equipos locales (Angular + Spring Boot)                                                                     |
| Disponibilidad                           | Acceso web a demanda. El plan gratuito suspende el servicio tras 15 minutos sin tráfico y tarda hasta un minuto en reactivarlo \[1\]                                                                       |
| Ancho de banda y crecimiento             | Tráfico de oficina de pocos usuarios simultáneos; el recurso limitante es la memoria del contenedor (512 MB, JVM acotada a -Xmx280m), no el ancho de banda                                                 |

### 2.2 Servicios y procesos críticos dependientes de la infraestructura

| **Proceso**                                               | **Servicio TI que lo soporta**   | **Criticidad** |
|-----------------------------------------------------------|----------------------------------|----------------|
| Ventas, cobros y correlativos (B001, F001, T001, NC01)    | API REST + PostgreSQL            | Alta           |
| Caja chica, caja general y arqueos                        | API REST + PostgreSQL            | Alta           |
| Autenticación y control de acceso (JWT, roles y permisos) | Spring Security en el contenedor | Alta           |
| Compras, inventario y kardex                              | API REST + PostgreSQL            | Media          |
| Planilla y honorarios médicos                             | API REST + PostgreSQL            | Media          |
| Auditoría de eventos (lab_auditoria)                      | PostgreSQL                       | Alta           |
| Entrega de nuevas versiones                               | GitHub + build de Render         | Media          |

### 2.3 Partes interesadas relevantes

| **Parte interesada**                    | **Qué espera de la conectividad y la seguridad**                                                                       |
|-----------------------------------------|------------------------------------------------------------------------------------------------------------------------|
| Usuarios (cajeros, finanzas, logística) | Acceso rápido y estable por HTTPS; que el sistema no esté caído en horario de atención                                 |
| Dirección / cliente                     | Confidencialidad de ventas, caja y planilla; trazabilidad de operaciones                                               |
| Equipo del proyecto (administradores)   | Despliegue reproducible, diagnóstico por logs, control de costos                                                       |
| Render (proveedor PaaS)                 | Cumplimiento de sus límites de uso; responsable del borde, TLS y servidores subyacentes                                |
| GitHub                                  | Repositorio fuente y disparador de despliegues                                                                         |
| Docente / evaluadores                   | Evidencia verificable y reproducible                                                                                   |
| Organismos reguladores                  | Protección de datos personales (Ley 29733, Perú) para los datos de pacientes y personal cuando se use con datos reales |

## 3. Alcance del diseño del despliegue

### 3.1 Declaración de alcance

El diseño comprende la infraestructura de ejecución y conectividad de Finanzas EPE en Render: el Web Service Docker que sirve la SPA y la API, el PostgreSQL administrado, el flujo de entrega continua desde GitHub, las zonas lógicas de seguridad, los puertos y protocolos entre componentes y los controles que atraviesa una petición.

### 3.2 Límites

| **Tipo**         | **Dentro del alcance**                                                                                                                                                                                                                                          |
|------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Físicos          | Ninguno: no hay centro de datos propio ni equipos de comunicaciones. La infraestructura física es del proveedor; la base de datos está en la región Oregon (US West) según el panel de Render; la región del Web Service se completa desde su panel: \[región\] |
| Tecnológicos     | Dockerfile multi-etapa, Web Service, PostgreSQL 18 administrado, variables de entorno, dominio onrender.com con TLS, webhook GitHub → Render, JVM de Java 21, Spring Security y JWT                                                                             |
| Organizacionales | El equipo administra el código, el Dockerfile, las variables de entorno y los datos de la aplicación. Render administra el hardware, el sistema operativo base, el borde y los certificados. Asignación nominal de responsables: \[a completar por el equipo\]  |

### 3.3 Exclusiones justificadas

## 4. Objetivos del diseño del despliegue

Cada objetivo tiene un indicador medible y una meta. Son metas de diseño; su cumplimiento se evidencia en la implementación y las pruebas de los entregables siguientes.

| **Objetivo**                                | **Indicador**                                  | **Meta**         | **Responsable**          |
|---------------------------------------------|------------------------------------------------|------------------|--------------------------|
| Cifrar el 100 % del tráfico de usuarios     | Peticiones atendidas solo por HTTPS            | 100 %            | Equipo (Infraestructura) |
| Exponer un único punto público              | Servicios con acceso desde Internet            | 1 (Web Service)  | Equipo (Seguridad)       |
| Aislar la base de datos de Internet         | Conexiones externas a 5432 no autorizadas      | 0 permitidas     | Equipo (Seguridad)       |
| Mantener los secretos fuera del repositorio | Credenciales en variables de entorno           | 100 %            | Equipo (Seguridad)       |
| Despliegue reproducible                     | Entorno reconstruible desde GitHub con un push | 1 comando/evento | Equipo (Infraestructura) |
| Recuperación ante pérdida de la base        | Respaldo externo verificado                    | ≥ 1 por semana   | Equipo (Datos)           |
| Tolerar el arranque en frío                 | Tiempo de reactivación tras suspensión         | ≤ 60 s \[1\]     | Equipo (Infraestructura) |

## 5. Topologías del despliegue (diseño)

Se presentan cuatro vistas, cada una con un propósito distinto. Los diagramas se entregan además como archivos PNG independientes.

### 5.1 Topología general de la infraestructura

Vista maestra: el equipo publica código en GitHub; Render construye la imagen Docker y la ejecuta como Web Service; los usuarios llegan por HTTPS al borde de Render, que reenvía al puerto 10000 del contenedor; el contenedor consulta PostgreSQL por JDBC.

<img src="../../img_s1-e1-analisis-y-diseno-de-red/media/fe8cbd37f9f70672f5cdcaf5059aa3c8ad94b8d5.png" title="Figura 5.1 — Topología general del despliegue en Render." style="width:4.89583in;height:3.36458in" alt="Figura 5.1 — Topología general del despliegue en Render." />

*Figura 5.1 — Topología general del despliegue en Render.*

### 5.2 Topología lógica: zonas y direccionamiento

En una PaaS no se definen VLAN ni subredes: las direcciones IP las asigna la plataforma y pueden cambiar. El equivalente de la segmentación es un modelo de cinco zonas lógicas con reglas de comunicación entre ellas.

<img src="../../img_s1-e1-analisis-y-diseno-de-red/media/2525bdea1d7002df524f1fb07275f50d796d5276.png" title="Figura 5.2 — Zonas lógicas y comunicación permitida o denegada." style="width:4.47917in;height:2.17708in" alt="Figura 5.2 — Zonas lógicas y comunicación permitida o denegada." />

*Figura 5.2 — Zonas lógicas y comunicación permitida o denegada.*

**Tabla 1. Zonas lógicas (equivalente a VLAN/subredes).**

| **Zona** | **Nombre** | **Direccionamiento**                                                      | **Propósito**                              |
|----------|------------|---------------------------------------------------------------------------|--------------------------------------------|
| 1        | PÚBLICA    | Internet (dinámico)                                                       | Usuarios y navegadores, sin confianza      |
| 2        | BORDE      | Dominio finanzasepe.onrender.com, IP gestionada por Render                | Terminación TLS y enrutamiento a la zona 3 |
| 3        | APLICACIÓN | Contenedor; escucha en el puerto 10000 (variable PORT)                    | SPA estática, API REST, seguridad          |
| 4        | DATOS      | Base finanzas-db (Oregon); host \[completar desde el panel\]; puerto 5432 | Persistencia y auditoría                   |
| 5        | ENTREGA    | github.com/Jaqueline2101/FinanzasEPE                                      | Código fuente y disparo de despliegues     |

### 5.3 Diseño de seguridad por capas

No hay un firewall de red administrado por el equipo. Los controles que atraviesa una petición son, en orden: cifrado en el borde, autenticación por token, autorización por permiso, reglas de negocio transaccionales y acceso a la base con credenciales externas al código. Además, las operaciones relevantes quedan en la tabla de auditoría.

<img src="../../img_s1-e1-analisis-y-diseno-de-red/media/9cd3a1d3bfad623d8704e3e3fd0a2f0177f77279.png" title="Figura 5.3 — Controles que atraviesa una petición antes de llegar a los datos." style="width:6.25in;height:0.97917in" alt="Figura 5.3 — Controles que atraviesa una petición antes de llegar a los datos." />

*Figura 5.3 — Controles que atraviesa una petición antes de llegar a los datos.*

### 5.4 Conectividad entre los componentes

En lugar de una VPN híbrida, el diseño usa seis enlaces entre servicios de la propia plataforma y de GitHub:

<img src="../../img_s1-e1-analisis-y-diseno-de-red/media/dc046a94c8394070a8eb213b4d928dea06b41853.png" title="Figura 5.4 — Enlaces de conectividad del despliegue (numerados como en la tabla 2)." style="width:6.25in;height:1.5in" alt="Figura 5.4 — Enlaces de conectividad del despliegue (numerados como en la tabla 2)." />

*Figura 5.4 — Enlaces de conectividad del despliegue (numerados como en la tabla 2).*

**Tabla 2. Parámetros de cada enlace.**

| **N.º** | **Origen → destino**      | **Protocolo · puerto** | **Cifrado / autenticación**                                    | **Estado**   |
|---------|---------------------------|------------------------|----------------------------------------------------------------|--------------|
| ①       | Usuario → borde de Render | HTTPS · 443            | TLS administrado por Render \[1\]                              | En operación |
| ②       | Borde → contenedor        | HTTP · 10000           | Dentro de la plataforma                                        | En operación |
| ③       | Contenedor → PostgreSQL   | JDBC · 5432            | Usuario y contraseña por variables de entorno; TLS a verificar | En operación |
| ④       | GitHub → Render           | HTTPS (webhook)        | Integración de la plataforma con el repositorio                | En operación |
| ⑤       | Build → GitHub            | HTTPS (clone)          | Autorización de la integración                                 | En operación |
| ⑥       | Build → contenedor        | Imagen Docker          | Interno de Render                                              | En operación |

### 5.5 Diagrama de flujo de información

**Tabla 3. Flujos de datos con protocolo y puerto.**

| **Origen**          | **Protocolo · puerto**        | **Destino**       | **Datos que viajan**                                            |
|---------------------|-------------------------------|-------------------|-----------------------------------------------------------------|
| Navegador           | HTTPS · 443                   | Web Service (SPA) | Archivos estáticos (index.html, JS, CSS)                        |
| Navegador           | HTTPS · 443 · /api/auth/login | API REST          | Credenciales de acceso; retorna token JWT                       |
| Navegador           | HTTPS · 443 · /api/\*\*       | API REST          | Operaciones de negocio con token en el encabezado Authorization |
| API REST            | JDBC · 5432                   | PostgreSQL        | Consultas y transacciones del negocio                           |
| DemoDataInitializer | JDBC · 5432                   | PostgreSQL        | Poblado inicial de datos de demostración al arrancar            |
| GitHub              | HTTPS                         | Render            | Notificación de nuevo commit (webhook)                          |
| Contenedor          | stdout                        | Logs de Render    | Registros de la aplicación                                      |

Como el frontend se sirve desde el mismo dominio que la API, el navegador no realiza llamadas entre orígenes y no hace falta configurar CORS \[informe de despliegue, sección 5.1\].

### 5.6 Matriz de componentes

Todo elemento dibujado aparece aquí. Como la plataforma asigna las IP, la columna «Dirección planeada» muestra el identificador estable (URL, puerto o repositorio).

| **ID** | **Componente**             | **Dirección planeada**        | **Zona** | **Función**                                   | **Crit.** |
|--------|----------------------------|-------------------------------|----------|-----------------------------------------------|-----------|
| USR    | Navegador del usuario      | Dinámica                      | 1        | Cliente de la SPA                             | Alta      |
| EDG    | Borde de Render (TLS)      | finanzasepe.onrender.com :443 | 2        | Cifra y enruta                                | Alta      |
| WS01   | Web Service Docker         | :10000 (PORT)                 | 3        | Sirve SPA y API                               | Alta      |
| JVM    | Spring Boot / Java 21      | Interno                       | 3        | Reglas de negocio, JPA, seguridad             | Alta      |
| SEC    | JwtAuthenticationFilter    | Interno                       | 3        | Autentica y asigna autoridades                | Alta      |
| INI    | DemoDataInitializer        | Interno                       | 3        | Poblado de datos al arrancar                  | Baja      |
| CFG    | Variables de entorno       | Panel de Render               | 3        | SPRING_DATASOURCE_URL/USERNAME/PASSWORD, PORT | Alta      |
| DB01   | PostgreSQL 18 administrado | \[host del panel\] :5432      | 4        | Persistencia y auditoría                      | Alta      |
| GIT    | Repositorio GitHub         | Jaqueline2101/FinanzasEPE     | 5        | Código y webhook                              | Media     |
| BLD    | Build Docker multi-etapa   | maven:3.9-eclipse-temurin-21  | 5        | Compila y empaqueta                           | Media     |
| RUN    | Imagen de ejecución        | eclipse-temurin:21-jre-alpine | 3        | Contenedor ligero                             | Alta      |
| LOG    | Registros de Render        | Panel de Render               | 3        | Diagnóstico                                   | Media     |

<img src="../../img_s1-e1-analisis-y-diseno-de-red/media/66c0d601c72cf08881ed45325ff1fcbba1d1b148.png" title="Figura 5.5 — Panel de Render: Web Service FinanzasEPE (Docker, plan Free, rama main, URL pública, historial de despliegues)." style="width:5.41667in;height:2.59375in" alt="Figura 5.5 — Panel de Render: Web Service FinanzasEPE (Docker, plan Free, rama main, URL pública, historial de despliegues)." />

*Figura 5.5 — Panel de Render: Web Service FinanzasEPE (Docker, plan Free, rama main, URL pública, historial de despliegues).*

<img src="../../img_s1-e1-analisis-y-diseno-de-red/media/f5ae9369df6aa528984934a24725396a22361b27.png" title="Figura 5.6 — Panel de Render: base de datos finanzas-db (PostgreSQL 18, plan Free, región Oregon, aviso de vencimiento el 17/12/2026)." style="width:5.41667in;height:2.69792in" alt="Figura 5.6 — Panel de Render: base de datos finanzas-db (PostgreSQL 18, plan Free, región Oregon, aviso de vencimiento el 17/12/2026)." />

*Figura 5.6 — Panel de Render: base de datos finanzas-db (PostgreSQL 18, plan Free, región Oregon, aviso de vencimiento el 17/12/2026).*

## 6. Diseño de conectividad entre componentes

La plantilla pide una conectividad híbrida entre dos entornos. Este proyecto tiene un solo entorno de ejecución (Render), por lo que no hay VPN, pares, redes anunciadas ni túneles IPsec que diseñar, y no se inventan. La conectividad relevante es la de la tabla 2. Como evaluación de alternativa, una VPN site-to-site solo se justificaría si el sistema debiera comunicarse con una red privada propia (por ejemplo, un servidor local del laboratorio); hoy no existe ese requisito y queda fuera del alcance.

Matriz de pruebas de conectividad planeadas, para ejecutar durante la implementación y el monitoreo:

| **Prueba de conectividad planeada**                    | **Resultado esperado**                         | **Estado**                                          |
|--------------------------------------------------------|------------------------------------------------|-----------------------------------------------------|
| Internet → https://finanzasepe.onrender.com            | Permitido; responde la aplicación «Finanzas»   | Verificada el 17/11/2026 (título de la página)      |
| Internet → http:// (puerto 80)                         | Redirección a HTTPS o rechazo                  | Planeada                                            |
| Internet → API protegida sin token (/api/\*\*)         | Denegado: 401 o 403                            | Planeada                                            |
| Internet → /api/auth/login con credenciales válidas    | Permitido: devuelve token JWT                  | Planeada                                            |
| Contenedor → PostgreSQL :5432                          | Permitido con las credenciales configuradas    | Planeada (evidencia indirecta: la aplicación opera) |
| Internet → PostgreSQL :5432 desde una IP no autorizada | Denegado según las reglas de acceso de la base | Planeada (requiere revisar el panel)                |
| Primera petición tras 15 minutos sin tráfico           | Atendida tras ≤ 60 s de arranque en frío       | Planeada                                            |
| Push a main → nuevo despliegue                         | Build y reinicio automáticos                   | Descrita en el informe de despliegue                |

## 7. Diseño de seguridad perimetral

### 7.1 Firewall y perímetro

El equipo no configura un firewall propio. El perímetro lo constituyen el borde de Render (HTTPS sobre un dominio con certificado administrado) y la restricción de la base de datos. Política por defecto de diseño: denegar todo excepto 443 hacia el Web Service. Criterios de reglas a verificar en el panel de Render: el servicio solo escucha en el puerto asignado (10000) y la base solo acepta conexiones de la aplicación; en Render los servicios gratuitos no pueden recibir tráfico de la red privada ni usar puertos reservados, y el correo saliente por los puertos 25, 465 y 587 está bloqueado \[1\].

### 7.2 «DMZ»: el Web Service como único punto expuesto

Sin red propia no existe una DMZ física. El Web Service cumple su rol: es el único componente alcanzable desde Internet y concentra los controles (TLS, autenticación y autorización). Puede alcanzar la base de datos únicamente por 5432 y no tiene acceso al repositorio ni a otros sistemas. La base de datos no debe tener otra vía de acceso público que la estrictamente necesaria para administración, restringida por IP.

### 7.3 Segmentación planeada

| **Origen → destino**                        | **Resultado esperado**                           |
|---------------------------------------------|--------------------------------------------------|
| Internet → Web Service (443)                | PERMITIDO                                        |
| Internet → API protegida sin token          | DENEGADO (401/403)                               |
| Internet → PostgreSQL (5432)                | DENEGADO (salvo IP de administración autorizada) |
| Web Service → PostgreSQL (5432)             | PERMITIDO (solo este puerto)                     |
| Usuario sin permiso → operación restringida | DENEGADO (@PreAuthorize)                         |
| Administrador → operaciones granulares      | PERMITIDO (ver hallazgo S-02)                    |
| GitHub → Web Service                        | Solo despliegue por webhook; sin acceso a datos  |

#### Hallazgos de diseño sobre el estado actual (a corregir en la implementación)

| **ID** | **Hallazgo**                                                                                                                                                             | **Riesgo**                                        | **Acción de diseño**                                                                                     |
|--------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| S-01   | El informe de despliegue publica un usuario administrador de demostración con contraseña simple. No se reproduce en este documento                                       | Alto: acceso total a la producción                | Cambiar la contraseña, usar una credencial no trivial y retirarla de los documentos                      |
| S-02   | JwtAuthenticationFilter inyecta todas las autoridades al administrador, y las directivas del frontend lo tratan como superusuario                                        | Medio: sin mínimo privilegio                      | Cargar los permisos desde la base y limitar «ACCESO_TOTAL»                                               |
| S-03   | DemoDataInitializer se ejecuta en el arranque de producción                                                                                                              | Medio: datos de demostración mezclados con reales | Activarlo solo con un perfil de demostración                                                             |
| S-04   | Plan gratuito: el panel avisa que la base finanzas-db se elimina el 17/12/2026 si no se pasa a un plan de pago; el plan gratuito no ofrece respaldos administrados \[1\] | Crítico: pérdida total de datos en 30 días        | Respaldo externo (pg_dump) antes del 17/12/2026 y migración a un plan de pago antes de usar datos reales |
| S-05   | Servicio de instancia única que se suspende por inactividad \[1\]                                                                                                        | Medio: indisponibilidad                           | Sonda de disponibilidad y/o instancia de pago                                                            |

## 8. Diseño de detección y monitoreo

En una PaaS el equipo no puede instalar un IDS/IPS ni un honeypot de red, porque no controla las interfaces ni el tráfico de la plataforma. La detección se diseña, por lo tanto, a nivel de aplicación y de servicio. Se distingue lo que existe de lo que se planifica:

| **Tecnología**                   | **Estado**                                                                                      | **Ubicación**                         | **Qué debería detectar**                                    | **Cómo se validará**                              |
|----------------------------------|-------------------------------------------------------------------------------------------------|---------------------------------------|-------------------------------------------------------------|---------------------------------------------------|
| Tabla de auditoría lab_auditoria | Existe (eventos de login, emisión de comprobantes, apertura de caja, autorización de descuento) | PostgreSQL (zona 4)                   | Accesos y operaciones sensibles, con usuario y hora         | Consulta de eventos tras pruebas controladas      |
| Logs de Render y de Spring Boot  | Existe (salida estándar)                                                                        | Zona 3                                | Errores 5xx, fallos de arranque, intentos de login fallidos | Revisión de logs tras provocar fallos controlados |
| Sonda de disponibilidad externa  | Planificada                                                                                     | Fuera de Render, hacia la URL pública | Caídas y arranques en frío                                  | Alerta ante una caída simulada                    |
| Spring Boot Actuator (health)    | Planificada                                                                                     | Zona 3                                | Estado de la aplicación y de la conexión a la base          | Consulta de /actuator/health                      |
| Detección de abuso en login      | Planificada                                                                                     | Zona 3                                | Fuerza bruta, bloqueo por intentos                          | Intentos fallidos repetidos en entorno de prueba  |
| IDS/IPS y honeypot de red        | No aplicable en PaaS                                                                            | —                                     | —                                                           | Alternativa futura: WAF delante del servicio      |

La validación se hará en la etapa de implementación con pruebas controladas y sin afectar datos reales.

## 9. Consideraciones de escalabilidad y disponibilidad

El diseño actual es de instancia única, adecuado para una demostración y para pocos usuarios, y presenta límites que deben conocerse:

| **Límite o riesgo**                                    | **Evidencia**                                                                                                                    | **Opción de crecimiento**                                                                 |
|--------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------|
| Memoria de 512 MB y JVM ajustada                       | Se eliminó un error 502 recalibrando la JVM (-Xmx280m, -Xss512k, SerialGC) y quitando la transacción global del poblado de datos | Instancia de mayor tamaño; revisar el consumo con las métricas de Render                  |
| Suspensión tras 15 minutos de inactividad              | Documentación de Render \[1\]                                                                                                    | Instancia de pago o sonda de disponibilidad                                               |
| Base de datos gratuita con vencimiento y sin respaldos | Panel de Render (aviso de eliminación el 17/12/2026) y documentación \[1\]                                                       | Plan de pago y respaldo externo programado                                                |
| Sistema de archivos efímero                            | Documentación de Render \[1\]                                                                                                    | No guardar archivos en el contenedor; usar almacenamiento externo                         |
| Frontend y API en un solo servicio                     | Informe de despliegue, sección 5.1                                                                                               | Separar el frontend como sitio estático si crece el tráfico                               |
| Una sola instancia                                     | Arquitectura actual                                                                                                              | Réplicas del Web Service; las notificaciones en tiempo real requerirían un broker externo |

El crecimiento no compromete la segmentación: las zonas de la sección 5.2 se mantienen y solo cambian las capacidades de cada componente.

## 10. Conclusiones y siguientes pasos

Se documentó el despliegue real de Finanzas EPE en Render como un diseño verificable: dos componentes administrados (Web Service Docker y PostgreSQL), una cadena de entrega automática desde GitHub y cinco zonas lógicas con reglas de comunicación. La plantilla de red se adaptó sin inventar elementos: no hay VLAN, VPN, firewall propio ni IDS, y el documento lo declara.

Pendiente para los entregables siguientes: (1) ejecutar la matriz de pruebas de conectividad; (2) cerrar los hallazgos S-01 a S-05; (3) implementar respaldos externos y sonda de disponibilidad; (4) completar la región del Web Service y las reglas de acceso de la base de datos desde el panel de Render; (5) evidenciar la operación con capturas de logs y métricas.

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<tbody>
<tr class="odd">
<td><p><strong>[ Evidencia de sustentación ]</strong></p>
<p><em>Fecha, docente evaluador, observaciones y firma (a completar por el equipo)</em></p></td>
</tr>
</tbody>
</table>

## 11. Referencias

**\[1\]** Render, «Deploy for Free», Render Docs. \[En línea\]. Disponible: https://render.com/docs/free (consultado el 17/11/2026).

**\[2\]** Equipo del proyecto, «Informe Técnico Integral: Proceso de Desarrollo, Solución de Errores, Poblado Masivo de Datos y Despliegue en Render», Finanzas EPE, versión de producción 1.0.4, 2026.

**\[3\]** ISO/IEC, «ISO/IEC 27001:2022 — Information security management systems — Requirements», 2022.

**\[4\]** ISO/IEC, «ISO/IEC 27002:2022 — Information security controls», 2022.

**\[5\]** Docker Inc., «Multi-stage builds», Docker Docs. \[En línea\]. Disponible: https://docs.docker.com/build/building/multi-stage/

**\[6\]** Spring, «Spring Security Reference». \[En línea\]. Disponible: https://docs.spring.io/spring-security/reference/

**\[7\]** Congreso de la República del Perú, «Ley N.º 29733, Ley de Protección de Datos Personales», 2011.

---

## Material complementario

- [Diagramas (ZIP: png y dot)](../descargas/ce03/CE0311_S1_E1_Diagramas.zip)
