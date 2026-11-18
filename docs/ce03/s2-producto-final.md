# Producto final del Semestre 2: Informe de Implementación Integral y Operativa

**Equipo:** Omar Condori Pachauri (201810211) · Merry Elizabeth Flores Llanque (202211953) · Jaqueline Huahuaccapa Ccama (202210712) — Ciclo 10.

## 1. Presentación ejecutiva

**Qué es.** Finanzas EPE es un ERP financiero web (Angular y Spring Boot) desplegado en Render como Web Service Docker más PostgreSQL gestionado, con código en GitHub. Este informe integra los tres entregables del Semestre 2: SGSI, implementación, testing y monitoreo, y gestión del centro de datos en la nube.

**Adaptación.** El diseño del Semestre 1 supone red híbrida con ESXi, AWS y VPN. El proyecto real solo usa Render y GitHub; cada elemento del diseño se implementó en su equivalente real, o se declara no aplicable.

| **Dimensión**    | **Resultado**                                                                                                                 |
|------------------|-------------------------------------------------------------------------------------------------------------------------------|
| Seguridad (SGSI) | 16 riesgos: 2 críticos, 8 altos, 6 medios. SoA de 93 controles: 4 implementados, 40 parciales, 27 planificados, 22 no aplican |
| Pruebas          | 16 pruebas: 8 pasan, 1 observación (Swagger UI público), 7 pendientes                                                         |
| Continuidad      | Respaldo y restauración verificados en laboratorio (1,04 s, conteos idénticos); pendiente sobre Render                        |
| Despliegue       | Servicio Live; 30 despliegues; recuperación de la aplicación ≈ 6 min                                                          |
| Brecha principal | Base gratuita sin respaldos, se elimina el 17/12/2026; credencial de administrador de demostración publicada                  |

|                                                                                                                                                                                                                                          |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Recomendación central.** Antes del 20/11 cambiar la credencial y activar MFA; antes del 10/12 hacer el respaldo externo de la base de Render y medir su restauración; antes del 17/12 decidir el plan de pago (≈ 13 USD/mes) o migrar. |

## 2. Trazabilidad con el diseño técnico del Semestre 1

| **Diseño del Semestre 1**    | **Implementación real (Semestre 2)**                                 | **Estado**   | **Dónde**  |
|------------------------------|----------------------------------------------------------------------|--------------|------------|
| Red con zonas y reglas (E1)  | Cinco zonas lógicas en Render; TLS en el borde, JWT en la aplicación | Implementado | S2-E2 §2–4 |
| Segmentación y firewall      | Sin firewall propio; Web Service como único punto expuesto           | Parcial      | S2-E2 §4–5 |
| VPN híbrida                  | No existe entorno local de producción                                | No aplicable | S2-E2 §2   |
| IDS/IPS y honeypot           | No permitido en PaaS; auditoría y logs como alternativa              | No aplicable | S2-E2 §6   |
| Centro de datos híbrido (E2) | Entorno local de desarrollo más Render como nube                     | Implementado | S2-E3 §2–4 |
| Servicios AWS                | Sustituidos por Web Service y PostgreSQL de Render                   | Adaptado     | S2-E3 §4   |
| Continuidad y recuperación   | Scripts probados en laboratorio; en Render pendiente                 | Parcial      | S2-E3 §7   |
| Gobierno de seguridad        | SGSI ISO 27001/27002 con 16 riesgos y SoA                            | Implementado | S2-E1      |

## 3. Evidencias unificadas

| **ID** | **Evidencia**                                                          | **Nivel**              | **Ubicación**                  |
|--------|------------------------------------------------------------------------|------------------------|--------------------------------|
| U-01   | Capturas del panel: Web Service y finanzas-db                          | Verificado             | S2-E2 fig. 4.1; S2-E3 fig. 4.1 |
| U-02   | Lecturas externas de /actuator, /actuator/health y /actuator/env       | Verificado             | S2-E2 §8                       |
| U-03   | Historial de 30 despliegues, Rollback y despliegue fallido sin impacto | Verificado en panel    | S2-E2 §8, S2-E3 §9             |
| U-04   | Registro de respaldo y restauración (logs)                             | Probado en laboratorio | repo/logs; S2-E3 §7            |
| U-05   | Matrices de activos, riesgos y SoA; plan de pruebas                    | Documental             | Infra_S2_Matrices.xlsx         |
| U-06   | Scripts de pruebas, sonda y respaldo                                   | Reproducible           | Repositorio técnico, scripts/  |
| U-07   | Capturas de Logs, Metrics, MFA y respaldo sobre Render                 | Pendiente              | —                              |

## 4. Riesgos, SoA y declaración ética

<img src="../../img_s2-producto-final/media/38ad1f6063d9f696c4c1c9174d3b498ba66d4f6e.png" title="Figura 4.1 — Riesgo inherente frente a residual." style="width:5.20833in;height:2.47917in" alt="Figura 4.1 — Riesgo inherente frente a residual." />

*Figura 4.1 — Riesgo inherente frente a residual.*

**Tabla 4.1. Riesgos críticos y altos y su tratamiento.**

| **ID** | **Riesgo**                                                 | **Nivel**  | **Acción**                                                                       | **Plazo**  |
|--------|------------------------------------------------------------|------------|----------------------------------------------------------------------------------|------------|
| R-01   | Pérdida total de la base de datos                          | 25 Crítico | Respaldo externo diario y migración a plan de pago antes de la fecha             | 10/12/2026 |
| R-02   | Acceso no autorizado como administrador                    | 20 Crítico | Cambiar la contraseña, usar una no trivial y retirarla de todos los documentos   | 20/11/2026 |
| R-03   | Abuso de privilegios                                       | 12 Alto    | Cargar permisos desde la base y limitar ACCESO_TOTAL                             | 15/12/2026 |
| R-05   | Indisponibilidad del servicio                              | 12 Alto    | Sonda externa de disponibilidad; evaluar plan de pago                            | 30/11/2026 |
| R-06   | Caída por falta de memoria (error 502)                     | 12 Alto    | Medir con Metrics y reducir el tope de metaspace                                 | 30/11/2026 |
| R-07   | Fuga de secretos                                           | 15 Alto    | Secretos solo en variables de entorno; rotar llaves; revisar el historial de git | 30/11/2026 |
| R-09   | Toma de control de las cuentas de Render o GitHub          | 15 Alto    | Activar MFA en ambas cuentas y guardar códigos de recuperación                   | 20/11/2026 |
| R-10   | Creación de usuarios con roles elegidos por el solicitante | 15 Alto    | Restringir el registro a administradores y ignorar roles enviados                | 30/11/2026 |
| R-11   | Tratamiento ilícito de datos personales                    | 10 Alto    | No cargar datos reales hasta cumplir; mantener datos sintéticos                  | Permanente |
| R-12   | Incidentes no detectados                                   | 12 Alto    | Sonda externa y revisión periódica de logs                                       | 30/11/2026 |

SoA completa y justificación de cada control: hoja «SoA» de Infra_S2_Matrices.xlsx y S2-E1 §8. Ética: el equipo adopta el código ACM; pruebas solo sobre el sistema propio, sin datos personales reales y con divulgación responsable (S2-E1 §12).

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<tbody>
<tr class="odd">
<td><p><strong>[ Declaración de responsabilidad ética — firmas ]</strong></p>
<p><em>Omar Condori Pachauri · Merry Elizabeth Flores Llanque · Jaqueline Huahuaccapa Ccama (a completar)</em></p></td>
</tr>
</tbody>
</table>

## 5. Pruebas, monitoreo y continuidad

Pruebas: el plan completo está en S2-E2 §8. Cierre pendiente de las siete pruebas restantes con scripts/pruebas_conectividad.sh (acceso externo a la base, arranque en frío, login fallido, usuario sin permiso, redirección HTTP).

| **Tema**     | **Estado actual**                                                       | **Cierre**                             |
|--------------|-------------------------------------------------------------------------|----------------------------------------|
| Monitoreo    | Events, Logs y Metrics de Render; /actuator/health público (UP)         | Sonda cada 5 min y capturas            |
| Alertas      | Plan definido (salud, memoria, respaldo, vencimiento de la base)        | Contratar la sonda                     |
| Respaldo     | pg_dump con verificación sha256; 394 407 bytes en 0,59 s en laboratorio | Ejecutar sobre finanzas-db y programar |
| Restauración | 1,04 s, 4 tablas idénticas                                              | Medir RTO real en Render               |
| RTO / RPO    | Aplicación 15 min (≈ 6 obtenido); base 2 h / 24 h                       | Confirmar tras el ejercicio            |

Margen de memoria: la JVM suma 508 de 512 MB (margen de 4 MB), causa del incidente 502 ya resuelto; vigilar con Metrics.

## 6. Recomendaciones finales

| **Prioridad** | **Recomendación**                                                                               | **Plazo**  | **Responsable** |
|---------------|-------------------------------------------------------------------------------------------------|------------|-----------------|
| 1             | Cambiar la credencial del administrador, retirarla del informe y activar MFA en Render y GitHub | 20/11/2026 | RS              |
| 2             | Respaldo externo de finanzas-db, prueba de restauración y RTO medido                            | 10/12/2026 | RD              |
| 3             | Decidir plan de pago (≈ 13 USD/mes) o migración antes del vencimiento                           | 15/12/2026 | Comité          |
| 4             | Cargar permisos del administrador desde la base y deshabilitar Swagger UI y registro público    | 15/12/2026 | RI              |
| 5             | Sonda de disponibilidad y captura de Logs y Metrics                                             | 30/11/2026 | RI              |
| 6             | Retirar los datos de demostración en producción                                                 | 15/12/2026 | RD              |

## 7. Guion de demostración y sustentación

Recorrido técnico sugerido, de unos 12 minutos:

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<tbody>
<tr class="odd">
<td><p><strong>[ Enlace a video o recorrido técnico ]</strong></p>
<p><em>Pendiente de grabación</em></p></td>
</tr>
</tbody>
</table>

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

## 8. Conclusión de cierre de la línea

La solución se diseñó, implementó y documentó con criterio, y se declaran sin maquillaje los límites de una plataforma gratuita. Hay coherencia entre requerimientos, diseño, implementación, seguridad y operación. Para sostener la afirmación de «operada y validada» falta cerrar las evidencias pendientes y los dos riesgos críticos.

### Referencias

**\[1\]** Render, «Deploy for Free», Render Docs. https://render.com/docs/free (consultado el 17/11/2026).

**\[2\]** Render, «Pricing». https://render.com/pricing (consultado el 17/11/2026).

**\[3\]** ISO/IEC, «ISO/IEC 27001:2022» e «ISO/IEC 27002:2022», 2022.

**\[4\]** ACM, «Code of Ethics and Professional Conduct», 2018. https://www.acm.org/code-of-ethics

**\[5\]** Equipo del proyecto, Entregables 1 y 2 de Infraestructura (Semestre 1) y Entregables 1 a 3 (Semestre 2), 2026.
