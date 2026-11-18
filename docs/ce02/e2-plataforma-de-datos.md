# CE022 - S2 - E2: Plataforma de Datos del Sistema

**Equipo:** Omar Condori Pachauri (201810211) · Merry Elizabeth Flores Llanque (202211953) · Jaqueline Huahuaccapa Ccama (202210712) — Ciclo 10.

## Resumen ejecutivo

FINANZAS es el ERP administrativo-financiero de Reumacenter E.I.R.L. (laboratorio clínico, Juliaca). Todo lo que el sistema hace —vender un examen, cobrarlo en caja, anularlo con autorización, comprar insumos, valorizar el inventario, pagar honorarios y auditar a quien hizo cada cosa— termina siendo una fila en una base de datos PostgreSQL. Por eso la base de datos no es un componente más: es el registro contable de la empresa y la fuente de la que salen los comprobantes, los reportes financieros y la evidencia ante una auditoría.

Este entregable documenta esa plataforma de datos: su modelo (conceptual, lógico y normalizado), su implementación en SQL, la lógica programada dentro del motor (funciones, procedimientos y triggers), las pruebas de rendimiento y las medidas de seguridad, respaldo y monitoreo.

#### Resultados principales

- **Modelo:** 76 tablas relacionales (73 entidades JPA + 3 tablas puente) organizadas en cuatro dominios: seguridad y auditoría, ventas y caja, compras e inventario, y configuración.

- **Integridad:** se verificaron 70 claves foráneas, 76 claves primarias, 22 restricciones únicas y 8 restricciones CHECK; las 6 pruebas de violación de reglas fueron rechazadas por el motor (sección 2.4).

- **Programación en BD:** 2 funciones, 2 procedimientos, 3 triggers y 1 vista que protegen reglas críticas: caja sin saldo negativo, ventas anuladas inmutables, auditoría automática.

- **Rendimiento:** un índice compuesto (sucursal, fecha) redujo una consulta de reporte de 15,5 ms a 2,0 ms sobre 400 000 filas (7,6 veces más rápido), pasando de recorrido secuencial a índice.

- **Seguridad y continuidad:** tres roles de motor con mínimo privilegio, seguridad por fila (RLS) por sucursal, columnas sensibles ocultas y un respaldo con restauración verificada fila a fila.

|                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Alcance y transparencia de la evidencia.** El esquema del sistema real lo genera Hibernate (ddl-auto=update) y las migraciones Flyway están desactivadas (deuda técnica registrada como ADR-10 en el Entregable 1). Por eso el DDL de este entregable se obtuvo leyendo las anotaciones JPA de las 73 entidades, y toda la evidencia de ejecución proviene de una base de pruebas PostgreSQL 16 con datos sintéticos (3 000 ventas, 5 usuarios, 2 sucursales). No se usaron datos reales de pacientes. Los scripts son reejecutables sobre la base real para repetir las capturas. |

## 1. Modelo de datos

### 1.1 Alcance y fuente del modelo

El sistema usa PostgreSQL 16 con la base «laboratorio_db». El backend (Spring Boot 4, Java 21) accede mediante Spring Data JPA; las tablas se derivan de las entidades anotadas con @Entity y @Table. El modelo se construyó en tres pasos: (1) inventario de las 73 entidades y sus relaciones @ManyToOne, @OneToOne y @ManyToMany; (2) generación del DDL a partir de ese inventario (script 01 y 02); (3) revisión de las referencias que el ORM mapea como simples columnas numéricas (por ejemplo id_sucursal en venta_registro) para convertirlas en claves foráneas explícitas (script 03).

| **Dominio**                         | **Tablas principales**                                                                                                                                                                                 | **Responsabilidad**                                                           |
|-------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------|
| Seguridad y auditoría               | datos_personales, roles, permissions, user_roles, role_permissions, sessions, intrusion_attempts, lab_auditoria, segregacion_funciones, delegacion_temporal                                            | Identidad, RBAC, sesiones, bloqueo por intentos, trazabilidad                 |
| Ventas y caja                       | venta_registro, venta_detalle, autorizacion_descuento, autorizacion_descuento_detalle, nota_credito, solicitud_anulacion, caja_chica, caja_chica_movimiento, caja_general\*, metodos_pago, arqueo_caja | Comprobantes, descuentos autorizados, anulaciones, movimientos de caja        |
| Compras e inventario                | proveedores, ordenes_compra, orden_compra_detalles, compras, detalle_compra, productos, kardex, cuenta_proveedor\*                                                                                     | Abastecimiento, stock por sucursal, valorización                              |
| Pacientes, catálogo y configuración | catalogo, paquetes, niveles, tipo_paciente, cred_ctacte, cred_planes, datos_empresa, datos_companias, sucursal, menu_items, configuracion                                                              | Servicios y tarifas, cuentas corrientes de crédito, estructura organizacional |

### 1.2 Modelo entidad-relación (conceptual)

La figura 1 resume las entidades de negocio y su cardinalidad. Sucursal y Catálogo (en naranja) son los dos ejes compartidos: casi todo dato operativo pertenece a una sucursal, y todo lo que se vende o se stockea se describe en el catálogo.

<img src="../../img_e2-plataforma-de-datos/media/473940245d7c4986578c9dd78539fbbcdabf1ad0.png" title="Figura 1. Modelo entidad-relación conceptual por dominios." style="width:6.25in;height:3.76042in" alt="Figura 1. Modelo entidad-relación conceptual por dominios." />

*Figura 1. Modelo entidad-relación conceptual por dominios.*

### 1.3 Modelo lógico

Las figuras 2 a 5 muestran el modelo lógico de los cuatro dominios con claves primarias (azul), claves foráneas (naranja) y los atributos clave. Línea continua: clave foránea declarada por el ORM. Línea discontinua: clave foránea agregada en el script 03. Línea punteada: referencia lógica que hoy no tiene restricción en la base (candidata a endurecerse).

<img src="../../img_e2-plataforma-de-datos/media/d014f392cf2e738a0f0cfd2a998fa526d85817d8.png" title="Figura 2. Modelo lógico — ventas, catálogo y autorizaciones." style="width:4.89583in;height:4.4375in" alt="Figura 2. Modelo lógico — ventas, catálogo y autorizaciones." />

*Figura 2. Modelo lógico — ventas, catálogo y autorizaciones.*

<img src="../../img_e2-plataforma-de-datos/media/e66205f03555dfb7bf2ac10da23927a22fa84d40.png" title="Figura 3. Modelo lógico — caja chica." style="width:5.41667in;height:2.03125in" alt="Figura 3. Modelo lógico — caja chica." />

*Figura 3. Modelo lógico — caja chica.*

<img src="../../img_e2-plataforma-de-datos/media/38b2bafba837a023b47c9e9e511f9aa8d8aeea26.png" title="Figura 4. Modelo lógico — seguridad y auditoría (RBAC)." style="width:4.58333in;height:3.94792in" alt="Figura 4. Modelo lógico — seguridad y auditoría (RBAC)." />

*Figura 4. Modelo lógico — seguridad y auditoría (RBAC).*

<img src="../../img_e2-plataforma-de-datos/media/7518bda58aafb3bbe8be5728e875c264eac2e959.png" title="Figura 5. Modelo lógico — compras e inventario." style="width:4.89583in;height:4.57292in" alt="Figura 5. Modelo lógico — compras e inventario." />

*Figura 5. Modelo lógico — compras e inventario.*

### 1.4 Modelo normalizado

El diseño cumple tercera forma normal (3FN) en el núcleo transaccional. La tabla resume el análisis y las excepciones deliberadas.

| **Forma normal**                             | **Cómo se cumple**                                                             | **Ejemplo en el sistema**                                                                                                           |
|----------------------------------------------|--------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------|
| 1FN — valores atómicos, sin grupos repetidos | Cada columna guarda un solo valor; las líneas repetidas viven en tablas hijas. | Una venta con N servicios: venta_registro (cabecera) + venta_detalle (una fila por servicio). Igual en compras y órdenes de compra. |
| 2FN — sin dependencias parciales             | Las tablas puente usan clave compuesta y solo contienen la clave.              | user_roles(user_id, role_id) y role_permissions(role_id, permission_id) resuelven las relaciones N:M usuario-rol y rol-permiso.     |
| 3FN — sin dependencias transitivas           | Los datos descriptivos se guardan una vez y se referencian por clave.          | Nombre y precio del servicio están en catalogo; venta_detalle solo guarda id_articulo. Método de pago en metodos_pago.              |

#### Excepciones y decisiones de diseño

- **Atributos derivados guardados:** caja_chica.saldo_actual y venta_registro.total duplican información calculable. Se mantienen por rendimiento y para congelar el valor emitido; el trigger tg_caja_movimiento mantiene el saldo y la función fn_saldo_teorico_caja permite conciliarlo (consulta C5).

- **Desnormalización histórica en kardex:** guarda nombre de producto, sucursal y usuario además de sus claves. Es intencional: el kardex es un libro histórico y no debe cambiar si luego se renombra un producto.

- **Tabla ancha datos_personales:** concentra datos de persona, usuario de acceso y vínculo laboral (cargo, contrato, modalidad). Funciona, pero mezcla responsabilidades; se registra como mejora (separar usuario y vínculo laboral) para el siguiente ciclo.

- **Referencias lógicas sin FK:** venta_detalle.id_articulo, nota_credito.id_venta_ref y autorizacion_descuento.id_venta no tienen restricción. Se dejaron fuera del script 03 porque podrían existir datos históricos huérfanos en producción; antes de agregarlas hay que ejecutar una consulta de huérfanos.

### 1.5 Diccionario de datos

Se presentan las tablas críticas del negocio con sus columnas principales. El diccionario completo (1 457 columnas de 76 tablas, con tipo, nulabilidad, clave y referencia) se entrega como archivo diccionario_datos_completo.csv, generado automáticamente desde information_schema.

#### venta_registro — comprobante de venta (cabecera)

| **Columna**               | **Tipo**      | **Clave** | **Descripción**                                          |
|---------------------------|---------------|-----------|----------------------------------------------------------|
| id_venta                  | bigserial     | PK        | Identificador de la venta                                |
| id_sucursal               | bigint        | FK        | Sucursal que emite (sucursal)                            |
| fecha                     | date          |           | Fecha de emisión                                         |
| tipo_doc / serie / numdoc | varchar       |           | Comprobante; único por sucursal mientras no esté anulado |
| base_imp, igv, total      | numeric(12,4) |           | Base imponible, IGV y total (total ≥ 0)                  |
| descuento                 | numeric(12,4) |           | Descuento aplicado (≥ 0)                                 |
| estado                    | integer       |           | 1 = vigente, 10 = anulada (inmutable)                    |
| user_upd                  | varchar(50)   |           | Último usuario que modificó                              |

#### venta_detalle — líneas de la venta

| **Columna**            | **Tipo**      | **Clave** | **Descripción**                           |
|------------------------|---------------|-----------|-------------------------------------------|
| id_movart              | bigserial     | PK        | Identificador de la línea                 |
| id_venta               | bigint        | FK        | Venta a la que pertenece (obligatorio)    |
| id_articulo            | bigint        |           | Servicio o producto vendido (catalogo.id) |
| cantidad               | numeric(10,2) |           | Cantidad (\> 0)                           |
| precio_unitario, total | numeric(12,4) |           | Precio y total de la línea                |
| estado                 | integer       |           | Sigue el estado de la cabecera            |

#### caja_chica / caja_chica_movimiento — control de efectivo

| **Columna**               | **Tipo**  | **Clave** | **Descripción**                             |
|---------------------------|-----------|-----------|---------------------------------------------|
| caja_chica.id             | bigserial | PK        | Identificador de caja                       |
| caja_chica.id_sucursal    | bigint    | FK        | Sucursal dueña de la caja                   |
| caja_chica.saldo_actual   | numeric   |           | Saldo mantenido por trigger                 |
| caja_chica.estado         | varchar   |           | ABIERTA / CERRADA                           |
| movimiento.tipo           | varchar   |           | INGRESO o EGRESO (CHECK)                    |
| movimiento.monto          | numeric   |           | Importe, siempre \> 0 (CHECK)               |
| movimiento.id_caja_chica  | bigint    | FK        | Caja afectada                               |
| movimiento.id_metodo_pago | bigint    | FK        | Efectivo, tarjeta, Yape/Plin, transferencia |

#### datos_personales — personas y usuarios del sistema

| **Columna**            | **Tipo**  | **Clave** | **Descripción**                                      |
|------------------------|-----------|-----------|------------------------------------------------------|
| id                     | bigserial | PK        | Identificador                                        |
| nombre, apepat, apemat | varchar   |           | Nombres y apellidos                                  |
| tipodoc, numdoc        | varchar   |           | Documento; único como par (índice parcial)           |
| login, passwd          | varchar   |           | Acceso; passwd guarda hash BCrypt, nunca texto plano |
| active                 | boolean   |           | Usuario habilitado                                   |
| sucursal_actual_id     | bigint    | FK        | Sucursal activa del usuario                          |

#### roles / permissions / sessions — control de acceso

| **Columna**            | **Tipo** | **Clave** | **Descripción**                                      |
|------------------------|----------|-----------|------------------------------------------------------|
| roles.name             | varchar  | UQ        | ROLE_ADMIN, ROLE_GERENTE, ROLE_CONTADOR, ROLE_CAJERO |
| roles.parent_id        | bigint   | FK        | Jerarquía de roles                                   |
| permissions.code       | varchar  |           | Código de permiso, por ejemplo VENTAS_ANULAR         |
| sessions.user_id       | bigint   | FK        | Dueño de la sesión JWT                               |
| sessions.is_suspicious | boolean  |           | Marcada por cambio de IP en menos de 1 hora          |

#### kardex — libro de movimientos de inventario

| **Columna**                       | **Tipo** | **Clave** | **Descripción**                            |
|-----------------------------------|----------|-----------|--------------------------------------------|
| id_articulo_kardex                | varchar  | PK        | Identificador del movimiento               |
| id_catalogo, id_sucursal          | bigint   | FK        | Producto y sucursal                        |
| operacion, signo                  | varchar  |           | Tipo de operación y sentido (+ / −, CHECK) |
| cantidad, costo_unitario          | numeric  |           | Movimiento valorizado                      |
| saldo_cantidad, saldo_costo_total | numeric  |           | Saldo acumulado tras el movimiento         |

#### lab_auditoria — bitácora de auditoría

| **Columna**        | **Tipo**         | **Clave** | **Descripción**                              |
|--------------------|------------------|-----------|----------------------------------------------|
| id                 | bigserial        | PK        | Identificador                                |
| username, user_id  | varchar / bigint |           | Quién actuó                                  |
| module, action     | varchar          |           | Módulo y acción (ANULAR_VENTA, CIERRE_CAJA…) |
| old_data, new_data | text             |           | Estado anterior y posterior en JSON          |
| status, ip_address | varchar          |           | SUCCESS / WARNING y origen                   |
| timestamp          | timestamp        |           | Fecha y hora (indexada)                      |

## 2. Implementación de la base de datos

### 2.1 Entorno y scripts

| **Elemento**       | **Detalle**                                                                                                            |
|--------------------|------------------------------------------------------------------------------------------------------------------------|
| Motor              | PostgreSQL 16.15 (Ubuntu 24.04), base laboratorio_db_demo                                                              |
| Origen del esquema | Entidades JPA del backend finanzas-backend (73 entidades)                                                              |
| Datos              | Sintéticos: 2 sucursales, 5 usuarios, 4 roles, 30 ítems de catálogo, 3 000 ventas con detalle, 400 movimientos de caja |
| Ejecución          | bash 00_ejecutar_todo.sh — recrea la base y corre los scripts 01 a 05 y 07 en orden                                    |

| **Script**                               | **Contenido**                                                                                               |
|------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| 01_ddl_tablas.sql                        | CREATE TABLE de las 73 entidades (claves primarias, NOT NULL, longitudes y precisión)                       |
| 02_ddl_claves_foraneas.sql               | FK declaradas por el ORM y las 3 tablas puente (user_roles, role_permissions, usuario_sucursales_asignadas) |
| 03_integridad_indices.sql                | 8 FK lógicas, 11 CHECK/UNIQUE de negocio y 8 índices de rendimiento                                         |
| 04_dml_datos.sql                         | Datos de prueba (INSERT y generate_series)                                                                  |
| 05_funciones_procedimientos_triggers.sql | Funciones, procedimientos, triggers y vista                                                                 |
| 06_consultas.sql                         | Siete consultas de negocio                                                                                  |
| 07_seguridad_roles.sql                   | Roles, privilegios por tabla/columna y seguridad por fila                                                   |
| 08_evidencias.sh                         | Regenera todas las salidas usadas como evidencia                                                            |

### 2.2 Scripts DDL

Extracto del DDL generado (tabla de cabecera de ventas, abreviada). Los tipos salen de las anotaciones @Column: BigDecimal con precision=12 y scale=4 se traduce a numeric(12,4) y Long con IDENTITY a bigserial.

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<tbody>
<tr class="odd">
<td><p>CREATE TABLE IF NOT EXISTS "venta_registro" (</p>
<p>"id_venta" BIGSERIAL,</p>
<p>"id_sucursal" BIGINT,</p>
<p>"fecha" DATE,</p>
<p>"tipo_doc" VARCHAR(20),</p>
<p>"serie" VARCHAR(10),</p>
<p>"numdoc" VARCHAR(20),</p>
<p>"base_imp" NUMERIC(12,4),</p>
<p>"igv" NUMERIC(12,4),</p>
<p>"total" NUMERIC(12,4),</p>
<p>"descuento" NUMERIC(12,4),</p>
<p>"estado" INTEGER,</p>
<p>-- ... (otras 40 columnas contables y de auditoría)</p>
<p>PRIMARY KEY ("id_venta")</p>
<p>);</p>
<p>ALTER TABLE "venta_detalle" ADD CONSTRAINT fk_venta_detalle_id_venta</p>
<p>FOREIGN KEY ("id_venta") REFERENCES "venta_registro"("id_venta");</p></td>
</tr>
</tbody>
</table>

### 2.3 Restricciones de integridad agregadas

El script 03 traslada a la base reglas que antes solo vivían en el código Java. Así, aunque alguien escriba por psql o un reporte externo, el motor sigue protegiendo los datos.

| **Tipo** | **Restricción**                                                                                                 | **Regla de negocio**                                                       |
|----------|-----------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------|
| CHECK    | ck_venta_total_pos, ck_venta_descuento_pos                                                                      | Total y descuento nunca negativos                                          |
| CHECK    | ck_vdet_cantidad                                                                                                | Cantidad vendida mayor que cero                                            |
| CHECK    | ck_ccm_tipo, ck_ccm_monto                                                                                       | Movimiento de caja solo INGRESO/EGRESO y monto positivo                    |
| CHECK    | ck_kardex_signo                                                                                                 | Signo del kardex solo + o −                                                |
| CHECK    | ck_ad_estado, ck_ad_montos                                                                                      | Estados válidos de autorización y monto_final = monto_original − descuento |
| UNIQUE   | uq_venta_comprobante (parcial)                                                                                  | No repetir sucursal+tipo+serie+número mientras la venta no esté anulada    |
| UNIQUE   | uq_persona_documento (parcial), uq_roles_name, uq_permissions_name                                              | Sin personas, roles ni permisos duplicados                                 |
| FK       | fk_venta_sucursal, fk_caja_chica_sucursal, fk_prod\_\*, fk_kardex\_\*, fk_compras_sucursal, fk_sessions_usuario | Referencias lógicas convertidas en integridad referencial                  |

### 2.4 Evidencias de creación y restricciones

<img src="../../img_e2-plataforma-de-datos/media/30bd9b03b9d10a508d7fa1f126ea1cdb63405d8b.png" title="Figura 6. Totales de tablas, restricciones e índices de la base creada." style="width:5.83333in;height:3.75in" alt="Figura 6. Totales de tablas, restricciones e índices de la base creada." />

*Figura 6. Totales de tablas, restricciones e índices de la base creada.*

El esquema final contiene 76 tablas, 76 claves primarias, 70 claves foráneas, 22 restricciones únicas, 8 CHECK y 108 índices. La figura 7 prueba seis violaciones deliberadas: el motor rechazó todas.

<img src="../../img_e2-plataforma-de-datos/media/61ec8e6c1d8478dae613ca12c6c5e2895eb6e09d.png" title="Figura 7. Intentos de violar CHECK, FK y UNIQUE (rechazados por PostgreSQL)." style="width:6.25in;height:1.41667in" alt="Figura 7. Intentos de violar CHECK, FK y UNIQUE (rechazados por PostgreSQL)." />

*Figura 7. Intentos de violar CHECK, FK y UNIQUE (rechazados por PostgreSQL).*

### 2.5 Scripts DML y carga de datos

El script 04 inserta catálogos base (sucursales, roles, permisos, métodos de pago) y genera 3 000 ventas con sus detalles usando generate_series, de modo que los planes de ejecución sean representativos. Ejemplo:

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<tbody>
<tr class="odd">
<td><p>INSERT INTO venta_registro (id_sucursal, fecha, tipo_doc, serie, numdoc, valor_venta,</p>
<p>base_imp, igv, total, descuento, estado, id_personal_user)</p>
<p>SELECT 1+(g%2), date '2026-01-01' + (g%300),</p>
<p>CASE WHEN g%3=0 THEN 'FACTURA' ELSE 'BOLETA' END,</p>
<p>CASE WHEN g%3=0 THEN 'F001' ELSE 'B001' END, lpad(g::text,8,'0'),</p>
<p>round(((g%50)+10)::numeric,2), round(((g%50)+10)::numeric,2),</p>
<p>round(((g%50)+10)*0.18,2), round(((g%50)+10)*1.18,2),</p>
<p>CASE WHEN g%10=0 THEN 2 ELSE 0 END, CASE WHEN g%97=0 THEN 10 ELSE 1 END,</p>
<p>'cajero'||(1+g%2)</p>
<p>FROM generate_series(1,3000) g;</p></td>
</tr>
</tbody>
</table>

<img src="../../img_e2-plataforma-de-datos/media/5aa980b6110895172908b91f7037fab6af9c090c.png" title="Figura 8. Filas cargadas por tabla." style="width:3.95833in;height:2.25in" alt="Figura 8. Filas cargadas por tabla." />

*Figura 8. Filas cargadas por tabla.*

## 3. Consultas y programación en base de datos

### 3.1 Consultas SQL relevantes

| **Id** | **Pregunta de negocio**                           | **Técnica**                          |
|--------|---------------------------------------------------|--------------------------------------|
| C1     | Ventas netas, base e IGV por sucursal y mes       | Vista + agregación                   |
| C2     | Cinco servicios que más ingreso generan           | JOIN de 3 tablas + GROUP BY          |
| C3     | Tres mejores días de venta por sucursal           | Función de ventana row_number()      |
| C4     | Tasa de anulación y descuento por cajero          | FILTER sobre agregados               |
| C5     | Conciliación de caja: saldo registrado vs teórico | Función propia fn_saldo_teorico_caja |
| C6     | Productos bajo el stock mínimo                    | JOIN con filtro                      |
| C7     | Permisos efectivos por usuario                    | Cadena usuario → rol → permiso       |

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<tbody>
<tr class="odd">
<td><p>-- C3. Tres mejores días de venta por sucursal</p>
<p>SELECT * FROM (</p>
<p>SELECT id_sucursal, fecha, sum(total) AS total_dia,</p>
<p>row_number() OVER (PARTITION BY id_sucursal</p>
<p>ORDER BY sum(total) DESC, fecha) AS pos</p>
<p>FROM venta_registro WHERE estado IS DISTINCT FROM 10</p>
<p>GROUP BY id_sucursal, fecha) t</p>
<p>WHERE pos &lt;= 3 ORDER BY id_sucursal, pos;</p></td>
</tr>
</tbody>
</table>

<img src="../../img_e2-plataforma-de-datos/media/7b300bb962dd581a14092b03e4a289c432809018.png" title="Figura 9. Resultados de C1 y C2." style="width:5.83333in;height:3.19792in" alt="Figura 9. Resultados de C1 y C2." />

*Figura 9. Resultados de C1 y C2.*

<img src="../../img_e2-plataforma-de-datos/media/4aafdf56e7688adfe1171d5064f7ca1676815ade.png" title="Figura 10. Resultados de C3 a C7." style="width:5.83333in;height:7.22917in" alt="Figura 10. Resultados de C3 a C7." />

*Figura 10. Resultados de C3 a C7.*

### 3.2 Funciones y procedimientos almacenados

| **Objeto**                                | **Tipo**      | **Qué hace**                                                                                                          |
|-------------------------------------------|---------------|-----------------------------------------------------------------------------------------------------------------------|
| fn_total_ventas(sucursal, desde, hasta)   | Función       | Total de ventas vigentes de una sucursal en un rango                                                                  |
| fn_saldo_teorico_caja(caja)               | Función       | Saldo inicial + ingresos − egresos; base de la conciliación                                                           |
| sp_anular_venta(id, motivo, usuario)      | Procedimiento | Bloquea la fila, exige motivo ≥ 5 caracteres, anula cabecera y detalle y escribe la auditoría en una sola transacción |
| sp_cerrar_caja(caja, saldo_real, usuario) | Procedimiento | Cierra la caja, registra el arqueo y audita la diferencia (WARNING si no cuadra)                                      |
| vw_resumen_ventas_mensual                 | Vista         | Resumen mensual por sucursal sin ventas anuladas                                                                      |

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<tbody>
<tr class="odd">
<td><p>CREATE OR REPLACE PROCEDURE sp_anular_venta(p_id_venta BIGINT, p_motivo TEXT, p_usuario TEXT)</p>
<p>LANGUAGE plpgsql AS $$</p>
<p>DECLARE v_estado INT;</p>
<p>BEGIN</p>
<p>SELECT estado INTO v_estado FROM venta_registro WHERE id_venta = p_id_venta FOR UPDATE;</p>
<p>IF NOT FOUND THEN RAISE EXCEPTION 'Venta % no existe', p_id_venta; END IF;</p>
<p>IF v_estado = 10 THEN RAISE EXCEPTION 'Venta % ya esta anulada', p_id_venta; END IF;</p>
<p>IF p_motivo IS NULL OR length(trim(p_motivo)) &lt; 5 THEN</p>
<p>RAISE EXCEPTION 'Motivo de anulacion obligatorio (min. 5 caracteres)'; END IF;</p>
<p>UPDATE venta_registro SET estado = 10, user_upd = p_usuario WHERE id_venta = p_id_venta;</p>
<p>UPDATE venta_detalle SET estado = 10 WHERE id_venta = p_id_venta;</p>
<p>INSERT INTO lab_auditoria (username, module, action, details, status, "timestamp")</p>
<p>VALUES (p_usuario, 'VENTAS', 'ANULAR_VENTA',</p>
<p>'Venta '||p_id_venta||' anulada. Motivo: '||p_motivo, 'SUCCESS', now());</p>
<p>END $$;</p></td>
</tr>
</tbody>
</table>

### 3.3 Triggers

| **Trigger**        | **Tabla / momento**                         | **Regla que garantiza**                                                                                  |
|--------------------|---------------------------------------------|----------------------------------------------------------------------------------------------------------|
| tg_caja_movimiento | caja_chica_movimiento · BEFORE INSERT       | Actualiza saldo_actual; rechaza egresos que dejen saldo negativo y movimientos en cajas cerradas         |
| tg_venta_inmutable | venta_registro · BEFORE UPDATE              | Una venta anulada no puede cambiar de total ni de estado                                                 |
| tg_auditoria_venta | venta_registro · AFTER INSERT/UPDATE/DELETE | Registra en lab_auditoria el estado anterior y posterior (JSON) cuando cambian total, estado o descuento |

<img src="../../img_e2-plataforma-de-datos/media/794ae64132c72aea07eb1d0084e0d32f16efc6c4.png" title="Figura 11. Pruebas de triggers y procedimientos: rechazos y registros de auditoría." style="width:6.25in;height:5.28125in" alt="Figura 11. Pruebas de triggers y procedimientos: rechazos y registros de auditoría." />

*Figura 11. Pruebas de triggers y procedimientos: rechazos y registros de auditoría.*

La figura 11 muestra el flujo completo: un egreso mayor al saldo es rechazado; un ingreso sube el saldo de 700 a 750; la anulación sin motivo falla y con motivo se aplica; modificar la venta anulada es rechazado; el cierre de caja con diferencia deja una alerta WARNING; y la bitácora registra cada paso.

### 3.4 Planes de ejecución y optimización

Para medir el efecto de un índice se creó una tabla temporal de 400 000 ventas (se elimina al terminar) y se ejecutó la consulta típica de un reporte semanal por sucursal con EXPLAIN (ANALYZE, BUFFERS).

<img src="../../img_e2-plataforma-de-datos/media/e417901e1d234d587e3ead3b19216d98788ba059.png" title="Figura 12. Plan sin índice: recorrido secuencial paralelo." style="width:6.25in;height:2.01042in" alt="Figura 12. Plan sin índice: recorrido secuencial paralelo." />

*Figura 12. Plan sin índice: recorrido secuencial paralelo.*

<img src="../../img_e2-plataforma-de-datos/media/9c5166eb707babd5ba62409ce2a5ea0e24b8da5d.png" title="Figura 13. Plan con índice compuesto (id_sucursal, fecha): Bitmap Index Scan." style="width:6.25in;height:1.8125in" alt="Figura 13. Plan con índice compuesto (id_sucursal, fecha): Bitmap Index Scan." />

*Figura 13. Plan con índice compuesto (id_sucursal, fecha): Bitmap Index Scan.*

| **Métrica**                  | **Sin índice**    | **Con índice**          | **Mejora**           |
|------------------------------|-------------------|-------------------------|----------------------|
| Estrategia                   | Parallel Seq Scan | Bitmap Index Scan       | —                    |
| Tiempo de ejecución          | 15,465 ms         | 2,023 ms                | 7,6 veces más rápido |
| Bloques leídos (buffers)     | 2 560             | 597                     | 77 % menos           |
| Filas descartadas por filtro | 397 716           | 0 (el índice las evita) | —                    |

Criterio aplicado para los índices del script 03: se indexaron las columnas que aparecen en filtros de rango (fecha), en uniones frecuentes (id_venta en el detalle) y en la bitácora (timestamp, user_id). Cada índice agrega costo de escritura, por lo que no se indexó en exceso. Con solo 3 000 filas la ganancia es mínima; el beneficio crece con el volumen, y es allí donde se mide.

|                                                                                                                                                                                                                                                                                                        |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Lectura honesta de los números.** Las mediciones corresponden a un servidor de pruebas con tablas cacheadas en memoria (acierto de caché 99,76 %). En el servidor definitivo los tiempos absolutos serán otros; lo que se mantiene es la forma del plan (de recorrido completo a acceso por índice). |

## 4. Seguridad y administración

### 4.1 Usuarios y roles en el motor de base de datos

Hasta ahora el backend se conecta con un único usuario (postgres), que es superusuario. El script 07 define el diseño recomendado de mínimo privilegio antes del despliegue:

| **Rol / usuario**    | **Hereda de** | **Privilegios**                                                                                                                              | **Límite de conexiones** |
|----------------------|---------------|----------------------------------------------------------------------------------------------------------------------------------------------|--------------------------|
| app_finanzas (login) | rol_app       | SELECT, INSERT, UPDATE, DELETE y EXECUTE; sin DDL; auditoría de solo anexar (sin UPDATE/DELETE en lab_auditoria)                             | 20                       |
| analista_bi (login)  | rol_lectura   | SELECT en tablas; sin sessions ni intrusion_attempts; en datos_personales solo columnas no sensibles (sin passwd); filtro por sucursal (RLS) | 5                        |
| auditor_ext (login)  | rol_auditor   | SELECT en lab_auditoria, intrusion_attempts y columnas no sensibles de sessions (sin token)                                                  | 2                        |

<img src="../../img_e2-plataforma-de-datos/media/752b602e2fd5c7df8b847de21fd7ab64a7b16fbb.png" title="Figura 14. Roles creados en el motor y sus herencias." style="width:5in;height:2.35417in" alt="Figura 14. Roles creados en el motor y sus herencias." />

*Figura 14. Roles creados en el motor y sus herencias.*

<img src="../../img_e2-plataforma-de-datos/media/98ccec841352207171558a79685b68f0182cf05a.png" title="Figura 15. Pruebas de privilegios por rol (permitido y denegado)." style="width:5.625in;height:9.02083in" alt="Figura 15. Pruebas de privilegios por rol (permitido y denegado)." />

*Figura 15. Pruebas de privilegios por rol (permitido y denegado).*

Resultados de la prueba: la aplicación no puede borrar tablas (DROP) ni la bitácora; el analista no puede leer contraseñas ni tokens ni insertar; el auditor ve la bitácora pero no las ventas. La seguridad por fila (RLS) hace que el analista vea 0 ventas si no declara sucursal y 1 500 (solo la sucursal 1) cuando declara app.sucursal_id = 1.

### 4.2 Seguridad a nivel de aplicación

| **Control**     | **Implementación en el backend**                                                                                                        |
|-----------------|-----------------------------------------------------------------------------------------------------------------------------------------|
| Autenticación   | JWT firmado con RS256; expiración 30 minutos (jwt.expiration=1800000)                                                                   |
| Contraseñas     | Hash BCrypt; nunca se almacenan en texto plano                                                                                          |
| Bloqueo         | 5 intentos fallidos bloquean 15 minutos (tabla intrusion_attempts)                                                                      |
| Sesiones        | Máximo 3 concurrentes por usuario; se marca como sospechosa una sesión con cambio de IP en menos de 1 hora                              |
| Autorización    | RBAC con roles jerárquicos, permisos por módulo, segregación de funciones y delegación temporal                                         |
| Flujos críticos | Descuentos, notas de crédito y anulaciones requieren autorización de un segundo usuario (tablas autorizacion\_\* y solicitud_anulacion) |

### 4.3 Auditoría implementada

El sistema mantiene dos mecanismos complementarios: (1) auditoría de aplicación, que registra acciones de negocio (usuario, IP, módulo, antes y después) en audit_log y lab_auditoria; y (2) auditoría en la base, el trigger tg_auditoria_venta, que captura cualquier cambio sobre ventas aun cuando se haga fuera de la aplicación. La figura 11 muestra cuatro registros reales generados durante las pruebas. Consulta recomendada para auditores:

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<tbody>
<tr class="odd">
<td><p>SELECT "timestamp", username, module, action, status, details</p>
<p>FROM lab_auditoria</p>
<p>WHERE status &lt;&gt; 'SUCCESS' OR action IN ('ANULAR_VENTA','CIERRE_CAJA')</p>
<p>ORDER BY "timestamp" DESC LIMIT 50;</p></td>
</tr>
</tbody>
</table>

### 4.4 Respaldo y recuperación

| **Aspecto**            | **Definición**                                                                              |
|------------------------|---------------------------------------------------------------------------------------------|
| Método                 | Respaldo lógico con pg_dump -Fc (formato comprimido, restauración selectiva con pg_restore) |
| Frecuencia propuesta   | Diario a las 02:00 (cron); antes de cada despliegue; semanal copia fuera del servidor       |
| Retención propuesta    | 7 diarios y 4 semanales; rotación con find -mtime                                           |
| Objetivo               | RPO: 24 horas · RTO: menos de 1 hora (a validar con el volumen real)                        |
| Prueba de restauración | Se restaura en una base nueva y se compara tablas, filas y suma de ventas contra el origen  |

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<tbody>
<tr class="odd">
<td><p># Respaldo diario (cron: 0 2 * * *)</p>
<p>pg_dump -h localhost -U finanzas -d laboratorio_db -Fc -f /backups/laboratorio_$(date +%F).dump</p>
<p>find /backups -name 'laboratorio_*.dump' -mtime +7 -delete</p>
<p># Restauración de verificación</p>
<p>createdb laboratorio_db_restore</p>
<p>pg_restore -d laboratorio_db_restore --no-owner /backups/laboratorio_AAAA-MM-DD.dump</p></td>
</tr>
</tbody>
</table>

<img src="../../img_e2-plataforma-de-datos/media/b20cb64a1f3d51fc72d3836bac90824410438e5b.png" title="Figura 16. Respaldo, restauración y verificación (origen = restaurada)." style="width:5.41667in;height:4.32292in" alt="Figura 16. Respaldo, restauración y verificación (origen = restaurada)." />

*Figura 16. Respaldo, restauración y verificación (origen = restaurada).*

La verificación comparó 76 tablas, 3 000 ventas con suma de 122 130,00 y 401 movimientos de caja: los valores son idénticos en origen y copia restaurada. El respaldo de 388 KB tardó 0,13 s y la restauración 0,37 s en este volumen de prueba.

### 4.5 Monitoreo

Se usan las vistas estadísticas nativas de PostgreSQL. La figura 17 muestra tamaño de la base, tablas más grandes, relación entre recorridos secuenciales e indexados, tasa de aciertos de caché y sesiones activas.

<img src="../../img_e2-plataforma-de-datos/media/5e3b8a474da8d610a6b7cef953d76efd366d6076.png" title="Figura 17. Indicadores de monitoreo (pg_stat_database, pg_stat_user_tables, pg_stat_activity)." style="width:5.41667in;height:4.15625in" alt="Figura 17. Indicadores de monitoreo (pg_stat_database, pg_stat_user_tables, pg_stat_activity)." />

*Figura 17. Indicadores de monitoreo (pg_stat_database, pg_stat_user_tables, pg_stat_activity).*

| **Indicador**                             | **Umbral de atención**                           | **Fuente**                     |
|-------------------------------------------|--------------------------------------------------|--------------------------------|
| Acierto de caché                          | por debajo de 95 %                               | pg_stat_database               |
| Recorridos secuenciales en tablas grandes | seq_scan creciendo mientras idx_scan se mantiene | pg_stat_user_tables            |
| Consultas lentas                          | mayores a 500 ms                                 | pg_stat_statements (extensión) |
| Conexiones activas                        | cercanas al límite del rol (20)                  | pg_stat_activity               |
| Espacio en disco y último respaldo        | menos del 20 % libre; respaldo con más de 26 h   | Monitoreo del servidor / cron  |

### 4.6 Hallazgos y riesgos detectados

|                                                                                  |
|----------------------------------------------------------------------------------|
| **Hallazgos de la revisión.** Se registran para corregirlos antes de producción: |

- **Credenciales en el repositorio:** el archivo finanzas-backend/CheckDb.java contiene usuario y contraseña de una base local escritos en el código. Debe eliminarse del repositorio y la contraseña cambiarse.

- **Esquema sin migraciones completas:** solo hay 23 migraciones Flyway (desactivadas) y el esquema real lo crea Hibernate. Recomendación: usar el script 01/02 o un pg_dump -s de producción como migración base V1 y activar Flyway.

- **Conexión con superusuario:** el backend usa el usuario postgres. En producción debe usar app_finanzas (script 07).

- **Autenticación del motor:** la instancia de pruebas usa confianza local (trust) para facilitar la evidencia; en el servidor deben usarse scram-sha-256 en pg_hba.conf y la base no debe exponerse a internet (en el despliegue con Docker Compose solo la ve el contenedor del backend).

- **FK pendientes:** las referencias lógicas de la sección 1.4 deben verificarse contra huérfanos y convertirse en FK.

- **Archivo database.db vacío:** residuo de SQLite sin uso; puede eliminarse.

## Anexos

### Anexo A. Scripts completos

Se entregan junto con este documento en la carpeta de scripts:

- 00_ejecutar_todo.sh, 01_ddl_tablas.sql, 02_ddl_claves_foraneas.sql, 03_integridad_indices.sql

- 04_dml_datos.sql, 05_funciones_procedimientos_triggers.sql, 06_consultas.sql, 07_seguridad_roles.sql, 08_evidencias.sh

- diccionario_datos_completo.csv (1 457 columnas), esquema completo en laboratorio_db_demo_esquema.sql (pg_dump -s)

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<tbody>
<tr class="odd">
<td><p># Reproducir todo en su propio PostgreSQL</p>
<p>export PGHOST=localhost PGUSER=postgres</p>
<p>bash 00_ejecutar_todo.sh # crea laboratorio_db_demo y aplica los scripts</p>
<p>psql -d laboratorio_db_demo -f 06_consultas.sql</p>
<p>bash 08_evidencias.sh # regenera las salidas en ./evidencias</p></td>
</tr>
</tbody>
</table>

### Anexo B. Capturas de ejecución

Las figuras 6 a 17 son salidas reales de psql ejecutadas en la base de pruebas, renderizadas en formato de terminal. Para la sustentación se recomienda repetirlas en pgAdmin o psql sobre el equipo del estudiante con los scripts del Anexo A.

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<tbody>
<tr class="odd">
<td><p><strong>[ Captura propia en pgAdmin / psql ]</strong></p>
<p><em>Insertar aquí la captura de la ejecución en el equipo del estudiante (fecha y hora visibles)</em></p></td>
</tr>
</tbody>
</table>

### Anexo C. Resultados de consultas

Los resultados de C1 a C7 están en las figuras 9 y 10; las pruebas de reglas en las figuras 7 y 11.

### Anexo D. Documentación técnica de administración

| **Tarea**                    | **Comando / acción**                                                                                                |
|------------------------------|---------------------------------------------------------------------------------------------------------------------|
| Crear la base desde cero     | bash 00_ejecutar_todo.sh                                                                                            |
| Cambiar contraseña de un rol | ALTER ROLE app_finanzas PASSWORD '…';                                                                               |
| Respaldo manual              | pg_dump -Fc -d laboratorio_db -f respaldo.dump                                                                      |
| Restaurar                    | createdb nueva_bd && pg_restore -d nueva_bd respaldo.dump                                                           |
| Actualizar estadísticas      | ANALYZE; (diario por autovacuum; manual tras cargas masivas)                                                        |
| Ver consultas lentas         | CREATE EXTENSION pg_stat_statements; SELECT query, mean_exec_time FROM pg_stat_statements ORDER BY 2 DESC LIMIT 10; |
| Variables del backend        | DB_URL, DB_USER, DB_PASSWORD (deploy/.env)                                                                          |

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
