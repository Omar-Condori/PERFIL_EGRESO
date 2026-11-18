# CE0332-CE0333 - S2 - E3: Implementación, Gestión y Control de Centro de Datos

**Equipo:** Omar Condori Pachauri (201810211) · Merry Elizabeth Flores Llanque (202211953) · Jaqueline Huahuaccapa Ccama (202210712) — Ciclo 10.

## 1. Resumen ejecutivo

**Qué se implementó.** Del diseño del Entregable 2 (centro de datos en la nube) está en operación un Web Service Docker y una base PostgreSQL 18 en Render, con despliegue continuo desde GitHub. El centro de datos local y la conexión híbrida con AWS del modelo de referencia no existen: el entorno local es el equipo de desarrollo de cada integrante, y la nube es Render.

**Continuidad.** La base gratuita no ofrece respaldos administrados y se elimina el 17/12/2026 si no se actualiza el plan \[1\]. Por eso se construyó y se probó, en laboratorio, un procedimiento de respaldo y restauración: respaldo de 394 407 bytes en 0,59 s, restauración en 1,04 s y conteos idénticos en cuatro tablas. El ejercicio equivalente sobre la base de Render sigue pendiente porque requiere el host externo de la base y la ejecución del script.

**Tiempos.** Recuperación de la aplicación por rollback o redespliegue: unos 6 minutos (5 min 50 s a 5 min 59 s de build, según el panel). Recuperación de la base: no medida en Render; el objetivo es RTO 2 h y RPO 24 h.

|                                                                                                                                                                                                                                                                                     |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Honestidad de la evidencia.** Se distinguen tres niveles: verificado en el panel o por Internet; probado en laboratorio; pendiente. Las contraseñas, cadenas de conexión y llaves no aparecen en ningún documento ni archivo del repositorio técnico (solo nombres de variables). |

## 2. Trazabilidad con el diseño del centro de datos (Entregable 2)

| **Componente del diseño**                 | **Estado**      | **Evidencia o cambio**                                                       |
|-------------------------------------------|-----------------|------------------------------------------------------------------------------|
| Entorno local (desarrollo, pruebas)       | Implementado    | Equipos del equipo; la base local se usa para el laboratorio de recuperación |
| Servicio de aplicación en Render (Docker) | Implementado    | Panel: Web Service FinanzasEPE, Live                                         |
| Base PostgreSQL gestionada                | Implementado    | Panel: finanzas-db, PostgreSQL 18, Oregon                                    |
| Despliegue continuo desde GitHub          | Implementado    | 30 despliegues en el historial                                               |
| Variables de entorno y secretos           | Implementado    | Solo en el panel de Render; nombres en el repo técnico                       |
| Respaldo diario externo                   | Parcial         | Scripts probados en laboratorio; programación en Render pendiente            |
| Prueba de recuperación                    | Parcial         | Verificada en laboratorio; en Render pendiente                               |
| Alta disponibilidad / réplicas            | No implementado | No se permite en plan gratuito; mejora futura                                |
| Conexión híbrida (VPN) y AWS              | No aplicable    | No existen en el proyecto real                                               |
| Alertas de monitoreo                      | Parcial         | Sonda diseñada; sin contratar                                                |

Cambios justificados: AWS se sustituye por Render, y el entorno on-premise por el entorno local de desarrollo y pruebas.

## 3. Implementación del entorno local

| **Elemento**             | **Implementación**                                                              | **Verificación**                                                           |
|--------------------------|---------------------------------------------------------------------------------|----------------------------------------------------------------------------|
| Base local               | PostgreSQL 16 con el esquema del proyecto (76 tablas) y 3 000 ventas sintéticas | Suite de 30 pruebas SQL aprobadas (Entregable 4 de Ingeniería de Software) |
| Aplicación local         | Spring Boot + Angular con el perfil de desarrollo                               | Documentado en el informe de despliegue                                    |
| Control de configuración | Repositorio GitHub; variables fuera del código                                  | Archivo variables_entorno.example (solo nombres)                           |
| Datos                    | Sintéticos; ningún dato personal real                                           | Declarado en el SGSI                                                       |

Nota: el laboratorio local usa PostgreSQL 16 mientras Render usa 18. pg_dump en formato custom es restaurable hacia versiones iguales o superiores, de modo que la restauración desde 16 hacia 18 es posible, pero el respaldo tomado en Render con una herramienta cliente 16 fallaría; usar un cliente 18.

## 4. Implementación en Render

| **Recurso**        | **Configuración real**                                                                                          | **Fuente**                           |
|--------------------|-----------------------------------------------------------------------------------------------------------------|--------------------------------------|
| Web Service        | Docker, plan Free (512 MB RAM), repositorio Jaqueline2101/FinanzasEPE, rama main, Auto-Deploy, puerto 10000     | Captura del panel                    |
| Base de datos      | finanzas-db, PostgreSQL 18, plan Free, Oregon (US West), vence 17/12/2026; 1 GB, 100 conexiones                 | Captura del panel; Render Docs \[1\] |
| Proyecto / entorno | My project / Production                                                                                         | Captura del panel                    |
| Imagen             | Multi-etapa; runtime eclipse-temurin:21-jre-alpine                                                              | Informe de despliegue                |
| JVM                | -Xmx280m, metaspace 180 MB, code cache 48 MB: 508 de 512 MB; SerialGC; Xss512k                                  | Informe de despliegue                |
| Variables          | SPRING_DATASOURCE_URL, \_USERNAME, \_PASSWORD, PORT; valores no mostrados                                       | Informe de despliegue                |
| Ciclo de ejecución | El servicio se suspende tras 15 min sin tráfico y reinicia en cerca de 1 min; el sistema de archivos es efímero | Render Docs \[1\]                    |

<img src="../../img_s2-e3-centro-de-datos/media/f5ae9369df6aa528984934a24725396a22361b27.png" title="Figura 4.1 — Panel de Render: base de datos finanzas-db." style="width:4.89583in;height:2.4375in" alt="Figura 4.1 — Panel de Render: base de datos finanzas-db." />

*Figura 4.1 — Panel de Render: base de datos finanzas-db.*

<img src="../../img_s2-e3-centro-de-datos/media/b5bf102af3278968f01d2db2a3ca4e7eb3bd4768.png" title="Figura 4.2 — Flujo de construcción y despliegue implementado (Entregable 2)." style="width:3.125in;height:5.65625in" alt="Figura 4.2 — Flujo de construcción y despliegue implementado (Entregable 2)." />

*Figura 4.2 — Flujo de construcción y despliegue implementado (Entregable 2).*

## 5. Gestión de identidades y accesos

| **Capa**                     | **Control implementado**                                                                                | **Estado**             |
|------------------------------|---------------------------------------------------------------------------------------------------------|------------------------|
| Render (workspace)           | Cuenta del equipo; MFA y lista de colaboradores por verificar                                           | Pendiente de evidencia |
| GitHub                       | Repositorio con rama main; protección de rama y MFA por verificar                                       | Pendiente de evidencia |
| Base de datos                | Credenciales como variables de entorno; sin usuarios adicionales                                        | Parcial                |
| Aplicación                   | JWT firmado con RSA y permisos por rol; el administrador recibe todas las autoridades por filtro (R-03) | Parcial                |
| Credenciales de demostración | Credencial del administrador simple y publicada en un informe (R-02)                                    | No conforme            |

Principio aplicado: mínimo privilegio, sin cuentas compartidas y sin secretos en documentos. La acción inmediata es cambiar la contraseña del administrador y activar MFA, con captura de evidencia.

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<tbody>
<tr class="odd">
<td><p><strong>[ Captura: ajustes de MFA / colaboradores de Render y GitHub ]</strong></p>
<p><em>Pendiente; ocultar correos y códigos</em></p></td>
</tr>
</tbody>
</table>

## 6. Inventario final de recursos

| **Recurso**                               | **Tipo**      | **Propietario** | **Evidencia**                       |
|-------------------------------------------|---------------|-----------------|-------------------------------------|
| Web Service FinanzasEPE                   | Servicio      | RI              | Captura del panel                   |
| Base finanzas-db (PostgreSQL 18)          | Base de datos | RD              | Captura del panel; vence 17/12/2026 |
| Repositorio GitHub FinanzasEPE            | Código        | RI              | Panel de Render                     |
| Imagen y Dockerfile                       | Imagen        | RI              | Informe de despliegue               |
| Variables SPRING_DATASOURCE\_\*, PORT     | Configuración | RS              | Informe (solo nombres)              |
| Dominio y TLS finanzasepe.onrender.com    | Red           | RI              | HTTPS verificado                    |
| Endpoint /actuator/health                 | Monitoreo     | RI              | Responde UP                         |
| Scripts de respaldo, restauración y sonda | Operación     | RD / RI         | Repositorio técnico y logs          |

El inventario con criticidad está en la hoja «InventarioFinal» de Infra_S2_Matrices.xlsx. Costos: actualmente 0 USD; el mínimo sostenible es de unos 13 USD al mes (Web Service 7 + PostgreSQL 6) según la tarifa publicada \[2\].

## 7. Continuidad: respaldo, restauración y recuperación

**Tabla 7.1. Procedimiento de respaldo y restauración (repositorio técnico, scripts/).**

| **Paso**               | **Script**          | **Detalle**                                                                                          |
|------------------------|---------------------|------------------------------------------------------------------------------------------------------|
| Respaldo               | respaldo_render.sh  | pg_dump -Fc con la cadena de conexión tomada de una variable de entorno; nunca escrita en el archivo |
| Verificación           | respaldo.sh         | pg_restore --list y suma sha256                                                                      |
| Restauración de prueba | restaurar_prueba.sh | Restaura en una base temporal y compara conteos de tablas clave                                      |
| Retención              | Política            | 7 respaldos diarios y 4 semanales, con copia fuera de Render                                         |

**Tabla 7.2. Registro de la prueba de recuperación en laboratorio (18/11/2026).**

| **Paso**                              | **Resultado**                                       |
|---------------------------------------|-----------------------------------------------------|
| Origen                                | laboratorio_db_demo, PostgreSQL 16 local, 76 tablas |
| Respaldo                              | Archivo de 394 407 bytes en 0,59 s                  |
| Integridad                            | sha256 OK; pg_restore --list OK                     |
| Restauración en base temporal         | 1,04 s                                              |
| venta_registro / venta_detalle        | 3000 = 3000 / 3000 = 3000                           |
| caja_chica_movimiento / lab_auditoria | 400 = 400 / 0 = 0                                   |
| Veredicto                             | RESTAURACION_VERIFICADA                             |

Alcance de la prueba: demuestra que el procedimiento y los scripts funcionan sobre un volumen pequeño con datos sintéticos. No demuestra el tiempo de recuperación en Render ni sobre volúmenes de producción.

**Tabla 7.3. Objetivos frente a lo obtenido.**

| **Servicio**                | **RTO objetivo** | **RPO objetivo** | **Obtenido**                   | **Estado**            |
|-----------------------------|------------------|------------------|--------------------------------|-----------------------|
| Aplicación                  | 15 min           | N/A              | ≈ 6 min (panel de despliegues) | Cumple                |
| Base de datos (laboratorio) | 2 h              | 24 h             | Restauración 1,04 s            | Cumple en laboratorio |
| Base de datos (Render)      | 2 h              | 24 h             | Sin medir                      | Pendiente             |

Riesgo crítico: sin respaldo externo, perder la base equivale a perder los datos (R-01). Con el plan de pago de PostgreSQL se incluye recuperación a un punto en el tiempo \[2\]. Acción: ejecutar respaldo_render.sh antes del 10/12/2026 y decidir el plan.

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<tbody>
<tr class="odd">
<td><p><strong>[ Captura: ejecución del respaldo sobre la base de Render ]</strong></p>
<p><em>Pendiente; sin cadena de conexión visible</em></p></td>
</tr>
</tbody>
</table>

## 8. Monitoreo operativo y alertas

| **Indicador**          | **Fuente**          | **Umbral**                        | **Estado**                           |
|------------------------|---------------------|-----------------------------------|--------------------------------------|
| Salud de la aplicación | /actuator/health    | Distinto de UP dos veces seguidas | Endpoint verificado; sonda pendiente |
| Uso de memoria         | Metrics de Render   | \> 85 % sostenido                 | Por capturar                         |
| Errores                | Logs de Render      | Excepciones o 5xx                 | Por capturar                         |
| Respaldo               | Registro de scripts | \> 26 h                           | Pendiente de programar               |
| Vencimiento de la base | Panel de Render     | \< 14 días                        | Activo: quedan 29 días al 18/11/2026 |

Impacto de la ejecución en memoria: la suma de topes de la JVM es 508 de 512 MB, margen de 4 MB; un pico puede provocar reinicios. Es una causa directa del incidente 502 ya resuelto.

## 9. Control de cambios y versiones

| **Carpeta**     | **Contenido**                                                       |
|-----------------|---------------------------------------------------------------------|
| scripts/        | Pruebas de conectividad, sonda, respaldo, restauración              |
| render/         | variables_entorno.example (solo nombres) y Dockerfile de referencia |
| observabilidad/ | Plan de alertas de la sonda                                         |
| diagramas/      | Diagramas de los Entregables 1 y 2 (png y dot)                      |
| logs/           | Salidas reales de respaldo y restauración                           |

## 10. Evaluación de resultados y conclusiones

El centro de datos en la nube del diseño está implementado en Render, con despliegue continuo y trazabilidad de cambios. La continuidad es la brecha principal: el procedimiento funciona en laboratorio, pero falta aplicarlo sobre Render y programarlo. El riesgo de pérdida de datos es real y fechado.

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

**\[2\]** Render, «Pricing». \[En línea\]. Disponible: https://render.com/pricing (consultado el 17/11/2026).

**\[3\]** Equipo del proyecto, «Informe Técnico Integral … Despliegue en Render», Finanzas EPE, v1.0.4, 2026.

**\[4\]** PostgreSQL Global Development Group, «pg_dump» y «pg_restore», Documentación de PostgreSQL.

**\[5\]** Equipo del proyecto, «Análisis y Diseño del Centro de Datos en la Nube», Entregable 2 de Infraestructura, 2026.

---

## Material complementario

- [Repositorio técnico: scripts de respaldo y restauración, logs (ZIP)](../descargas/ce03/S2_Repositorio_Tecnico.zip)
