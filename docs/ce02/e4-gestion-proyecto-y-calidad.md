# CE024 - S2 - E4: Gestión del Proyecto y Calidad de Software

**Equipo:** Omar Condori Pachauri (201810211) · Merry Elizabeth Flores Llanque (202211953) · Jaqueline Huahuaccapa Ccama (202210712) — Ciclo 10.

## Resumen ejecutivo

Este entregable documenta cómo se gestiona y cómo se asegura la calidad del ERP «FINANZAS» de Reumacenter E.I.R.L. Todas las cifras provienen de ejecuciones reales hechas sobre el repositorio (115 commits, 15/01/2026 – 17/11/2026) y se indica en cada caso si la evidencia es ejecutada, estática o propuesta.

**Calidad.** Se creó una suite automatizada de 37 pruebas de frontend (36 pasan, 1 omitida a propósito por un defecto confirmado) y otra de 30 pruebas de base de datos (30 pasan). Las pruebas encontraron un defecto real (UIT 2026 desactualizada en el cálculo de planilla) y dos fallas en los propios scripts de base de datos, ya corregidas. La cobertura global del frontend es baja (≈ 0,5 % de las líneas; 26,6 % en los archivos que ya tienen pruebas) y la del backend no pudo medirse: sus 3 pruebas Java no se ejecutaron en el entorno de preparación. Se declara como la principal brecha.

**Gestión.** Se formalizó el proyecto con acta de constitución, EDT, registro de riesgos, backlog de producto (12 historias de usuario más 8 ítems técnicos) y un sprint de cierre planificado. El historial muestra 27 semanas con actividad y un ritmo que bajó entre junio y noviembre. No existen actas de ceremonias ni tablero previo, por lo que no se inventan: la retrospectiva es un análisis basado en el historial de git.

**Operación.** Se diseñó un pipeline de CI/CD en GitHub Actions (pruebas, compilación, imagen, aprobación manual, despliegue con verificación de salud y reversión automática) y se probó lo que es ejecutable aquí: respaldo y restauración verificados con conteos idénticos, y prueba de carga con 90 641 transacciones de escritura sin fallos (pico de 1 983 tps en entorno de laboratorio). El pipeline y la observabilidad en producción están listos como código; la ejecución real en GitHub y en la máquina virtual quedan como evidencia pendiente.

**Aseguramiento y auditoría.** La matriz ISO/IEC 25010, el análisis estático (lizard y búsqueda de code smells), la auditoría técnica con listas de verificación y la autoevaluación de madurez (CMMI, nivel actual entre 1 y 2) producen 12 hallazgos, de los cuales 3 son de severidad alta y 1 crítica: el registro público de usuarios que permite elegir roles. Cada hallazgo tiene acción, fecha y criterio de cierre en el plan de mejora (sección 4.6).

| **Indicador**                                | **Resultado**                                        | **Fuente**  |
|----------------------------------------------|------------------------------------------------------|-------------|
| Pruebas de frontend (Karma + ChromeHeadless) | 36 de 37 pasan · 1 omitida (defecto H-07) · 0 fallos | Figura 1    |
| Pruebas de base de datos (SQL)               | 30 de 30 pasan                                       | Figura 3    |
| Cobertura de frontend (líneas)               | 27,11 % de lo instrumentado · ≈ 0,5 % del proyecto   | Sección 1.4 |
| Carga (escritura / lectura)                  | 1 400 – 1 983 tps / 45 – 92 tps, 0 fallos            | Figura 4    |
| Respaldo y restauración                      | Verificados (4 tablas, conteos idénticos)            | Figura 6    |
| Complejidad ciclomática media                | Backend 3,09 · Frontend 2,13                         | Sección 4.3 |
| Hallazgos de auditoría                       | 12 (1 crítico, 3 altos, 5 medios, 3 bajos)           | Sección 4.5 |

|                                                                                                                                                                                                                                                                                                                                                               |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Honestidad de la evidencia.** Las pruebas y mediciones se ejecutaron en un entorno aislado de preparación con una copia del frontend y una base PostgreSQL 16 con datos sintéticos. Los cambios de prueba no se aplicaron al repositorio del estudiante: se entregan en un paquete aparte. Lo no ejecutado aparece rotulado como «pendiente» o «propuesta». |

## 1. Pruebas

### 1.1 Estrategia

Se aplicó una pirámide de pruebas adaptada a un sistema que hasta ahora no tenía pruebas automatizadas de frontend ni de base de datos: pruebas unitarias en la capa donde hay lógica pura (validadores, guards, interceptor, motor de planilla), pruebas de reglas de negocio directamente en PostgreSQL (donde viven los triggers y procedimientos del Entregable 2) y pruebas de carga sobre la transacción de cobro.

| **Nivel**                      | **Herramienta**                         | **Alcance**                                                     | **Casos**  | **Estado**         |
|--------------------------------|-----------------------------------------|-----------------------------------------------------------------|------------|--------------------|
| Unitarias frontend             | Karma + Jasmine, ChromeHeadless         | Validadores, guards, interceptor, pipe, motor de planilla       | 37         | Ejecutadas         |
| Reglas de negocio en BD        | SQL (09_pruebas_bd.sql, con ROLLBACK)   | Integridad, caja, ventas/anulación, seguridad y RLS             | 30         | Ejecutadas         |
| Carga                          | pgbench 2 escenarios                    | Cobro completo (venta + detalle + caja + auditoría) y reporte   | 7 corridas | Ejecutadas         |
| Unitarias/integración backend  | JUnit 5 + Spring Boot Test              | 3 archivos existentes (contexto, Jasper, liquidación de seguro) | 3 clases   | No ejecutadas (\*) |
| Funcional de extremo a extremo | Manual guiado + capturas (Entregable 3) | 8 pantallas críticas con el sistema real                        | 8 flujos   | Evidencia en E3    |

*(\*) Las dependencias de Maven no se pudieron descargar en el entorno de preparación (error 403 del repositorio central), por lo que el backend no se compiló ni se probó. El job «backend» del pipeline las ejecutará en GitHub Actions.*

### 1.2 Casos de prueba automatizados

**Tabla 1. Pruebas de frontend por archivo (7 archivos, 37 casos).**

| **Archivo de prueba**           | **Casos** | **Qué verifica**                                                                                                   |
|---------------------------------|-----------|--------------------------------------------------------------------------------------------------------------------|
| planilla-engine.service.spec.ts | 11        | AFP/ONP, renta de quinta categoría, descuentos por tardanza y faltas, redondeo; 1 caso omitido por el defecto H-07 |
| auth.guard.spec.ts              | 6         | Redirección a /login sin sesión, paso con token válido, rutas por rol y por permiso                                |
| auth.interceptor.spec.ts        | 6         | Agrega el encabezado Bearer, no lo agrega a /api/auth/login, maneja 401 y 403                                      |
| unique-id.validator.spec.ts     | 5         | Validador asíncrono de identificador único: libre, ocupado, error de red, vacío                                    |
| punto-venta-check.guard.spec.ts | 5         | Bloquea la venta si el usuario no tiene punto de venta asignado                                                    |
| app.spec.ts                     | 2         | Creación del componente raíz y arranque sin zone.js (zoneless)                                                     |
| safe.pipe.spec.ts               | 2         | Sanitización de URL, HTML y recursos                                                                               |

**Tabla 2. Pruebas de base de datos por grupo (30 casos; el script completo va en el paquete de artefactos).**

| **Grupo**                | **Casos** | **Reglas verificadas**                                                                                                                                    |
|--------------------------|-----------|-----------------------------------------------------------------------------------------------------------------------------------------------------------|
| INT — Integridad         | 7         | Clave foránea, CHECK de total y cantidad, unicidad de comprobante y de rol, tipo de movimiento válido, no borrar sucursal con datos                       |
| CAJ — Flujo de caja      | 7         | Trigger de saldo (ingreso suma, egreso resta, egreso mayor al saldo rechazado), conciliación con saldo teórico, cierre con arqueo, caja cerrada inmutable |
| VEN — Ventas y anulación | 8         | Motivo obligatorio, propagación al detalle, anulada inmutable, no doble anulación, totales sin anuladas, auditoría automática                             |
| SEG — Seguridad          | 8         | Rol de lectura sin acceso a contraseñas ni sesiones, aplicación sin DDL ni borrado de auditoría, RLS por sucursal, auditor sin acceso a ventas            |

### 1.3 Reporte de ejecución

<img src="../../img_e4-gestion-proyecto-y-calidad/media/b08f8b8618a477ea48bf61b13e6e20cf4e400ab2.png" title="Figura 1. Ejecución real de la suite de frontend con cobertura (Chrome 153 headless): 36 pasan, 1 omitida, 0 fallan." style="width:4.89583in;height:1.42708in" alt="Figura 1. Ejecución real de la suite de frontend con cobertura (Chrome 153 headless): 36 pasan, 1 omitida, 0 fallan." />

*Figura 1. Ejecución real de la suite de frontend con cobertura (Chrome 153 headless): 36 pasan, 1 omitida, 0 fallan.*

**Hallazgo producido por las pruebas.** El caso «usa la UIT vigente 2026 (5 500)» falla: la función calcularQuintaCategoria(5 000) devuelve 267,33 cuando se espera ≈ 230, porque el código usa una constante de UIT de 5 150. Se registró como H-07 y el caso se dejó con xit (omitido) para que la suite sea verde hasta que se corrija la constante; al quitar xit, la suite vuelve a fallar y evidencia el defecto.

<img src="../../img_e4-gestion-proyecto-y-calidad/media/b662d5f7196b1470bd3c09af837e4db7e2dc7239.png" title="Figura 2. La misma suite con el caso de la UIT activo: 1 falla y 36 pasan (defecto H-07 reproducible)." style="width:4.89583in;height:1.14583in" alt="Figura 2. La misma suite con el caso de la UIT activo: 1 falla y 36 pasan (defecto H-07 reproducible)." />

*Figura 2. La misma suite con el caso de la UIT activo: 1 falla y 36 pasan (defecto H-07 reproducible).*

**Hallazgos en los propios scripts de base de datos.** La suite SQL detectó que (a) la restricción uq_roles_name duplicaba la que ya creaba JPA (roles_name_key) y (b) el script de roles no era repetible cuando existía una copia restaurada de la base. Ambos se corrigieron en los scripts del Entregable 2 y la suite quedó en 30 de 30.

<img src="../../img_e4-gestion-proyecto-y-calidad/media/a32f419c0b9e24b3ea4338836a15907b2eb231d6.png" title="Figura 3. Reporte de la suite SQL: 30 casos, 30 PASA (se ejecuta dentro de una transacción con ROLLBACK)." style="width:2.60417in;height:2.94792in" alt="Figura 3. Reporte de la suite SQL: 30 casos, 30 PASA (se ejecuta dentro de una transacción con ROLLBACK)." />

*Figura 3. Reporte de la suite SQL: 30 casos, 30 PASA (se ejecuta dentro de una transacción con ROLLBACK).*

#### Prueba de carga (pgbench, PostgreSQL 16, máquina de 2 núcleos y 8 GB)

**Tabla 3. Resultados. Escritura = venta + detalle + ingreso a caja con sus triggers y auditoría; lectura = reporte mensual de ventas por sucursal sobre 3 000 ventas.**

| **Escenario**     | **Clientes** | **Transacciones/s** | **Latencia media** | **Fallos** |
|-------------------|--------------|---------------------|--------------------|------------|
| Escritura         | 1            | 957                 | 1,05 ms            | 0          |
| Escritura         | 4            | 1 983               | 2,02 ms            | 0          |
| Escritura         | 8            | 1 702               | 4,70 ms            | 0          |
| Escritura         | 16           | 1 401               | 11,42 ms           | 0          |
| Lectura (reporte) | 1            | 45                  | 22,3 ms            | 0          |
| Lectura (reporte) | 8            | 90                  | 88,7 ms            | 0          |
| Lectura (reporte) | 16           | 92                  | 174,4 ms           | 0          |

Interpretación: el sistema real atiende decenas de cajeros como máximo, lejos de estas cargas; el cuello de botella observado es el reporte de ventas, que recorre toda la tabla del período (≈ 22 ms con 3 000 filas) y crecerá con los datos. El resultado es de un laboratorio con datos sintéticos y no equivale a una medición de producción.

<img src="../../img_e4-gestion-proyecto-y-calidad/media/76c056ca4c9160cd514f7049f611d858f7fb9a7d.png" title="Figura 4. Salida de pgbench (7 corridas de 15 s)." style="width:2.5in;height:3.36458in" alt="Figura 4. Salida de pgbench (7 corridas de 15 s)." />

*Figura 4. Salida de pgbench (7 corridas de 15 s).*

### 1.4 Cobertura

**Tabla 4. Cobertura de frontend medida con ng test --code-coverage (Istanbul).**

| **Métrica** | **Cubierto / total instrumentado** | **Porcentaje** |
|-------------|------------------------------------|----------------|
| Sentencias  | 146 / 548                          | 26,64 %        |
| Ramas       | 65 / 238                           | 27,31 %        |
| Funciones   | 26 / 146                           | 17,80 %        |
| Líneas      | 141 / 520                          | 27,11 %        |

Karma solo instrumenta los archivos que alguna prueba importa; por eso el porcentaje parece razonable. Medido contra el tamaño total del frontend (≈ 26 700 líneas de código), las 141 líneas cubiertas son ≈ 0,5 %. En el backend la cobertura no se midió. La meta propuesta es 60 % de líneas en los servicios críticos (ventas, caja, autorización de descuentos y planilla) y se vigila desde el pipeline (artefacto «cobertura-frontend»; JaCoCo para el backend).

### 1.5 Evidencia sobre flujos críticos

| **Flujo crítico**           | **Prueba automatizada**         | **Evidencia manual**                | **Brecha**                       |
|-----------------------------|---------------------------------|-------------------------------------|----------------------------------|
| Inicio de sesión y token    | Interceptor (6), guard (6)      | Diagrama de secuencia de login (E3) | Sin prueba del backend (JWT)     |
| Cobro de una venta          | BD: CAJ-02/03/04, VEN-08, carga | Venta (E3, fig. 5)                  | crearVenta sin prueba unitaria   |
| Anulación / nota de crédito | BD: VEN-01…07                   | Bitácora de auditoría (E3, fig. 12) | Flujo de autorización sin prueba |
| Caja chica y cierre         | BD: CAJ-01…07                   | Caja general (E3, fig. 10)          | —                                |
| Permisos y sucursal         | Guards (11), BD: SEG-01…08      | Usuarios y roles (E3, fig. 6)       | @PreAuthorize sin prueba         |
| Planilla y honorarios       | Motor de planilla (11)          | Honorarios (E3, fig. 11)            | Defecto H-07                     |
| Comprobante único           | BD: INT-03                      | —                                   | Concurrencia sin prueba          |

## 2. Integración, despliegue y operación continua

### 2.1 Pipeline de CI/CD

El repositorio revisado no contiene un workflow de integración continua: hasta ahora la compilación y las pruebas dependían de ejecutarlas a mano. Se diseñó un pipeline en GitHub Actions (archivo .github/workflows/ci.yml, en el paquete de artefactos) con cinco jobs. Los tres primeros corren en paralelo en cada push y pull request; los dos últimos solo en main.

<img src="../../img_e4-gestion-proyecto-y-calidad/media/c081006bc05028ae2ed2a64cb173d186378deffc.png" title="Figura 5. Flujo del pipeline: pruebas en paralelo, imagen versionada por commit, aprobación manual, despliegue con verificación de salud y reversión automática." style="width:6.25in;height:0.9375in" alt="Figura 5. Flujo del pipeline: pruebas en paralelo, imagen versionada por commit, aprobación manual, despliegue con verificación de salud y reversión automática." />

*Figura 5. Flujo del pipeline: pruebas en paralelo, imagen versionada por commit, aprobación manual, despliegue con verificación de salud y reversión automática.*

| **Job**       | **Qué hace**                                                                                                          | **Criterio de paso**                       |
|---------------|-----------------------------------------------------------------------------------------------------------------------|--------------------------------------------|
| frontend      | npm ci, ng test con cobertura en ChromeHeadless sin sandbox, ng build de producción; sube la cobertura como artefacto | Todas las pruebas pasan y el build compila |
| backend       | mvn verify contra un PostgreSQL 16 de servicio; sube reportes surefire                                                | Pruebas Java y empaquetado correctos       |
| base-de-datos | Crea la base con los scripts 00–08 y ejecuta la suite 09 (30 casos)                                                   | Ningún caso con resultado FALLA            |
| imagen        | Construye la imagen del backend y la publica en GHCR con el SHA del commit como etiqueta (y latest)                   | Los tres jobs anteriores en verde          |
| desplegar     | Entorno «produccion» con aprobación manual; por SSH ejecuta ops/desplegar.sh \<sha\>                                  | Salud UP en 150 s; si no, reversión        |

### 2.2 Configuración

Fragmento representativo del job de frontend. El lanzador personalizado ChromeHeadlessCI (con --no-sandbox) está definido en karma.conf.js; es necesario porque los runners y los contenedores ejecutan Chrome como root. El proyecto es zoneless (provideZonelessChangeDetection), de modo que las pruebas lo declaran en TestBed y evitan fakeAsync.

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<tbody>
<tr class="odd">
<td><p>frontend:</p>
<p>runs-on: ubuntu-latest</p>
<p>steps:</p>
<p>- uses: actions/checkout@v4</p>
<p>- uses: actions/setup-node@v4</p>
<p>with: { node-version: 22, cache: npm, cache-dependency-path: finanzas-frontend/package-lock.json }</p>
<p>- run: npm ci</p>
<p>- run: npx ng test --watch=false --browsers=ChromeHeadlessCI --code-coverage</p>
<p>- run: npx ng build --configuration production</p></td>
</tr>
</tbody>
</table>

Los secretos (VM_HOST, VM_USER, VM_SSH_KEY) se guardan como secretos de GitHub y nunca en el repositorio; el despliegue exige aprobación porque el entorno «produccion» tiene revisores requeridos. Para que la reversión funcione, el docker-compose.yml debe referenciar la imagen con una variable (image: …/finanzas-backend:\${BACKEND_TAG:-latest}); es el único cambio previo necesario en deploy/docker-compose.yml.

### 2.3 Evidencias disponibles

| **Evidencia**                                              | **Resultado**                                                                                                                                                              | **Estado**                    |
|------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------|
| Compilación de producción del frontend (ng build)          | Correcta en 20 s; bundle inicial 811,89 kB (176,46 kB transferidos) frente a un presupuesto de 500 kB (advertencia H-11); 3 hojas de estilo exceden el presupuesto de 4 kB | Ejecutada                     |
| Pruebas de frontend y de base de datos                     | Secciones 1.3 y 1.4                                                                                                                                                        | Ejecutada                     |
| Respaldo y restauración                                    | Figura 6                                                                                                                                                                   | Ejecutada                     |
| Ejecución del workflow en GitHub (pestaña Actions)         | Captura de los 5 jobs en verde                                                                                                                                             | Pendiente                     |
| Despliegue en la VM con aprobación y verificación de salud | Registro del job «desplegar»                                                                                                                                               | Pendiente (VM en preparación) |

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<tbody>
<tr class="odd">
<td><p><strong>[ Captura: pestaña Actions con los 5 jobs del pipeline en verde ]</strong></p>
<p><em>Se toma tras el primer push de ci.yml a main</em></p></td>
</tr>
</tbody>
</table>

### 2.4 Observabilidad: registros, métricas y alertas

El pom del backend ya incluye Spring Boot Actuator, pero la exposición de sus endpoints no se verificó ni se protegió en producción. Se propone el siguiente esquema mínimo, proporcional al tamaño de la empresa (una VM y un usuario operador):

| **Pilar**        | **Estado actual (revisado)**                                                         | **Propuesta**                                                                                                                                       | **Herramienta**                       |
|------------------|--------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------|
| Registros (logs) | 22 usos de System.out/printStackTrace; 81 catch(Exception); sin formato estructurado | Logs JSON en consola (logging.structured.format.console=ecs), nivel INFO, correlación por petición; docker compose logs con rotación (max-size 10m) | Spring Boot + Docker                  |
| Métricas         | Actuator en el pom                                                                   | Exponer health, info, metrics y prometheus (solo desde la red interna o con autenticación); histogramas de latencia HTTP                            | Micrometer + Prometheus/Grafana Cloud |
| Disponibilidad   | Ninguna                                                                              | Sonda externa cada 5 min a /actuator/health y a la URL del frontend; ops/salud.sh como respaldo                                                     | UptimeRobot (plan gratuito)           |
| Alertas          | Ninguna                                                                              | 6 reglas (tabla 5) con aviso por correo o Telegram                                                                                                  | Alertmanager o alertas de UptimeRobot |

**Tabla 5. Reglas de alerta propuestas (archivo observabilidad/alertas.yml).**

| **Alerta**            | **Condición**                                                     | **Severidad** |
|-----------------------|-------------------------------------------------------------------|---------------|
| ApiCaida              | La API no responde durante 2 min                                  | Crítica       |
| ErroresServidor5xx    | Más del 2 % de peticiones con 5xx durante 5 min                   | Advertencia   |
| LatenciaP95Alta       | p95 de latencia mayor a 1,5 s durante 10 min                      | Advertencia   |
| MemoriaJvmAlta        | Heap usado mayor al 85 % durante 10 min (la JVM corre con -Xmx2g) | Advertencia   |
| PoolConexionesAgotado | Más del 90 % de conexiones Hikari activas durante 5 min           | Advertencia   |
| RespaldoAntiguo       | Sin respaldo en las últimas 26 horas                              | Crítica       |

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<tbody>
<tr class="odd">
<td><p><strong>[ Captura: panel de métricas o monitor de disponibilidad en producción ]</strong></p>
<p><em>Pendiente: requiere la VM desplegada; se adjunta en el Anexo C</em></p></td>
</tr>
</tbody>
</table>

### 2.5 Reversión y respaldos

Objetivos acordados para una operación de una sola empresa: pérdida máxima de datos (RPO) de 24 horas y tiempo de recuperación (RTO) de 1 hora. El respaldo diario cumple el RPO; el RTO se valida con la prueba de restauración.

| **Mecanismo**          | **Implementación**                                                                                             | **Verificado**                |
|------------------------|----------------------------------------------------------------------------------------------------------------|-------------------------------|
| Respaldo lógico diario | ops/respaldo.sh: pg_dump -Fc, comprueba con pg_restore --list, genera sha256, retención de 14 días; cron 02:00 | Sí (Figura 6)                 |
| Prueba de restauración | ops/restaurar_prueba.sh restaura en una base temporal y compara conteos con la original                        | Sí: 4 tablas idénticas        |
| Versionado de imágenes | Cada despliegue usa la etiqueta del SHA; ops/desplegar.sh guarda la versión previa en .version_actual          | Revisión de script (sintaxis) |
| Reversión automática   | Si /actuator/health no responde UP en 150 s, vuelve a la versión previa                                        | Pendiente (necesita la VM)    |
| Reversión manual       | ops/rollback.sh \<sha-anterior\>; si hubo cambio de esquema, pg_restore del respaldo previo al despliegue      | Pendiente (necesita la VM)    |

<img src="../../img_e4-gestion-proyecto-y-calidad/media/0264da2aee7a4b7bbec51e2adecce172673a8d03.png" title="Figura 6. Respaldo con verificación de integridad y restauración comparada (laboratorio, base de demostración)." style="width:4.89583in;height:1.38542in" alt="Figura 6. Respaldo con verificación de integridad y restauración comparada (laboratorio, base de demostración)." />

*Figura 6. Respaldo con verificación de integridad y restauración comparada (laboratorio, base de demostración).*

Limitación: el respaldo diario queda en la misma VM; para cumplir la regla 3-2-1 debe copiarse a un almacenamiento externo (por ejemplo, el bucket gratuito de Oracle) y se registra como acción M-04 en el plan de mejora. El esquema se versiona hoy con scripts manuales porque Flyway está desactivado (H-06).

## 3. Gestión ágil del proyecto

### 3.1 Acta de constitución (Project Charter)

| **Elemento**           | **Contenido**                                                                                                                                                                                                        |
|------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Proyecto               | ERP administrativo-financiero «FINANZAS» para Reumacenter E.I.R.L. (laboratorio clínico, Juliaca)                                                                                                                    |
| Patrocinador / cliente | Gerencia de Reumacenter E.I.R.L. (usuaria del sistema y validadora de requisitos)                                                                                                                                    |
| Problema               | Ventas, cajas, compras, inventario y honorarios se gestionaban en procesos separados, sin trazabilidad ni control de descuentos y anulaciones                                                                        |
| Objetivo general       | Integrar los procesos administrativos y financieros en una aplicación web con control de acceso, auditoría y reportes                                                                                                |
| Objetivos específicos  | \(1\) Requerimientos y diseño aprobados (E1). (2) Plataforma de datos segura de 76 tablas (E2). (3) Sistema funcional integrado y desplegable (E3). (4) Calidad medida, operación continua y mejora planificada (E4) |
| Alcance incluido       | Ventas y cobros, descuentos y autorizaciones, anulación y notas de crédito, caja chica y general, compras, kardex, honorarios y planilla, usuarios/roles/permisos, auditoría, reportes y panel                       |
| Fuera de alcance       | Aplicación móvil, historia clínica y resultados de laboratorio (el ERP es administrativo-financiero)                                                                                                                 |
| Supuestos              | Datos de prueba representativos; soporte multi-sucursal desde el diseño; infraestructura gratuita (Vercel y Oracle Always Free)                                                                                      |
| Restricciones          | Una persona como desarrollador principal; presupuesto de infraestructura cero; calendario académico (semestre 1)                                                                                                     |
| Criterios de éxito     | 12 historias de usuario operativas; pruebas automatizadas en verde en CI; cero hallazgos críticos abiertos; respaldo restaurable; sistema accesible por URL pública                                                  |
| Hitos                  | E1 requerimientos · E2 datos · E3 sistema integrado (17/11/2026) · E4 calidad y operación · Sustentación                                                                                                             |

#### Estructura de desglose del trabajo (EDT)

<img src="../../img_e4-gestion-proyecto-y-calidad/media/72f3b5df4019007b4428e62a319dff44d24d816b.png" title="Figura 7. EDT del proyecto en seis paquetes de trabajo." style="width:5.41667in;height:1.35417in" alt="Figura 7. EDT del proyecto en seis paquetes de trabajo." />

*Figura 7. EDT del proyecto en seis paquetes de trabajo.*

#### Equipo y roles

| **Persona**                        | **Rol en el marco ágil**                                        | **Evidencia**                             |
|------------------------------------|-----------------------------------------------------------------|-------------------------------------------|
| Omar Condori Pachauri (estudiante) | Product Owner delegado, Scrum Master y desarrollador full stack | 102 + 3 commits (≈ 91 % del historial)    |
| Colaborador «Samuel-910»           | Desarrollador                                                   | 10 commits (≈ 9 %)                        |
| Gerencia y usuarios de Reumacenter | Stakeholders / dueños del producto                              | Fuente de los requisitos del Entregable 1 |
| Docente asesor                     | Revisor de entregables                                          | A completar por el estudiante             |

*Un equipo de uno a dos desarrolladores concentra roles; el riesgo que esto genera (dependencia de una sola persona) está en el registro de riesgos.*

#### Registro de riesgos

| **ID** | **Riesgo**                                              | **Prob.** | **Impacto** | **Respuesta / mitigación**                                                   |
|--------|---------------------------------------------------------|-----------|-------------|------------------------------------------------------------------------------|
| R-01   | Dependencia de una sola persona (factor bus = 1)        | Alta      | Alto        | Documentar (E1–E4), automatizar despliegue, segundo desarrollador con acceso |
| R-02   | Acceso indebido por registro público de usuarios (H-01) | Alta      | Crítico     | Cerrar /api/auth/register o exigir ADMIN (sprint 1)                          |
| R-03   | Regresiones por cobertura casi nula                     | Alta      | Alto        | CI obligatorio, pruebas de servicios críticos, meta de 60 %                  |
| R-04   | Pérdida de datos por falla de la VM                     | Media     | Alto        | Respaldo diario fuera de la VM (M-04) y prueba de restauración               |
| R-05   | Cálculos tributarios obsoletos (UIT, tasas)             | Media     | Medio       | Parámetros en tabla, pruebas con valores oficiales                           |
| R-06   | Retraso del despliegue (cuenta Oracle, dominio)         | Media     | Medio       | Plan alterno: Docker Compose local o proveedor equivalente                   |
| R-07   | Rendimiento de reportes con el crecimiento de datos     | Media     | Medio       | Índices y vistas; pruebas de carga recurrentes                               |
| R-08   | Secretos versionados en el repositorio (H-03)           | Alta      | Alto        | Rotar llaves, usar secretos de entorno, limpiar historial                    |

### 3.2 Historias de usuario y backlogs

El Product Backlog parte de las 12 historias del Entregable 1 (ya implementadas) y añade 8 ítems técnicos que nacen de la auditoría de este entregable. Las estimaciones en puntos de historia (escala Fibonacci) son propuestas del estudiante a validar con el equipo; no provienen de un póquer de planificación previo.

**Tabla 6. Product Backlog ordenado por prioridad (MoSCoW).**

| **ID**   | **Historia (resumen)**                                                                                  | **Prio.** | **Pts** | **Estado** | **Verificación**         |
|----------|---------------------------------------------------------------------------------------------------------|-----------|---------|------------|--------------------------|
| TQ-01    | Como administrador, quiero que solo yo pueda crear usuarios para evitar auto-asignación de roles (H-01) | Must      | 3       | Sprint 1   | Prueba de seguridad      |
| TQ-02    | Como operador, quiero llaves JWT fuera del repositorio y rotadas (H-03)                                 | Must      | 2       | Sprint 1   | Revisión + secreto en CI |
| TQ-03    | Como contador, quiero la UIT 2026 correcta en el cálculo de renta (H-07)                                | Must      | 1       | Sprint 1   | Caso xit → it            |
| TQ-04    | Como equipo, quiero CI en cada push (sección 2.1)                                                       | Must      | 2       | Sprint 1   | Run verde en Actions     |
| HU-01    | Cajero: registrar venta, cobrar y emitir comprobante                                                    | Must      | 13      | Hecha      | BD + carga + manual      |
| HU-04    | Cajero: anular comprobante o emitir nota de crédito                                                     | Must      | 8       | Hecha      | BD (VEN-01…07)           |
| HU-05    | Cajero: abrir, arquear y cerrar caja chica                                                              | Must      | 8       | Hecha      | BD (CAJ-01…07)           |
| HU-10    | Administrador: crear usuarios, roles y permisos                                                         | Must      | 8       | Hecha      | Guards + BD (SEG)        |
| TQ-05    | Como equipo, quiero ejecutar las pruebas Java y medir cobertura (H-04)                                  | Must      | 5       | Sprint 2   | Informe JaCoCo           |
| TQ-06    | Como equipo, quiero pruebas y refactorización de crearVenta (H-05)                                      | Should    | 8       | Sprint 2   | CCN ≤ 15 por método      |
| HU-02/03 | Cajero/aprobador: solicitar y aprobar descuento especial                                                | Should    | 8       | Hecha      | Manual                   |
| TQ-07    | Como DBA, quiero migraciones versionadas con Flyway (H-06)                                              | Should    | 5       | Sprint 2   | Migración V1 aplicada    |
| HU-06/07 | Logística: orden de compra, recepción y kardex                                                          | Should    | 8       | Hecha      | Manual                   |
| HU-08    | Finanzas: liquidar honorarios por período                                                               | Should    | 8       | Hecha      | Planilla (11 pruebas)    |
| HU-09    | Finanzas: cuentas por cobrar y por pagar                                                                | Should    | 5       | Hecha      | Manual                   |
| HU-11    | Administrador: consultar bitácora de auditoría                                                          | Should    | 3       | Hecha      | BD (VEN-04, VEN-08)      |
| HU-12    | Finanzas: reportes y exportación a Excel                                                                | Could     | 5       | Hecha      | Manual                   |
| TQ-08    | Como operador, quiero métricas y alertas (sección 2.4)                                                  | Should    | 3       | Backlog    | Alerta de prueba         |

**Criterios de aceptación de ejemplo (formato Dado-Cuando-Entonces).** TQ-01 — Dado un visitante anónimo, cuando envía POST /api/auth/register, entonces recibe 401 y no se crea ningún usuario. HU-04 — Dado una venta ya anulada, cuando alguien intenta modificar su total, entonces la base de datos rechaza el cambio (VEN-05).

**Definición de terminado (DoD).** Código revisado por otra persona o por la lista de la sección 4.2; pruebas automatizadas nuevas o actualizadas; CI en verde; sin hallazgos críticos nuevos; documentación y script de datos actualizados cuando cambian tablas.

**Tabla 7. Sprint Backlog del Sprint 1 «Cerrar lo crítico y activar CI» (planificado: 18/11 al 01/12/2026, 8 puntos).**

| **Ítem** | **Tarea**                                                                                                   | **Horas** | **Responsable**    |
|----------|-------------------------------------------------------------------------------------------------------------|-----------|--------------------|
| TQ-01    | Restringir /api/auth/register a ADMIN; ignorar roles enviados por el cliente; prueba de integración         | 4         | Omar               |
| TQ-01    | Retirar /api/test-notifications/\*\* del permitAll (H-02)                                                   | 1         | Omar               |
| TQ-02    | Generar llaves nuevas con generar-llaves.sh, agregar \*.pem a .gitignore, leer ruta por variable de entorno | 3         | Omar               |
| TQ-03    | Actualizar UIT a 5 500, cambiar xit por it y verificar el caso                                              | 1         | Omar               |
| TQ-04    | Copiar ci.yml, configurar secretos y entorno «produccion», primer run                                       | 3         | Omar               |
| —        | Revisión del sprint y retrospectiva (sección 3.4)                                                           | 2         | Omar y colaborador |

Capacidad supuesta: 15 horas efectivas por semana (calendario académico). El Sprint 2 (02/12 al 15/12) toma TQ-05, TQ-06 y TQ-07 (18 puntos), con TQ-08 como reserva.

### 3.3 Ceremonias y métricas

| **Ceremonia**            | **Frecuencia / duración**                | **Adaptación a un equipo de 1–2 personas**                                          | **Evidencia**    |
|--------------------------|------------------------------------------|-------------------------------------------------------------------------------------|------------------|
| Planificación del sprint | Inicio de cada sprint de 2 semanas · 1 h | Se elige la meta y se toman ítems del backlog según capacidad                       | Tabla 7          |
| Scrum diario             | Diario · 5 min                           | Asíncrono: mensaje de texto o descripción del commit (qué hice, qué haré, bloqueos) | Historial git    |
| Revisión del sprint      | Fin del sprint · 45 min                  | Demostración a la gerencia o al docente con el sistema desplegado                   | Pendiente (acta) |
| Retrospectiva            | Fin del sprint · 30 min                  | Qué empezar, detener y continuar (sección 3.4)                                      | Sección 3.4      |
| Refinamiento del backlog | Semanal · 30 min                         | Revisar prioridad con los hallazgos nuevos                                          | Tabla 6          |

Métricas reales del historial (no hay sprints previos, así que los commits sirven como indicador aproximado de ritmo):

<img src="../../img_e4-gestion-proyecto-y-calidad/media/faa56ef8b1ce65847fd959a90a186556d25f61fb.png" title="Figura 8. Commits por mes. Septiembre y octubre no tienen actividad registrada; noviembre incluye la reestructuración del repositorio." style="width:4.79167in;height:1.78125in" alt="Figura 8. Commits por mes. Septiembre y octubre no tienen actividad registrada; noviembre incluye la reestructuración del repositorio." />

*Figura 8. Commits por mes. Septiembre y octubre no tienen actividad registrada; noviembre incluye la reestructuración del repositorio.*

| **Métrica**                                       | **Valor medido** | **Métrica a seguir desde el Sprint 1**    | **Meta**       |
|---------------------------------------------------|------------------|-------------------------------------------|----------------|
| Semanas con commits                               | 27 de 45 (60 %)  | Velocidad (puntos terminados por sprint)  | Estable ± 20 % |
| Commits con mensaje convencional (feat/fix/chore) | 36 de 115 (31 %) | Tiempo de ciclo (de «en curso» a «hecho») | ≤ 5 días       |
| Autores                                           | 2                | Pruebas fallidas en CI por semana         | Tendencia a 0  |
| Pruebas automatizadas                             | 67 (37 + 30)     | Cobertura de servicios críticos           | ≥ 60 %         |

### 3.4 Retrospectiva

No existen actas de retrospectivas anteriores. Esta es una retrospectiva de cierre del ciclo de construcción, elaborada por el estudiante a partir de la evidencia del historial y de las mediciones de este entregable; debe discutirse con el colaborador en la revisión del Sprint 1.

| **Continuar**                                         | **Empezar**                                                | **Detener**                                             |
|-------------------------------------------------------|------------------------------------------------------------|---------------------------------------------------------|
| Módulos por dominio con capas separadas (facilitó E3) | Ejecutar CI en cada cambio y no integrar con pruebas rojas | Commits con mensajes como «asd» o «a» (sin información) |
| Auditoría de escrituras y control de descuentos       | Escribir pruebas junto con cada servicio crítico           | Dejar secretos y endpoints de prueba abiertos           |
| Documentar cada entregable con evidencia ejecutada    | Planificar en sprints cortos con backlog visible           | Acumular lógica en una sola clase (VentaServiceImpl)    |

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<tbody>
<tr class="odd">
<td><p><strong>[ Evidencia: tablero del backlog (Trello, GitHub Projects o similar) y acta de revisión del Sprint 1 ]</strong></p>
<p><em>Pendiente: se crea el tablero con las tablas 6 y 7 y se adjunta la captura en el Anexo C</em></p></td>
</tr>
</tbody>
</table>

### 3.5 Comparación y adaptación de marcos: RUP frente a Scrum

| **Dimensión**           | **RUP (Proceso Unificado)**                                                                                                                               | **Scrum**                                                                              | **Decisión para FINANZAS**                                                                 |
|-------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------|
| Ciclo de vida           | Iterativo e incremental dirigido por casos de uso y por riesgos                                                                                           | Iterativo con sprints de duración fija                                                 | Sprints de 2 semanas con riesgos revisados cada sprint                                     |
| Fases                   | Inicio, Elaboración, Construcción, Transición                                                                                                             | Sin fases; el producto se entrega por incrementos                                      | Se conservan las fases como hitos de entregables (tabla 8)                                 |
| Disciplinas / prácticas | Modelado de negocio, requisitos, análisis y diseño, implementación, pruebas, despliegue, configuración y gestión de cambios, gestión de proyecto, entorno | Backlog, planificación, scrum diario, revisión, retrospectiva, definición de terminado | Prácticas de Scrum para ejecutar; disciplinas de RUP como lista de artefactos obligatorios |
| Roles                   | Analista, arquitecto, desarrollador, probador, gestor de proyecto…                                                                                        | Product Owner, Scrum Master, equipo de desarrollo                                      | Roles de Scrum, con el arquitecto y el probador como sombreros del equipo                  |
| Documentación           | Abundante y formal                                                                                                                                        | Mínima, software funcionando                                                           | La necesaria para la evaluación académica y la operación (E1–E4)                           |
| Cambios                 | Controlados por gestión de configuración                                                                                                                  | Bienvenidos entre sprints                                                              | Cambios entre sprints, registrados en el backlog                                           |

**Tabla 8. Adaptación híbrida: fases de RUP, entregables y prácticas de Scrum aplicadas.**

| **Fase RUP** | **Entregable**             | **Disciplinas cubiertas**                   | **Práctica de Scrum**                       |
|--------------|----------------------------|---------------------------------------------|---------------------------------------------|
| Inicio       | E1 Requerimientos y diseño | Modelado de negocio, requisitos             | Product Backlog inicial (12 HU)             |
| Elaboración  | E1–E2 Arquitectura y datos | Análisis y diseño, entorno                  | Spikes técnicos; decisiones de arquitectura |
| Construcción | E3 Sistema integrado       | Implementación, despliegue                  | Incrementos por módulo                      |
| Transición   | E4 Calidad y operación     | Pruebas, configuración, gestión de proyecto | Sprint 1–2 de cierre, retrospectiva         |

Conclusión: Scrum se adopta para el trabajo diario porque un equipo pequeño y requisitos que cambian de forma frecuente (descuentos, anulaciones, planilla) se benefician de iteraciones cortas. De RUP se conservan la gestión de riesgos temprana y la lista de artefactos, porque el sistema maneja dinero y exige trazabilidad.

## 4. Aseguramiento de calidad y auditoría

### 4.1 Matriz de calidad (ISO/IEC 25010)

Cada característica se mide con una métrica concreta; el valor es el medido en este entregable o se marca «sin medir». El semáforo resume el estado frente a la meta: verde cumple, ámbar parcial, rojo no cumple.

| **Característica**            | **Métrica**                                                                     | **Valor medido**                            | **Meta**                    | **Riesgo asociado**        | **Estado** |
|-------------------------------|---------------------------------------------------------------------------------|---------------------------------------------|-----------------------------|----------------------------|------------|
| Adecuación funcional          | Historias implementadas; flujos críticos con prueba automática de punta a punta | 12 de 12; 0 de 7 (hay pruebas parciales)    | 100 %; 7 de 7               | R-03 regresiones           | Ámbar      |
| Eficiencia de desempeño       | tps de cobro; latencia del reporte; bundle inicial                              | 1 983 tps; 22 ms (3 000 filas); 812 kB      | ≥ 100 tps; \< 1 s; ≤ 500 kB | R-07 reportes lentos       | Ámbar      |
| Compatibilidad / portabilidad | Despliegue reproducible con un comando                                          | Docker Compose (3 servicios)                | Un comando                  | R-06 retraso de despliegue | Verde      |
| Usabilidad                    | Encuesta SUS con usuarios de caja                                               | Sin medir                                   | ≥ 68                        | —                          | Gris       |
| Fiabilidad                    | Transacciones sin fallo; respaldo restaurable; reversión                        | 90 641 de 90 641; sí; pendiente             | 100 %; sí; sí               | R-04 pérdida de datos      | Ámbar      |
| Seguridad                     | Hallazgos críticos/altos abiertos; pruebas de seguridad en BD                   | 1 crítico y 3 altos; 8 de 8                 | 0 críticos y 0 altos        | R-02, R-08                 | Rojo       |
| Mantenibilidad                | CCN media; funciones con CCN \> 15; cobertura                                   | 3,09 / 2,13; 30 BE y 22 FE; ≈ 0,5 %         | \< 5; \< 10; ≥ 60 %         | R-01, R-03                 | Rojo       |
| Capacidad de prueba           | Pruebas automáticas ejecutadas en CI                                            | 67 pruebas, ejecutadas a mano; CI propuesta | En cada push                | R-03                       | Ámbar      |

### 4.2 Revisiones, inspecciones y listas de verificación

El historial de git no muestra evidencia de revisión por pares (solo una fusión con el remoto). Se adopta la siguiente lista para cada pull request y una inspección trimestral de seguridad.

| **Lista de verificación de pull request**                                          | **Sí** | **No** |
|------------------------------------------------------------------------------------|--------|--------|
| ¿Compila y las pruebas del pipeline están en verde?                                | ☐      | ☐      |
| ¿La lógica nueva tiene una prueba automatizada que falla sin el cambio?            | ☐      | ☐      |
| ¿Los endpoints nuevos tienen @PreAuthorize y validan la entrada (Bean Validation)? | ☐      | ☐      |
| ¿Se evita catch(Exception) genérico, System.out y console.log?                     | ☐      | ☐      |
| ¿Se transmiten solo DTO (no entidades) y no se exponen contraseñas ni tokens?      | ☐      | ☐      |
| ¿Los cambios de tablas llevan script de migración y actualización del diccionario? | ☐      | ☐      |
| ¿El mensaje del commit usa feat/fix/chore y explica el porqué?                     | ☐      | ☐      |
| ¿Ninguna función nueva supera CCN 15 ni 50 líneas?                                 | ☐      | ☐      |

**Inspección realizada en este entregable.** Se aplicó la lista de seguridad y mantenibilidad sobre todo el código con herramientas automáticas (lizard y búsquedas por patrón, sección 4.3). Revisión humana por pares: pendiente de asignar al colaborador.

### 4.3 Calidad del código, code smells y deuda técnica

**Tabla 9. Complejidad medida con lizard (CCN = complejidad ciclomática; NLOC = líneas sin comentarios ni blancos).**

| **Métrica**                             | **Backend (Java)**                                                   | **Frontend (TypeScript)**                                                                 |
|-----------------------------------------|----------------------------------------------------------------------|-------------------------------------------------------------------------------------------|
| Funciones analizadas / NLOC             | 1 247 / 17 325                                                       | 4 180 / 26 732                                                                            |
| CCN media / mediana                     | 3,09 / 1                                                             | 2,13 / 1                                                                                  |
| Funciones con CCN \> 10 / \> 15 / \> 20 | 68 / 30 / 14                                                         | 63 / 22 / 15                                                                              |
| Funciones de más de 50 líneas           | 42 (6 superan 100)                                                   | 10                                                                                        |
| Funciones con 5 o más parámetros        | 16                                                                   | 17                                                                                        |
| Archivos más grandes (NLOC)             | UserServiceImpl 871 · VentaServiceImpl 753 · CatalogoServiceImpl 491 | venta-servicios.component 1 141 · cobrar-servicio.component 1 029 · gestion-descuento 697 |

La media es baja y saludable porque la mayor parte son captadores y métodos cortos; el problema está en la cola: crearVenta (VentaServiceImpl) tiene CCN 99 y unas 457 líneas, 99 caminos de ejecución distintos para una sola operación crítica, con la cobertura de pruebas más baja posible. Le siguen ProductoServiceImpl.save (CCN 44), ReportService.generarTicketVenta (40) y los métodos de reimpresión (39 y 36).

**Tabla 10. Code smells detectados por búsqueda de patrones (conteos exactos).**

| **Smell**                           | **Conteo** | **Riesgo**                               | **Tratamiento**                                  |
|-------------------------------------|------------|------------------------------------------|--------------------------------------------------|
| catch(Exception) genérico (BE)      | 81         | Oculta errores y dificulta la causa raíz | Excepciones específicas + GlobalExceptionHandler |
| System.out / printStackTrace (BE)   | 22         | Sin nivel ni destino de log              | Reemplazar por SLF4J                             |
| catch vacío (BE)                    | 2          | Errores silenciados                      | Registrar o propagar                             |
| console.log (FE)                    | 101        | Ruido y posible fuga de datos            | Eliminar o usar un logger con nivel              |
| alert() nativo (FE)                 | 13         | UX inconsistente con SweetAlert2         | Sustituir                                        |
| Tipo any (FE)                       | 910        | Pierde la verificación de tipos          | Tipar DTO de los servicios críticos primero      |
| subscribe() manual (FE)             | 380        | Fugas si no se cancelan                  | async pipe, takeUntilDestroyed                   |
| OnPush solo en 4 de 143 componentes | 4 / 143    | Detección de cambios más costosa         | Aplicar a listas y tablas                        |
| TODO/FIXME (FE) · TODO (BE)         | 14 · 0     | Trabajo pendiente sin dueño              | Convertir a ítems de backlog                     |

**Tabla 11. Deuda técnica estimada (estimación del estudiante, en horas efectivas; 15 h por semana).**

| **Ítem de deuda**                                           | **Hallazgo** | **Horas**           | **Prioridad** |
|-------------------------------------------------------------|--------------|---------------------|---------------|
| Pruebas de servicios críticos del backend y JaCoCo          | H-04         | 40                  | Alta          |
| Refactorizar crearVenta con pruebas de caracterización      | H-05         | 24                  | Alta          |
| Tipar DTO de servicios críticos (no los 910 any)            | H-08         | 16                  | Media         |
| Activar Flyway con migración base                           | H-06         | 12                  | Media         |
| OnPush en los 20 componentes de listas más usados           | H-09         | 12                  | Baja          |
| Cerrar registro público, endpoints de prueba y rotar llaves | H-01–H-03    | 8                   | Crítica       |
| Reducir catch(Exception), System.out y console.log          | H-08         | 16                  | Media         |
| Reducir el bundle inicial a ≤ 500 kB                        | H-11         | 8                   | Baja          |
| Total                                                       |              | 136 h (≈ 9 semanas) |               |

**Plan de refactorización de crearVenta.** Primero pruebas de caracterización que fijen el comportamiento actual (misma venta de entrada, mismos totales y asientos de salida); después extracción en métodos de responsabilidad única: validarSolicitud, resolverPrecioYDescuento, calcularTotales, persistirCabecera, persistirDetalles, registrarIngresoEnCaja y registrarAuditoría, coordinados por un método de orquestación de menos de 40 líneas, todos con CCN ≤ 15. La transacción se conserva en el método público para no alterar la atomicidad.

### 4.4 Informe de auditoría técnica

Alcance: proceso de desarrollo y producto FINANZAS al 17/11/2026. Método: revisión documental (entregables 1–4), inspección del repositorio e historial, análisis estático y ejecución de pruebas. Es una autoauditoría del estudiante; no la realiza un tercero independiente. Leyenda: ● cumple, ◐ cumple en parte, ○ no cumple.

| **Lista**  | **Criterio**                                          | **Resultado** | **Evidencia**                                         |
|------------|-------------------------------------------------------|---------------|-------------------------------------------------------|
| Gestión    | Acta de constitución, alcance y EDT definidos         | ●             | Sección 3.1                                           |
| Gestión    | Riesgos identificados con respuesta                   | ●             | Sección 3.1 (8 riesgos)                               |
| Gestión    | Planificación por iteraciones y seguimiento visible   | ◐             | Sprints planificados, sin tablero previo              |
| Gestión    | Registro de reuniones y decisiones                    | ○             | No existen actas                                      |
| Ingeniería | Requisitos trazables a historias y diseño             | ●             | Entregable 1                                          |
| Ingeniería | Modelo de datos documentado y seguro                  | ●             | Entregable 2 (76 tablas, roles, RLS)                  |
| Ingeniería | Arquitectura por capas y despliegue reproducible      | ●             | Entregable 3                                          |
| Ingeniería | Gestión de configuración (ramas, etiquetas, revisión) | ◐             | 115 commits; sin revisión por pares ni etiquetas      |
| Software   | Pruebas automatizadas en integración continua         | ○             | 67 pruebas ejecutadas a mano; CI propuesta            |
| Software   | Seguridad: autenticación, autorización y secretos     | ◐             | JWT RS256, 173 @PreAuthorize; H-01, H-02, H-03        |
| Software   | Calidad de código y deuda bajo control                | ◐             | CCN media baja; cola con CCN 99                       |
| Software   | Operación: respaldo, reversión, monitoreo             | ◐             | Respaldo verificado; monitoreo y reversión por probar |

**Tabla 12. Autoevaluación de madurez de procesos (escala CMMI: 1 Inicial, 2 Gestionado, 3 Definido). Es una revisión informal inspirada en el método SCAMPI clase C, no una evaluación SCAMPI formal con un evaluador líder.**

| **Área de proceso**                         | **Nivel actual** | **Evidencia principal**                           | **Meta Sprint 2** |
|---------------------------------------------|------------------|---------------------------------------------------|-------------------|
| Gestión de requisitos (REQM)                | 2                | RF/RNF/HU con trazabilidad (E1)                   | 3                 |
| Planificación del proyecto (PP)             | 2                | Charter, EDT, backlog y sprints                   | 2                 |
| Seguimiento y control (PMC)                 | 1                | Solo historial git; sin tablero ni métricas       | 2                 |
| Gestión de riesgos (RSKM)                   | 2                | Registro de 8 riesgos                             | 2                 |
| Gestión de configuración (CM)               | 2                | Repositorio versionado; sin revisión ni etiquetas | 3                 |
| Aseguramiento de calidad del proceso (PPQA) | 1                | Listas definidas, aún sin aplicarse               | 2                 |
| Medición y análisis (MA)                    | 2                | Métricas de este entregable                       | 2                 |
| Verificación (VER)                          | 1                | Cobertura ≈ 0,5 %; sin revisión por pares         | 2                 |
| Validación (VAL)                            | 1                | Sin pruebas de aceptación con usuarios            | 2                 |
| Solución técnica e integración (TS/PI)      | 2                | Arquitectura por capas; despliegue reproducible   | 3                 |
| Promedio                                    | 1,6              |                                                   | 2,3               |

### 4.5 Hallazgos

| **ID** | **Hallazgo**                                                                   | **Severidad** | **Evidencia**                                                                     |
|--------|--------------------------------------------------------------------------------|---------------|-----------------------------------------------------------------------------------|
| H-01   | POST /api/auth/register es público y asigna los roles que envía el solicitante | Crítica       | SecurityConfig, AuthService.register                                              |
| H-02   | Endpoints /api/test-notifications/\*\* abiertos (permitAll)                    | Media         | SecurityConfig                                                                    |
| H-03   | Llaves JWT de desarrollo versionadas en el backend                             | Alta          | Llaves JWT de desarrollo versionadas en el backend (deploy/keys sí está ignorada) |
| H-04   | Backend sin pruebas ejecutadas ni cobertura medida (3 clases de prueba)        | Alta          | Sección 1.1                                                                       |
| H-05   | crearVenta con CCN 99 y ≈ 457 líneas                                           | Media         | lizard                                                                            |
| H-06   | Flyway desactivado: el esquema no se versiona                                  | Media         | Configuración de Spring                                                           |
| H-07   | UIT 2026 desactualizada (5 150 en lugar de 5 500) en quinta categoría          | Media         | Figura 2                                                                          |
| H-08   | console.log (101), alert() (13), any (910), catch(Exception) (81)              | Media         | Tabla 10                                                                          |
| H-09   | OnPush solo en 4 de 143 componentes                                            | Baja          | Búsqueda por patrón                                                               |
| H-10   | No se encontró workflow de CI/CD en el repositorio revisado                    | Alta          | Estructura del repositorio                                                        |
| H-11   | Bundle inicial de 812 kB (presupuesto 500 kB)                                  | Baja          | ng build                                                                          |
| H-12   | Solo 31 % de los commits con convención; mensajes sin información              | Baja          | Historial git                                                                     |

### 4.6 Plan de mejora

| **ID** | **Acción**                                                                | **Hallazgo**     | **Responsable**    | **Fecha**  | **Criterio de cierre**                        |
|--------|---------------------------------------------------------------------------|------------------|--------------------|------------|-----------------------------------------------|
| M-01   | Restringir registro de usuarios y cerrar endpoints de prueba              | H-01, H-02       | Omar               | 24/11/2026 | Prueba: POST anónimo devuelve 401             |
| M-02   | Rotar llaves JWT y sacarlas del repositorio e historial                   | H-03             | Omar               | 26/11/2026 | Llaves nuevas por secreto; repositorio limpio |
| M-03   | Publicar CI y corregir la UIT                                             | H-10, H-07       | Omar               | 01/12/2026 | Run verde en Actions; caso xit → it           |
| M-04   | Copia externa del respaldo (3-2-1) y prueba de restauración mensual       | R-04             | Omar               | 08/12/2026 | Respaldo en bucket externo restaurado         |
| M-05   | Ejecutar pruebas Java, medir JaCoCo y llegar a 60 % en servicios críticos | H-04             | Omar y colaborador | 15/12/2026 | Informe JaCoCo ≥ 60 %                         |
| M-06   | Refactorizar crearVenta                                                   | H-05             | Omar               | 15/12/2026 | Todos los métodos con CCN ≤ 15                |
| M-07   | Activar Flyway con migración base                                         | H-06             | Omar               | 22/12/2026 | Arranque con Flyway sin errores               |
| M-08   | Tablero visible, revisión por pares y convención de commits               | H-12, PMC, VER   | Equipo             | 01/12/2026 | Tablero y 100 % de PR revisados               |
| M-09   | Reducir smells de frontend y bundle                                       | H-08, H-09, H-11 | Omar               | 19/01/2027 | console.log = 0; bundle ≤ 500 kB              |

*Las fechas son compromisos propuestos por el estudiante, sujetos a confirmación del docente.*

## Anexos

### Anexo A. Paquete de artefactos entregado

| **Archivo**                                                               | **Contenido**                                                                         |
|---------------------------------------------------------------------------|---------------------------------------------------------------------------------------|
| pruebas_frontend/                                                         | karma.conf.js y los 7 archivos .spec.ts (37 casos) para copiar a finanzas-frontend    |
| db/09_pruebas_bd.sql, carga_venta.pgbench, consulta_reporte.pgbench       | Suite SQL de 30 casos y los dos escenarios de pgbench                                 |
| db/03_integridad_indices.sql, 07_seguridad_roles.sql                      | Scripts del Entregable 2 corregidos por los hallazgos de la suite                     |
| .github/workflows/ci.yml                                                  | Pipeline de 5 jobs                                                                    |
| ops/respaldo.sh, restaurar_prueba.sh, desplegar.sh, rollback.sh, salud.sh | Respaldo, restauración, despliegue con reversión y sonda de salud                     |
| observabilidad/application-prod.properties, alertas.yml                   | Configuración de Actuator, logs y 6 reglas de alerta                                  |
| evidencias/                                                               | Salidas originales de las ejecuciones (pruebas, cobertura, pgbench, respaldo, lizard) |

### Anexo B. Cómo reproducir las mediciones

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<tbody>
<tr class="odd">
<td><p># Frontend</p>
<p>cd finanzas-frontend &amp;&amp; npm ci</p>
<p>npx ng test --watch=false --browsers=ChromeHeadlessCI --code-coverage</p>
<p># Base de datos</p>
<p>cd db/sql &amp;&amp; bash 00_ejecutar_todo.sh &amp;&amp; psql -d laboratorio_db_demo -f 09_pruebas_bd.sql</p>
<p>pgbench -n -f carga_venta.pgbench -c 4 -j 2 -T 15 lab_bench</p>
<p># Complejidad</p>
<p>pip install lizard &amp;&amp; lizard finanzas-backend/src -l java &amp;&amp; lizard finanzas-frontend/src -l typescript</p></td>
</tr>
</tbody>
</table>

### Anexo C. Evidencias pendientes de captura

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<tbody>
<tr class="odd">
<td><p><strong>[ Pestaña Actions con el pipeline en verde ]</strong></p>
<p><em>Fecha y número de ejecución (a completar tras publicar ci.yml)</em></p></td>
</tr>
</tbody>
</table>

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<tbody>
<tr class="odd">
<td><p><strong>[ Panel de métricas o monitor de disponibilidad en producción ]</strong></p>
<p><em>URL monitoreada, fecha y captura (requiere la VM desplegada)</em></p></td>
</tr>
</tbody>
</table>

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<tbody>
<tr class="odd">
<td><p><strong>[ Tablero del backlog y acta de revisión del Sprint 1 ]</strong></p>
<p><em>Captura del tablero con las tablas 6 y 7; acta firmada por quien asista</em></p></td>
</tr>
</tbody>
</table>

### Anexo D. Correspondencia con la rúbrica

| **Criterio**                         | **Dónde se atiende** | **Estado honesto**                                            |
|--------------------------------------|----------------------|---------------------------------------------------------------|
| Cobertura y calidad de pruebas       | 1.1 – 1.5            | 67 pruebas ejecutadas; cobertura baja y declarada             |
| Automatización CI/CD                 | 2.1 – 2.3            | Diseñada y validada en sintaxis; ejecución pendiente          |
| Operación y observabilidad           | 2.4 – 2.5            | Respaldo probado; monitoreo propuesto                         |
| Gestión ágil                         | 3.1 – 3.4            | Charter, EDT, backlogs, métricas reales; sprints planificados |
| Adaptación de marcos                 | 3.5                  | RUP frente a Scrum y adaptación híbrida                       |
| Aseguramiento de calidad del proceso | 4.1 – 4.2            | Matriz ISO 25010 y listas de verificación                     |
| Auditoría de madurez y deuda técnica | 4.3 – 4.6            | Autoevaluación CMMI, 12 hallazgos, 136 h de deuda, plan       |
| Sustentación                         | Anexo E              | A completar                                                   |

### Anexo E. Evidencia de sustentación

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<tbody>
<tr class="odd">
<td><p><strong>[ Evidencia de sustentación ]</strong></p>
<p><em>Fecha, docente evaluador, observaciones y firma (a completar por el estudiante)</em></p></td>
</tr>
</tbody>
</table>

---

## Material complementario

- [Artefactos del entregable (ZIP: CI, scripts, pruebas)](../descargas/ce02/CE024_S2_E4_Artefactos.zip)
