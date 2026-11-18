# CE0331 - S1 - E2: Análisis y Diseño de Centro de Datos Híbrido

**Equipo:** Omar Condori Pachauri (201810211) · Merry Elizabeth Flores Llanque (202211953) · Jaqueline Huahuaccapa Ccama (202210712) — Ciclo 10.

## 1. Resumen ejecutivo del diseño del centro de datos

**Proyecto y organización.** Finanzas EPE es la plataforma de gestión financiera y administrativa de un laboratorio clínico (ventas, caja, compras, kardex, planilla, honorarios y auditoría). Este entregable analiza y diseña el «centro de datos» que la aloja. La plantilla de la línea asume un centro de datos híbrido con VMware ESXi y AWS; el proyecto real se ejecuta en la nube pública de Render, por lo que cada elemento se documenta en su equivalente real y se declara lo que no existe.

**Problema de capacidad, disponibilidad y costos.** El sistema opera con recursos mínimos y costo cero: un Web Service Docker de 512 MB de RAM y una base PostgreSQL gratuita. De esa decisión de costos nacen tres riesgos medibles: (1) el servicio se suspende tras 15 minutos sin tráfico; (2) los topes de memoria configurados para la JVM suman 508 MB de un límite de 512 MB; (3) el panel de Render informa que la base de datos finanzas-db se eliminará el 17/12/2026 si no se pasa a un plan de pago, y el plan gratuito no incluye respaldos administrados.

**Tecnología local y cloud.** Local: equipos de desarrollo con Angular y Spring Boot, donde se compila el frontend y se empaqueta junto al backend. Cloud: proyecto «My project», entorno «Production» de Render, con el Web Service FinanzasEPE (Docker, rama main de GitHub) y la base finanzas-db (PostgreSQL 18, región Oregon).

**Dimensionamiento y siguientes pasos.** El diseño propone mantener el dimensionamiento actual para la demostración, medir el consumo real con las métricas de Render, y antes del 17/12/2026 ejecutar un respaldo externo y decidir la migración a un plan de pago. El costo mínimo sostenible, según la lista de precios de Render, es de 13 USD al mes (Web Service de menos de 1 CPU y 512 MB: 7 USD; PostgreSQL de 256 MB: 6 USD). La implementación y la prueba de recuperación corresponden al entregable siguiente.

<img src="../../img_s1-e2-diseno-centro-de-datos/media/23fbcb950e54c55bbbd78a73f8a5fdd3494d0fe1.png" title="Figura 1.1 — Arquitectura del centro de datos en Render (vista ejecutiva)." style="width:3.4375in;height:2.86458in" alt="Figura 1.1 — Arquitectura del centro de datos en Render (vista ejecutiva)." />

*Figura 1.1 — Arquitectura del centro de datos en Render (vista ejecutiva).*

|                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Estado de la evidencia.** Los datos provienen del informe técnico de despliegue del equipo (v1.0.4) y de capturas del panel de Render del 17/11/2026 (figuras 7.1 y 7.2). Los límites de la plataforma y los precios se toman de la documentación y la lista de precios oficiales de Render \[1\], \[2\]. Lo que no se puede verificar (región del Web Service, uso real de memoria) se marca como pendiente. Las proyecciones de crecimiento son estimaciones y se rotulan así. |

## 2. Descripción de la organización y necesidades de infraestructura

| **Aspecto**                   | **Descripción**                                                                                                                                                                                                                                                                            |
|-------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Servicios a alojar            | Aplicación web (SPA Angular servida por Spring Boot), API REST con autenticación JWT, base de datos PostgreSQL, tarea de poblado inicial de datos (DemoDataInitializer)                                                                                                                    |
| Información gestionada        | Ventas y comprobantes con correlativos, cajas y arqueos, compras e inventario, planilla y honorarios, usuarios y permisos, bitácora de auditoría. Hoy son datos de demostración (una sede sembrada, 2026); con datos reales incluiría datos personales de pacientes y personal (Ley 29733) |
| Uso previsto de cloud y local | Producción 100 % en la nube (Render). El entorno local se usa solo para desarrollo, compilación del frontend y pruebas                                                                                                                                                                     |
| Crecimiento estimado          | Pocos usuarios simultáneos (personal administrativo de una empresa). El crecimiento de datos es lento (sección 10.3); el límite que condiciona el crecimiento es el plan gratuito, no el volumen                                                                                           |
| Disponibilidad esperada       | Horario laboral; aceptable un arranque en frío de hasta un minuto en la demostración, no aceptable en operación real                                                                                                                                                                       |

| **Servicio**                       | **Proceso que soporta**                        | **Criticidad** |
|------------------------------------|------------------------------------------------|----------------|
| Web Service FinanzasEPE            | Acceso de todos los usuarios; ventas y cobros  | Alta           |
| PostgreSQL finanzas-db             | Persistencia de todos los procesos y auditoría | Alta           |
| Repositorio GitHub + build         | Entrega de nuevas versiones                    | Media          |
| Logs, eventos y métricas de Render | Diagnóstico y operación                        | Media          |

## 3. Alcance del diseño

| **Tipo**         | **Dentro del alcance**                                                                                                                        | **Exclusiones justificadas**                                                                                |
|------------------|-----------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| Físicos          | Equipos de desarrollo del equipo. La infraestructura física del centro de datos es del proveedor (Render) y no se diseña                      | Rack, UPS, switches y almacenamiento propios: no existen en el proyecto                                     |
| Tecnológicos     | Docker multi-etapa, JVM, Web Service, PostgreSQL 18, GitHub, variables de entorno, monitoreo de Render, estrategia de respaldo y recuperación | VMware ESXi, vSwitches, VLAN, AWS (VPC, EC2, NACL, IAM), VPN IPsec: el proyecto no los usa y no se inventan |
| Organizacionales | El equipo administra código, Dockerfile, variables y datos; Render administra hardware, sistema base, red y certificados                      | Operación de la red del usuario final; asignación nominal de responsables: \[a completar por el equipo\]    |

## 4. Objetivos medibles del diseño

| **Objetivo**                             | **Indicador**                                       | **Meta**                       | **Línea base actual**                                     |
|------------------------------------------|-----------------------------------------------------|--------------------------------|-----------------------------------------------------------|
| Dimensionar el contenedor con margen     | Suma de topes de memoria JVM / RAM del plan         | ≤ 90 %                         | 99,2 % (508 de 512 MB): objetivo no cumplido              |
| Mantener el heap con holgura             | Heap máximo / RAM del plan                          | ≤ 55 %                         | 54,7 % (280 de 512 MB)                                    |
| Arquitectura costo-eficiente             | Costo mensual de infraestructura                    | Hoy 0 USD; sostenible ≤ 15 USD | 0 USD (plan gratuito)                                     |
| Disponibilidad mínima en horario laboral | Disponibilidad medida por sonda externa             | ≥ 99 %                         | Sin medir (suspensión por inactividad)                    |
| RTO de la aplicación                     | Tiempo para restablecer el Web Service              | ≤ 15 min                       | ≈ 6 min (despliegues manuales de 5 min 50 s y 5 min 59 s) |
| RPO de la base de datos                  | Antigüedad máxima del respaldo externo              | ≤ 24 h                         | Sin respaldo                                              |
| RTO de la base de datos                  | Tiempo de restauración en una base nueva            | ≤ 2 h                          | Sin probar                                                |
| Capacidad de la base                     | Uso del almacenamiento (1 GB fijo en plan gratuito) | ≤ 50 %                         | A medir con pg_database_size                              |

## 5. Arquitectura del entorno local (desarrollo y construcción)

El equivalente de la arquitectura on-premise es el entorno local donde se desarrolla y se prepara cada versión. El flujo real, descrito en el informe de despliegue, es:

<table>
<colgroup>
<col style="width: 50%" />
<col style="width: 50%" />
</colgroup>
<tbody>
<tr class="odd">
<td><p><img src="../../img_s1-e2-diseno-centro-de-datos/media/b5bf102af3278968f01d2db2a3ca4e7eb3bd4768.png" title="Figura 5.1 — Flujo del entorno local a la nube." style="width:2.44792in;height:4.42708in" alt="Figura 5.1 — Flujo del entorno local a la nube." /></p>
<p><em>Figura 5.1 — Flujo del entorno local a la nube.</em></p></td>
<td><p><img src="../../img_s1-e2-diseno-centro-de-datos/media/d9454a7bb3ea9ca1d19cf9e9e10d78e91893b4ab.png" title="Figura 5.2 — Continuidad planificada de la base de datos (sección 11)." style="width:2.44792in;height:4.3125in" alt="Figura 5.2 — Continuidad planificada de la base de datos (sección 11)." /></p>
<p><em>Figura 5.2 — Continuidad planificada de la base de datos (sección 11).</em></p></td>
</tr>
</tbody>
</table>

*No hay servidores locales ni red local de producción. Datos del equipo (modelo, sistema operativo, versiones de herramientas): \[a completar por el equipo\].*

## 6. Diseño del entorno de ejecución del contenedor

La plantilla pide diseñar el hipervisor (vSwitches, port groups, NIC teaming, máquinas virtuales). En Render el hipervisor es del proveedor; lo que el equipo diseña es la imagen Docker y los límites de memoria de la JVM, que son el equivalente de dimensionar una máquina virtual.

**Tabla 1. Etapas de la imagen Docker.**

| **Etapa**   | **Imagen base**                                                             | **Función**                                                      |
|-------------|-----------------------------------------------------------------------------|------------------------------------------------------------------|
| 1 — Build   | maven:3.9-eclipse-temurin-21 (MAVEN_OPTS=-Xmx1024m, compilación escalonada) | Compila y empaqueta el backend con mvn clean package -DskipTests |
| 2 — Runtime | eclipse-temurin:21-jre-alpine                                               | Ejecuta app.jar; imagen ligera; escucha en el puerto 10000       |

**Tabla 2. Parámetros de la JVM (Dockerfile vigente tras corregir el error 502).**

| **Parámetro**             | **Valor**       | **Propósito**                                                |
|---------------------------|-----------------|--------------------------------------------------------------|
| -XX:+UseContainerSupport  | activo          | La JVM respeta los límites del contenedor                    |
| -XX:+UseSerialGC          | activo          | Recolector de un solo hilo: menor consumo en CPU compartida  |
| -Xms / -Xmx               | 128 MB / 280 MB | Heap inicial y máximo                                        |
| -XX:MaxMetaspaceSize      | 180 MB          | Tope de metadatos de clases                                  |
| -XX:ReservedCodeCacheSize | 48 MB           | Memoria para código compilado                                |
| -Xss                      | 512 KB          | Tamaño de pila por hilo (con 256 KB hubo fallos de arranque) |

<img src="../../img_s1-e2-diseno-centro-de-datos/media/35f64a38751e1e703053a748f463a167b458bf8f.png" title="Figura 6.1 — Topes de memoria configurados frente al límite de 512 MB del plan." style="width:5.41667in;height:1.46875in" alt="Figura 6.1 — Topes de memoria configurados frente al límite de 512 MB del plan." />

*Figura 6.1 — Topes de memoria configurados frente al límite de 512 MB del plan.*

**Análisis.** Los topes configurados suman 508 MB (280 + 180 + 48), y quedan 4 MB para pilas de hilos, memoria nativa y el propio contenedor. Es un tope teórico: el metaspace rara vez alcanza su máximo, pero el diseño no tiene margen ante un pico y el error 502 ya ocurrido fue un problema de memoria. Recomendación de diseño: medir el uso real de memoria en la pestaña Metrics de Render y, con ese dato, reducir el tope de metaspace para que la suma de topes no pase del 90 % del límite (objetivo de la sección 4).

El poblado de datos de demostración también se rediseñó: se retiró la transacción global del método de arranque para que Hibernate no retuviera todas las entidades en memoria durante el inicio.

## 7. Arquitectura en la nube (Render)

### 7.1 Organización de los recursos

Render organiza los recursos en espacio de trabajo, proyecto y entorno. El proyecto «My project», entorno «Production», contiene dos servicios. Las capturas siguientes son la evidencia del panel:

<img src="../../img_s1-e2-diseno-centro-de-datos/media/66c0d601c72cf08881ed45325ff1fcbba1d1b148.png" title="Figura 7.1 — Web Service FinanzasEPE: Docker, plan Free, repositorio Jaqueline2101/FinanzasEPE, rama main, commit 4901fa3 «Live», aviso de suspensión por inactividad (50 s o más)." style="width:5.20833in;height:2.5in" alt="Figura 7.1 — Web Service FinanzasEPE: Docker, plan Free, repositorio Jaqueline2101/FinanzasEPE, rama main, commit 4901fa3 «Live», aviso de suspensión por inactividad (50 s o más)." />

*Figura 7.1 — Web Service FinanzasEPE: Docker, plan Free, repositorio Jaqueline2101/FinanzasEPE, rama main, commit 4901fa3 «Live», aviso de suspensión por inactividad (50 s o más).*

<img src="../../img_s1-e2-diseno-centro-de-datos/media/f5ae9369df6aa528984934a24725396a22361b27.png" title="Figura 7.2 — Base finanzas-db: plan Free, PostgreSQL 18, región Oregon (US West), disponible; aviso de eliminación el 17/12/2026." style="width:5.20833in;height:2.59375in" alt="Figura 7.2 — Base finanzas-db: plan Free, PostgreSQL 18, región Oregon (US West), disponible; aviso de eliminación el 17/12/2026." />

*Figura 7.2 — Base finanzas-db: plan Free, PostgreSQL 18, región Oregon (US West), disponible; aviso de eliminación el 17/12/2026.*

### 7.2 Equivalencias entre la plantilla (AWS) y Render

| **Elemento de la plantilla (AWS)** | **Equivalente real en Render**                                                                                         | **Estado**         |
|------------------------------------|------------------------------------------------------------------------------------------------------------------------|--------------------|
| Región                             | Oregon (US West) para la base de datos; región del Web Service \[completar desde su panel\]                            | Verificada (BD)    |
| VPC y CIDR                         | Red de la plataforma, sin direccionamiento definido por el equipo                                                      | No configurable    |
| Subredes públicas y privadas       | No existen; el servicio es público y la base se accede por JDBC                                                        | No aplica          |
| Internet Gateway                   | Borde público de Render (HTTPS, dominio onrender.com)                                                                  | En operación       |
| NAT Gateway, VPN Gateway           | No existen ni se requieren                                                                                             | No aplica          |
| Instancia EC2                      | Web Service Docker «FinanzasEPE», plan Free                                                                            | En operación       |
| RDS / base de datos                | Render Postgres «finanzas-db», plan Free, versión 18                                                                   | En operación       |
| Security Groups y NACL             | Reglas de acceso de la base de datos y restricciones del servicio gratuito (no recibe tráfico de la red privada) \[2\] | Reglas por revisar |
| IAM                                | Accesos al workspace de Render y a GitHub; autenticación de la aplicación por JWT                                      | En operación       |
| CloudWatch                         | Events, Logs y Metrics de Render (visibles en el panel)                                                                | Disponible         |

**Región.** La base está en Oregon, lejos de los usuarios en Perú; no hay una latencia medida. Se medirá con la sonda externa del plan de monitoreo y se decidirá si conviene otra región al migrar a un plan de pago. La justificación de la elección original queda por documentar por el equipo.

## 8. Mecanismo de integración entre el entorno local y la nube

La integración «híbrida» de este proyecto es la cadena local → GitHub → Render, coherente con los enlaces ④, ⑤ y ⑥ del Entregable 1. No existe VPN IPsec: la comunicación entre el Web Service y la base de datos ocurre dentro de la plataforma por JDBC (enlace ③).

**Tabla 3. Despliegues observados en el panel de Render (30 despliegues en total; se listan los 7 visibles).**

| **Commit** | **Mensaje (resumen)**                                         | **Disparador** | **Resultado** | **Duración** |
|------------|---------------------------------------------------------------|----------------|---------------|--------------|
| 4901fa3    | docs: informe de despliegue y datos de demostración           | Manual         | Live (activo) | 5 min 50 s   |
| 4901fa3    | igual                                                         | Manual         | Cancelado     | 0,6 s        |
| 4901fa3    | igual                                                         | Auto-Deploy    | Cancelado     | 1 min 48 s   |
| f4224b5    | feat: vincular ventas a cta. cte. y personal                  | Auto-Deploy    | Fallido       | 33,7 s       |
| e61f181    | fix: aumentar heap de JVM y quitar transacción de clase (502) | Manual         | Exitoso       | 5 min 59 s   |
| e61f181    | igual                                                         | Manual         | Cancelado     | 0,3 s        |
| e61f181    | igual                                                         | Auto-Deploy    | Cancelado     | 52,3 s       |

Lecturas: (1) el despliegue completo tarda cerca de 6 minutos, valor que se usa como estimación de RTO de la aplicación; (2) cada versión genera un auto-despliegue que el equipo cancela al lanzar uno manual, lo que desperdicia minutos de build; (3) el panel ofrece «Rollback» a despliegues anteriores, mecanismo real de reversión; (4) un despliegue falló en 33,7 s y no llegó a producción, lo que muestra que la plataforma no reemplaza el servicio activo cuando el build falla.

## 9. Matriz de componentes del centro de datos

| **Componente**                       | **Entorno**         | **Tipo**                    | **Función**                                   | **Criticidad** |
|--------------------------------------|---------------------|-----------------------------|-----------------------------------------------|----------------|
| Equipos de desarrollo                | Local               | Estación de trabajo         | Desarrollo, build del frontend, pruebas       | Media          |
| Repositorio GitHub (main)            | Externo             | Servicio SaaS               | Código fuente y disparador de despliegues     | Media          |
| Build Docker (maven:3.9-temurin-21)  | Render              | Proceso de build            | Compila y empaqueta app.jar                   | Media          |
| Imagen eclipse-temurin:21-jre-alpine | Render              | Imagen de contenedor        | Runtime de la aplicación                      | Alta           |
| Web Service FinanzasEPE              | Render (Production) | Servicio, plan Free, 512 MB | Sirve SPA y API en el puerto 10000            | Alta           |
| JVM Java 21                          | Render              | Runtime                     | Ejecuta Spring Boot con topes de memoria      | Alta           |
| PostgreSQL finanzas-db (v18)         | Render, Oregon      | Base gestionada, plan Free  | Persistencia y auditoría                      | Alta           |
| Variables de entorno                 | Render              | Configuración               | SPRING_DATASOURCE_URL/USERNAME/PASSWORD, PORT | Alta           |
| Dominio y TLS onrender.com           | Render              | Red/seguridad               | Acceso HTTPS público                          | Alta           |
| Events, Logs, Metrics                | Render              | Monitoreo                   | Diagnóstico operativo                         | Media          |
| Respaldo externo (pg_dump)           | Externo             | Planificado                 | Continuidad de datos                          | Alta           |

## 10. Dimensionamiento y capacidad

### 10.1 Recursos actuales

| **Recurso**                  | **Valor**                                                                         | **Fuente**                                              |
|------------------------------|-----------------------------------------------------------------------------------|---------------------------------------------------------|
| Web Service: RAM             | 512 MB (plan Free)                                                                | Lista de precios de Render \[2\]; informe de despliegue |
| Web Service: CPU             | Compartida; no especificada en la documentación consultada                        | Documentación de Render \[1\]                           |
| Web Service: horas           | 750 horas gratuitas por mes y por workspace; se suspende a los 15 min sin tráfico | Render \[1\]                                            |
| Web Service: disco           | Efímero; se pierde al reiniciar o redesplegar                                     | Render \[1\]                                            |
| PostgreSQL: RAM y conexiones | 256 MB y 100 conexiones (plan Free)                                               | Lista de precios de Render \[2\]                        |
| PostgreSQL: almacenamiento   | 1 GB fijo; vence a los 30 días y se elimina 14 días después si no se actualiza    | Render \[1\]; el panel indica el 17/12/2026             |
| Versión de PostgreSQL        | 18                                                                                | Captura del panel (figura 7.2)                          |

### 10.2 Carga esperada

Pocos usuarios administrativos simultáneos. El pool de conexiones de Spring Boot (HikariCP, 10 conexiones por defecto) usa el 10 % de las 100 permitidas, por lo que las conexiones no son el límite. Con una sola instancia de 512 MB, el factor limitante es la memoria (sección 6).

### 10.3 Crecimiento de datos (estimación)

Medición propia en una base PostgreSQL local con un esquema análogo de 76 tablas y 3 000 ventas sintéticas: cada venta con su detalle ocupa cerca de 521 bytes incluyendo índices, y la base vacía con su esquema ocupa unos 13 MB. Es una medición de orden de magnitud sobre otro esquema; debe repetirse con pg_database_size en finanzas-db.

| **Escenario (ventas por día)** | **Ventas por año** | **Crecimiento anual solo de ventas** | **Uso del 1 GB gratuito** |
|--------------------------------|--------------------|--------------------------------------|---------------------------|
| 50                             | 18 250             | ≈ 9 MB                               | ≈ 2 % con la base inicial |
| 200                            | 73 000             | ≈ 38 MB                              | ≈ 5 %                     |
| 1 000                          | 365 000            | ≈ 190 MB                             | ≈ 20 %                    |

Conclusión: el almacenamiento de 1 GB no es el cuello de botella en ninguno de los escenarios (sin contar las demás tablas, que se miden después); el límite real es el vencimiento de la base gratuita.

### 10.4 Costo y justificación técnica

*Montos de la lista de precios de Render \[2\], sin tarifa de espacio de trabajo ni impuestos; confirmar al decidir.*

| **Opción**                                                    | **Costo mensual**      | **Qué resuelve**                                                                                                                  |
|---------------------------------------------------------------|------------------------|-----------------------------------------------------------------------------------------------------------------------------------|
| Actual: Web Service Free + PostgreSQL Free                    | 0 USD                  | Demostración; con suspensión y vencimiento de la base                                                                             |
| Web Service de menos de 1 CPU y 512 MB + PostgreSQL de 256 MB | 7 + 6 = 13 USD \[2\]   | Elimina la suspensión y el vencimiento; la base paga incluye recuperación a un punto en el tiempo (3 días en el plan Hobby) \[2\] |
| Web Service de 1 CPU y 2 GB + PostgreSQL de 1 GB              | 25 + 19 = 44 USD \[2\] | Margen de memoria amplio para crecimiento                                                                                         |

## 11. Disponibilidad y continuidad

**Tabla 4. Objetivos de continuidad por servicio y estado actual.**

| **Servicio**             | **RTO objetivo**      | **RPO objetivo**       | **Mecanismo**                                                                                             | **Estado actual**               |
|--------------------------|-----------------------|------------------------|-----------------------------------------------------------------------------------------------------------|---------------------------------|
| Web Service (aplicación) | ≤ 15 min              | No aplica (sin estado) | Rollback o redespliegue desde GitHub (≈ 6 min medidos)                                                    | Existe; prueba formal pendiente |
| Base de datos            | ≤ 2 h                 | ≤ 24 h                 | pg_dump diario a almacenamiento externo; restauración en una base nueva y cambio de SPRING_DATASOURCE\_\* | No existe: brecha crítica       |
| Código fuente            | ≤ 1 h                 | ≈ 0 (cada push)        | Repositorio GitHub                                                                                        | Existe                          |
| Acceso público (dominio) | Depende del proveedor | —                      | Dominio onrender.com con TLS administrado                                                                 | Existe                          |

#### Brechas y plan

| **ID** | **Brecha**                                                      | **Acción**                                                                              | **Plazo**            |
|--------|-----------------------------------------------------------------|-----------------------------------------------------------------------------------------|----------------------|
| C-01   | La base finanzas-db se elimina el 17/12/2026 si no se actualiza | Respaldo externo inmediato y decisión de migrar a un plan de pago antes de esa fecha    | Antes del 10/12/2026 |
| C-02   | Sin respaldos administrados en el plan gratuito \[1\]           | Script pg_dump programado (diario) con verificación y copia fuera de Render             | Entregable 3         |
| C-03   | Suspensión tras 15 minutos de inactividad                       | Sonda de disponibilidad externa o plan de pago                                          | Entregable 3         |
| C-04   | Auto-despliegues cancelados a mano                              | Elegir un solo modo de despliegue (automático o manual) para no gastar minutos de build | Inmediato            |
| C-05   | Sin simulacro de recuperación                                   | Restaurar el respaldo en una base nueva y medir RTO/RPO                                 | Entregable 3         |

La validación esperada en el semestre siguiente es un ejercicio completo de recuperación documentado: restaurar la base desde el respaldo externo, reapuntar el servicio y comprobar el inicio de sesión y una venta de prueba, registrando los tiempos obtenidos frente a los objetivos de la tabla 4.

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

## 12. Conclusiones y hoja de ruta

Se diseñó el centro de datos de Finanzas EPE tal como existe: dos servicios gestionados en Render y una cadena de entrega desde un entorno local, sin inventar infraestructura on-premise ni AWS. El análisis encontró tres hallazgos cuantificados: la suma de topes de memoria deja solo 4 MB de margen, el RTO de la aplicación es de cerca de 6 minutos y la base de datos desaparecerá el 17/12/2026 sin una acción del equipo.

| **Paso** | **Acción**                                                                          | **Entregable**       |
|----------|-------------------------------------------------------------------------------------|----------------------|
| 1        | Respaldo externo y decisión sobre el plan de pago (C-01, C-02)                      | Antes del 10/12/2026 |
| 2        | Medir memoria y almacenamiento con Metrics y pg_database_size; ajustar el metaspace | Entregable 3         |
| 3        | Sonda de disponibilidad y alertas (C-03)                                            | Entregable 3         |
| 4        | Simulacro de recuperación y registro de RTO/RPO (C-05)                              | Entregable 3         |
| 5        | Revisar reglas de acceso de la base y asignar responsables                          | Entregable 3         |

## 13. Referencias

**\[1\]** Render, «Deploy for Free», Render Docs. \[En línea\]. Disponible: https://render.com/docs/free (consultado el 17/11/2026).

**\[2\]** Render, «Pricing». \[En línea\]. Disponible: https://render.com/pricing (consultado el 17/11/2026).

**\[3\]** Equipo del proyecto, «Informe Técnico Integral: Proceso de Desarrollo, Solución de Errores, Poblado Masivo de Datos y Despliegue en Render», Finanzas EPE, v1.0.4, 2026.

**\[4\]** Docker Inc., «Multi-stage builds», Docker Docs. \[En línea\]. Disponible: https://docs.docker.com/build/building/multi-stage/

**\[5\]** The PostgreSQL Global Development Group, «PostgreSQL Documentation: Backup and Restore». \[En línea\]. Disponible: https://www.postgresql.org/docs/current/backup.html

**\[6\]** ISO/IEC, «ISO/IEC 27002:2022 — Information security controls», 2022.

**\[7\]** Congreso de la República del Perú, «Ley N.º 29733, Ley de Protección de Datos Personales», 2011.

---

## Material complementario

- [Diagramas (ZIP: png y dot)](../descargas/ce03/CE0331_S1_E2_Diagramas.zip)
