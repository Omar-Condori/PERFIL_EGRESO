# CE0312-CE0313 - S2 - E2: Implementación, Testing y Monitoreo de Red

**Equipo:** Omar Condori Pachauri (201810211) · Merry Elizabeth Flores Llanque (202211953) · Jaqueline Huahuaccapa Ccama (202210712) — Ciclo 10.

## 1. Resumen ejecutivo de la implementación

**Diseño de referencia.** Entregable 1 de Infraestructura: cinco zonas lógicas y seis enlaces entre usuarios, borde de Render, Web Service, PostgreSQL y GitHub. Este documento demuestra qué parte de ese diseño está implementada, cómo se probó y qué falta. No hay VPN, firewall propio, IDS/IPS ni honeypot de red: el proyecto se ejecuta en una plataforma administrada (Render), y se implementó el equivalente real de cada elemento.

**Implementado y verificado.** Los seis enlaces del diseño están en operación (HTTPS en el borde, puerto 10000 al contenedor, JDBC a la base, webhook de GitHub, build y despliegue). Se verificaron desde Internet el acceso HTTPS, un endpoint de salud público que informa «UP», que Actuator expone solo el estado de salud, que no se exponen las variables de entorno y que la API rechaza una consulta sin token. En el panel se verificaron el disparo automático de despliegues, el botón de Rollback y que un despliegue fallido no reemplaza al servicio activo.

**Resultados de pruebas.** Del plan de 16 pruebas, 8 pasan, 1 queda como observación y 7 están pendientes de ejecución con el script del repositorio técnico (pruebas_conectividad.sh). No se declara como probado nada que no se ejecutó.

**Hallazgos.** La documentación de la API (Swagger UI) parece publicada en producción; la base de datos vence el 17/12/2026; el servicio se suspende por inactividad; faltan monitoreo externo y alertas; y los accesos de la base de datos aún deben verificarse en el panel. Los hallazgos se vinculan con el procedimiento de incidentes del SGSI.

|                                                                                                                                                                                                                                                                                             |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Cumplimiento del diseño.** Elementos del diseño: 17. Implementados: 11. Parciales: 4. No implementados o no aplicables en PaaS: 2 (IDS/IPS y honeypot de red). El detalle está en la sección 2. Las capturas y salidas sensibles se entregan sin contraseñas, tokens ni datos personales. |

## 2. Trazabilidad con el diseño de red (Entregable 1)

| **Elemento del diseño**                          | **Estado**   | **Evidencia o cambio justificado**                                                     |
|--------------------------------------------------|--------------|----------------------------------------------------------------------------------------|
| Zona 1 Pública (Internet)                        | Implementado | URL pública accesible (PT-01)                                                          |
| Zona 2 Borde: TLS y enrutamiento                 | Implementado | HTTPS activo en finanzasepe.onrender.com; TLS lo administra Render                     |
| Zona 3 Aplicación: contenedor en el puerto 10000 | Implementado | Informe de despliegue; servicio «Live» (E2, fig. 7.1)                                  |
| Zona 4 Datos: PostgreSQL                         | Implementado | finanzas-db disponible (E2, fig. 7.2); versión 18 (el diseño decía 16: se corrigió)    |
| Zona 5 Entrega: GitHub + build                   | Implementado | Despliegues Auto-Deploy y manuales (PT-13)                                             |
| Enlace ① HTTPS 443                               | Implementado | PT-01                                                                                  |
| Enlace ② HTTP 10000 borde→contenedor             | Implementado | Variable PORT=10000                                                                    |
| Enlace ③ JDBC 5432                               | Implementado | La aplicación opera con la base; TLS y reglas por verificar                            |
| Enlaces ④⑤⑥ GitHub, build, imagen                | Implementado | Historial de 30 despliegues                                                            |
| Autenticación JWT y @PreAuthorize                | Implementado | PT-05; informe de despliegue, sección 3.2                                              |
| Registro de auditoría lab_auditoria              | Implementado | Informe de despliegue, sección 4                                                       |
| Logs y métricas de Render                        | Parcial      | Menú Events/Logs/Metrics visible; capturas de contenido pendientes                     |
| Actuator /health (planificado en el diseño)      | Implementado | Responde UP públicamente (PT-03). Cambio: el diseño lo daba por planificado; ya existe |
| Sonda de disponibilidad externa                  | Parcial      | Script entregado; servicio de sonda sin contratar                                      |
| Restricción de acceso externo a la base          | Parcial      | Por verificar en el panel (PT-11)                                                      |
| Segmentación de zonas y reglas del diseño        | Parcial      | Lógica por capas; sin VLAN por naturaleza de la PaaS                                   |
| Detección de abuso en login                      | Parcial      | Pendiente PT-16                                                                        |
| IDS/IPS y honeypot de red                        | No aplicable | La PaaS no permite instalarlos; alternativa futura: WAF                                |
| VPN híbrida                                      | No aplicable | No existe entorno local de producción                                                  |

Cambios respecto al diseño: (1) versión de PostgreSQL 18; (2) Actuator ya activo; (3) se agrega el hallazgo de Swagger UI, no previsto en el diseño.

## 3. Implementación de la conectividad

Sin VPN, la conectividad implementada es la cadena de enlaces del Entregable 1. Qué se configuró, dónde y con qué resultado:

| **Enlace**                | **Qué se configuró**                                                                | **Dónde**                         | **Resultado**                                            |
|---------------------------|-------------------------------------------------------------------------------------|-----------------------------------|----------------------------------------------------------|
| ① Usuario → borde         | Dominio finanzasepe.onrender.com con TLS administrado                               | Render (automático)               | HTTPS responde (PT-01)                                   |
| ② Borde → contenedor      | Aplicación escucha en el puerto 10000 (PORT=10000)                                  | Variable de entorno y Spring Boot | Servicio «Live»                                          |
| ③ Contenedor → PostgreSQL | SPRING_DATASOURCE_URL/USERNAME/PASSWORD (solo nombres en este documento)            | Panel de Render \> Environment    | La aplicación consulta datos y siembra datos al arrancar |
| ④ GitHub → Render         | Servicio conectado al repositorio Jaqueline2101/FinanzasEPE, rama main; Auto-Deploy | Panel de Render                   | Despliegues con disparador Auto-Deploy                   |
| ⑤ Build → GitHub          | Clonado del código en el build Docker                                               | Render                            | Build de 5 min 50 s a 5 min 59 s                         |
| ⑥ Build → contenedor      | Dockerfile multi-etapa; JVM -Xmx280m, -Xss512k                                      | Repositorio                       | Servicio estable tras el ajuste                          |

<img src="../../img_s2-e2-implementacion-testing-monitoreo/media/dc046a94c8394070a8eb213b4d928dea06b41853.png" title="Figura 3.1 — Enlaces implementados (numeración del Entregable 1)." style="width:6.25in;height:1.5in" alt="Figura 3.1 — Enlaces implementados (numeración del Entregable 1)." />

*Figura 3.1 — Enlaces implementados (numeración del Entregable 1).*

Rutas activas: Render enruta por nombre de dominio al Web Service; no hay tablas de rutas administradas por el equipo. Pruebas bidireccionales: el sentido usuario → servicio se probó (PT-01 a PT-05); el sentido servicio → base de datos se prueba indirectamente porque la aplicación opera.

## 4. Implementación de la seguridad perimetral

| **Aspecto**          | **Implementación real**                                                                                                                                                      | **Estado**   |
|----------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------|
| Producto             | Borde de Render (TLS y enrutamiento) más Spring Security en el contenedor; no hay firewall del equipo                                                                        | En operación |
| Interfaces y zonas   | Entrada pública 443; contenedor en 10000; base en 5432 (zonas 1 a 4)                                                                                                         | En operación |
| Política por defecto | Solo el Web Service es público; la API exige token (PT-05); /actuator expone solo health; las variables no se exponen (PT-04)                                                | Verificado   |
| Reglas configuradas  | Reglas de acceso de la base de datos: por revisar en el panel de Render; sin reglas de red a nivel de servicio (no soportado en el plan gratuito para tráfico privado \[1\]) | Pendiente    |
| Zona «DMZ»           | El Web Service cumple ese rol: único punto expuesto, con los controles de la aplicación                                                                                      | En operación |
| Cifrado              | TLS en el borde; JWT firmado con RSA                                                                                                                                         | En operación |

<img src="../../img_s2-e2-implementacion-testing-monitoreo/media/66c0d601c72cf08881ed45325ff1fcbba1d1b148.png" title="Figura 4.1 — Panel de Render: Web Service FinanzasEPE (Docker, Free, rama main, Live)." style="width:4.89583in;height:2.34375in" alt="Figura 4.1 — Panel de Render: Web Service FinanzasEPE (Docker, Free, rama main, Live)." />

*Figura 4.1 — Panel de Render: Web Service FinanzasEPE (Docker, Free, rama main, Live).*

## 5. Pruebas de segmentación

Pruebas permitido/denegado del diseño, adaptadas a la arquitectura real. Los resultados «obtenidos» se leyeron el 18/11/2026 con una herramienta de lectura web que no muestra el código HTTP; por eso las pruebas que exigen un código exacto quedan para el script del repositorio técnico.

| **Origen → destino**                        | **Esperado**       | **Obtenido**                                          | **Estado**     |
|---------------------------------------------|--------------------|-------------------------------------------------------|----------------|
| Internet → Web Service (HTTPS)              | PERMITIDO          | Responde «Finanzas»                                   | PASA           |
| Internet → API protegida sin token          | DENEGADO           | Rechazo de cliente (4xx)                              | PASA (parcial) |
| Internet → /actuator/env                    | DENEGADO           | No expone variables (devuelve la SPA)                 | PASA           |
| Internet → /actuator/health                 | PERMITIDO (mínimo) | status UP                                             | PASA           |
| Internet → Swagger UI                       | DENEGADO           | La página responde con título «Swagger UI»            | OBSERVACIÓN    |
| Internet → PostgreSQL 5432                  | DENEGADO           | Sin ejecutar (necesita el host externo)               | PENDIENTE      |
| Usuario sin permiso → operación restringida | DENEGADO           | Sin ejecutar (requiere un usuario de prueba)          | PENDIENTE      |
| Administrador → operaciones granulares      | PERMITIDO          | Documentado en el informe de despliegue (sección 3.2) | Documental     |

## 6. Detección y alertas

| **Tecnología**     | **Implementación real**                                                                                                                 | **Eventos que detecta**          | **Evidencia de punta a punta**                         |
|--------------------|-----------------------------------------------------------------------------------------------------------------------------------------|----------------------------------|--------------------------------------------------------|
| lab_auditoria      | Tabla de auditoría con eventos de login, emisión de comprobantes, apertura de caja y autorización de descuentos (informe de despliegue) | Accesos y operaciones sensibles  | Consulta de eventos tras una prueba (pendiente, PT-16) |
| Logs de Render     | Salida estándar del contenedor en el panel                                                                                              | Errores de arranque y 5xx        | Captura de logs (pendiente)                            |
| /actuator/health   | Endpoint público que responde UP                                                                                                        | Aplicación y estado de readiness | Verificado (PT-03)                                     |
| Sonda externa      | Script sonda_disponibilidad.sh y plan alertas_sonda.md                                                                                  | Caídas y arranque en frío        | Sin contratar el servicio                              |
| IDS/IPS y honeypot | No aplican en PaaS                                                                                                                      | —                                | Alternativa: WAF delante del servicio                  |
| SIEM               | No existe; los logs de Render se revisan en el panel                                                                                    | —                                | Mejora futura                                          |

## 7. Monitoreo y visibilidad

El panel de Render ofrece, en el menú MONITOR de ambos servicios, las secciones Events, Logs y Metrics (visibles en las capturas). La centralización es la del propio panel; no hay un agregador externo. Dashboards: Metrics de Render (CPU y memoria del servicio). Correlación de eventos: manual, cruzando logs de Render con lab_auditoria. Conservación de evidencias: capturas fechadas y exportación de los logs relevantes a la carpeta logs/ del repositorio técnico.

| **Chequeo propuesto**           | **Intervalo**    | **Alerta**                           |
|---------------------------------|------------------|--------------------------------------|
| /actuator/health contiene «UP»  | 5 min            | Dos fallos seguidos                  |
| Memoria del servicio (Metrics)  | Revisión semanal | Uso sostenido \> 85 %                |
| Antigüedad del respaldo         | Diario           | \> 26 h                              |
| Vencimiento de la base gratuita | Semanal          | Quedan menos de 14 días (17/12/2026) |

Una sonda cada 5 minutos mantiene el servicio despierto y consume unas 720 de las 750 horas gratuitas mensuales \[1\].

## 8. Plan y resultados de pruebas técnicas

| **ID** | **Prueba**                                         | **Esperado**                              | **Obtenido**                                                                 | **Estado**     |
|--------|----------------------------------------------------|-------------------------------------------|------------------------------------------------------------------------------|----------------|
| PT-01  | Internet → https://finanzasepe.onrender.com        | Permitido (200, SPA «Finanzas»)           | Responde la aplicación «Finanzas»                                            | PASA           |
| PT-02  | GET /actuator                                      | Superficie mínima de Actuator             | Expone solo health y health/{\*path}                                         | PASA           |
| PT-03  | GET /actuator/health                               | Aplicación UP                             | {"groups":\["liveness","readiness"\],"status":"UP"}                          | PASA           |
| PT-04  | GET /actuator/env                                  | No expuesto                               | Devuelve la SPA «Finanzas», no variables                                     | PASA           |
| PT-05  | GET /api/usuarios sin token                        | Denegado 401/403                          | Rechazado (error de cliente 4xx); código exacto no visible en la herramienta | PASA (parcial) |
| PT-06  | GET /swagger-ui.html en producción                 | Denegado o no publicado                   | La página responde con título «Swagger UI»                                   | OBSERVACIÓN    |
| PT-07  | GET /v3/api-docs                                   | Denegado o no publicado                   | La consulta devolvió un error del servidor; no concluyente                   | PENDIENTE      |
| PT-08  | POST /api/auth/login con credenciales inválidas    | 401/403/400                               | —                                                                            | PENDIENTE      |
| PT-09  | POST /api/auth/register sin token                  | 401/403                                   | La consulta GET falló; la prueba POST no se ejecutó                          | PENDIENTE      |
| PT-10  | http:// (puerto 80)                                | Redirección a HTTPS                       | —                                                                            | PENDIENTE      |
| PT-11  | Internet → PostgreSQL 5432 desde IP no autorizada  | Denegado                                  | —                                                                            | PENDIENTE      |
| PT-12  | Arranque en frío tras 15 min sin tráfico           | ≤ 60 s                                    | —                                                                            | PENDIENTE      |
| PT-13  | Push a main dispara despliegue automático          | Build y reinicio automáticos              | Despliegues con disparador «Auto-Deploy» (4901fa3, f4224b5, e61f181)         | PASA           |
| PT-14  | Rollback disponible                                | Botón de reversión en despliegues previos | Botón «Rollback» visible                                                     | PASA           |
| PT-15  | Despliegue fallido no reemplaza el servicio activo | El commit activo no cambia                | f4224b5 falló en 33,7 s; el servicio sigue «Live» con 4901fa3                | PASA           |
| PT-16  | Login fallido genera evento de auditoría           | Registro en lab_auditoria o logs          | —                                                                            | PENDIENTE      |

El mismo plan, con las fórmulas de seguimiento, está en la hoja «PlanPruebas» de Infra_S2_Matrices.xlsx. Para cerrar las pruebas pendientes: ejecutar scripts/pruebas_conectividad.sh (con DB_HOST para PT-11) y pegar el resultado en la hoja.

## 9. Incidencias detectadas

| **ID** | **Incidencia**                                                    | **Clasificación**      | **Contención y solución**                                                          | **Lección aprendida**                                               |
|--------|-------------------------------------------------------------------|------------------------|------------------------------------------------------------------------------------|---------------------------------------------------------------------|
| I-01   | Error 502 persistente en finanzasepe.onrender.com                 | Media (servicio caído) | JVM reajustada (-Xmx280m, -Xss512k) y se retiró la transacción global del arranque | Medir memoria antes de fijar topes; suma de topes hoy 508 de 512 MB |
| I-02   | Despliegue f4224b5 fallido a los 33,7 s                           | Baja                   | El servicio siguió «Live» con la versión anterior; se corrigió después             | Probar antes del push (CI); causa exacta por documentar             |
| I-03   | Botones CRUD denegados (403) para el administrador                | Media                  | Se inyectaron todas las autoridades al administrador en el filtro JWT              | La solución rompió el mínimo privilegio (R-03)                      |
| I-04   | Auto-despliegues cancelados a mano al lanzar despliegues manuales | Baja                   | Elegir un solo modo de despliegue                                                  | Evitar minutos de build desperdiciados                              |
| I-05   | Credencial de administrador publicada en un informe               | Alta                   | Cambiarla y retirarla del documento (R-02)                                         | Los informes no deben contener secretos                             |

Vínculo con el SGSI: cada incidencia sigue el procedimiento de la sección 11 del Entregable del SGSI (controles 5.24 a 5.27).

## 10. Evidencias técnicas

| **ID** | **Evidencia**                                           | **Propósito**                                              | **Ubicación**                       |
|--------|---------------------------------------------------------|------------------------------------------------------------|-------------------------------------|
| E-01   | Captura del Web Service FinanzasEPE                     | Muestra Docker, plan Free, repositorio, rama y despliegues | Figura 4.1 / Entregable 2, fig. 7.1 |
| E-02   | Captura de finanzas-db                                  | Muestra PostgreSQL 18, Free, Oregon y vencimiento          | Entregable 2, fig. 7.2              |
| E-03   | Lecturas de /actuator, /actuator/health y /actuator/env | Superficie expuesta de Actuator                            | Sección 8 (PT-02 a PT-04)           |
| E-04   | scripts/pruebas_conectividad.sh                         | Reproducir las pruebas permitido/denegado                  | Repositorio técnico                 |
| E-05   | Hoja PlanPruebas                                        | Seguimiento de resultados                                  | Infra_S2_Matrices.xlsx              |
| E-06   | Capturas de Logs y Metrics de Render                    | Visibilidad operativa                                      | Pendiente                           |

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<tbody>
<tr class="odd">
<td><p><strong>[ Captura: Logs de Render con un evento de login y una petición rechazada ]</strong></p>
<p><em>Pendiente; sin tokens, contraseñas ni datos personales</em></p></td>
</tr>
</tbody>
</table>

## 11. Conclusiones, limitaciones y mejoras

El diseño del Entregable 1 se implementó en lo que la plataforma permite: seis enlaces operativos, perímetro de TLS y autenticación, y una superficie de Actuator mínima. Las limitaciones son claras: no hay VPN ni IDS por naturaleza de la PaaS, la lectura web no muestra códigos HTTP, y quedan pruebas por ejecutar (acceso a la base, arranque en frío, login fallido).

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

### Referencias

**\[1\]** Render, «Deploy for Free», Render Docs. \[En línea\]. Disponible: https://render.com/docs/free (consultado el 17/11/2026).

**\[2\]** Equipo del proyecto, «Informe Técnico Integral … Despliegue en Render», Finanzas EPE, v1.0.4, 2026.

**\[3\]** ISO/IEC, «ISO/IEC 27002:2022 — Information security controls», 2022.

**\[4\]** Spring, «Spring Boot Actuator». \[En línea\]. Disponible: https://docs.spring.io/spring-boot/reference/actuator/

**\[5\]** Equipo del proyecto, «Análisis y Diseño del Despliegue», Entregable 1 de Infraestructura, 2026.

---

## Material complementario

- [Repositorio técnico: scripts, logs y plantillas (ZIP)](../descargas/ce03/S2_Repositorio_Tecnico.zip)
- [Plan de pruebas en hoja de cálculo (XLSX)](../descargas/ce03/S2_Matrices_Activos_Riesgos_SoA_Pruebas.xlsx)
