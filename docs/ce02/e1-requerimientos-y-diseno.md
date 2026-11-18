# CE021 - S1 - E1: Requerimientos y Diseño del Sistema

**Equipo:** Omar Condori Pachauri (201810211) · Merry Elizabeth Flores Llanque (202211953) · Jaqueline Huahuaccapa Ccama (202210712) — Ciclo 10.

## Resumen ejecutivo

**Problema.** Reumacenter E.I.R.L., laboratorio clínico de Juliaca, necesitaba integrar en un solo sistema los procesos administrativos y financieros que se llevaban por separado: atención y registro de pacientes, ventas de servicios, compras e inventario, caja, cuentas por cobrar y pagar, honorarios médicos, seguridad y reportes. Esa dispersión dificultaba controlar importes, autorizaciones y saldos.

**Usuarios.** Cajeros y personal de admisión, logística y compras, finanzas y recursos humanos, aprobadores (jefaturas) y administrador del sistema.

**Alcance.** ERP administrativo-financiero web con 47 requisitos funcionales y 21 no funcionales agrupados en diez dominios. Quedan fuera la historia clínica electrónica, la facturación electrónica con SUNAT y la conexión bancaria.

**Solución propuesta.** Aplicación web de tres capas: una SPA en Angular 20, una API REST con WebSocket en Spring Boot (Java 21) con seguridad JWT RS256 y reportes JasperReports, y una base de datos relacional PostgreSQL (laboratorio_db). La arquitectura objetivo se despliega con el frontend en Vercel y el backend, la base de datos y un proxy HTTPS en un servidor Linux con Docker Compose.

**Contenido del documento.** La Sección 1 especifica los requerimientos; la Sección 2 presenta el prototipo navegable (el propio sistema en alta fidelidad); la Sección 3 documenta la arquitectura con vistas C4 y decisiones (ADR); la Sección 4 detalla el diseño con diagramas UML. Los anexos incluyen el acta de validación, la matriz de trazabilidad y la evidencia de sustentación.

|                                                                                                                                                                                                                                                                                                                                |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Aclaración sobre el origen de los requisitos.** Los requerimientos fueron comunicados verbalmente por el ingeniero responsable y validados en reuniones virtuales con los encargados de la empresa. Este documento los organiza de forma retrospectiva; no es una especificación contractual firmada al inicio del proyecto. |

## Sección 1: Especificación de requerimientos

### 1.1 Contexto, alcance y partes interesadas

La entidad es Reumacenter E.I.R.L., con sede en Jr. Ladislao Butrón N.° 177, Juliaca. El sistema, identificado como FINANZAS, centraliza la información administrativa y financiera del laboratorio en una aplicación web y una base de datos relacional compartida. La denominación ERP expresa esa integración de procesos; no implica cubrir todos los procesos clínicos, tributarios, contables o laborales de la empresa.

| **Parte interesada**           | **Interés / expectativa**                                                   | **Influencia**    |
|--------------------------------|-----------------------------------------------------------------------------|-------------------|
| Gerencia / dirección           | Control de ventas, caja y resultados; decisiones con información confiable. | Alta              |
| Ingeniero responsable (TI)     | Entrega funcional del software, código y base de datos en el repositorio.   | Alta              |
| Cajeros y admisión             | Registrar pacientes y ventas rápido, sin errores de importes.               | Alta (uso diario) |
| Logística y compras            | Control de proveedores, compras y existencias.                              | Media             |
| Finanzas y RR. HH.             | Cuentas, honorarios, pagos y reportes exportables.                          | Alta              |
| Aprobadores                    | Resolver descuentos, notas de crédito y anulaciones con trazabilidad.       | Media             |
| Administrador del sistema      | Gestión de usuarios, roles, permisos y auditoría.                           | Alta              |
| Pacientes y empresas atendidas | Comprobantes correctos y protección de sus datos personales.                | Baja (indirecta)  |

### 1.2 Requerimientos funcionales

Los requisitos se agrupan por dominio. La prioridad (Alta, Media, Baja) es una propuesta basada en la dependencia del proceso: los maestros y las ventas habilitan al resto de módulos.

| **ID**                                                  | **Requisito**                                 | **Descripción**                                                                                                                                                          | **Prioridad** |
|---------------------------------------------------------|-----------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------|
| **ADMISIÓN Y PACIENTES**                                |                                               |                                                                                                                                                                          |               |
| **RF-01**                                               | Gestión de personas                           | Registrar, editar, buscar y consultar los datos de identificación de una persona (tabla datos_personales), validando documento y datos obligatorios.                     | Alta          |
| **RF-02**                                               | Pacientes                                     | Registrar al paciente a partir de una persona (datos_paciente), con su tipo de paciente, contacto y la entidad responsable cuando exista.                                | Alta          |
| **RF-03**                                               | Grupos familiares                             | Asociar pacientes en un grupo familiar con titular y dependientes, para aplicar beneficios configurados.                                                                 | Media         |
| **RF-04**                                               | Empresas y compañías                          | Registrar empresas atendidas y compañías (aseguradoras), diferenciando su rol comercial.                                                                                 | Alta          |
| **RF-05**                                               | Tipos de paciente y de documento              | Mantener los catálogos de tipo de paciente, tipo de atención y tipo de documento usados en las operaciones.                                                              | Media         |
| **COMERCIAL Y FACTURACIÓN**                             |                                               |                                                                                                                                                                          |               |
| **RF-06**                                               | Venta de servicios                            | Registrar una venta con paciente, convenio, servicios, cantidades, precios, descuentos, IGV, medio de pago, punto, serie y correlativo (venta_registro / venta_detalle). | Alta          |
| **RF-07**                                               | Paquetes de servicios                         | Definir paquetes de servicios con sus condiciones y venderlos como un solo ítem.                                                                                         | Media         |
| **RF-08**                                               | Numeración de documentos                      | Asignar serie y correlativo por punto de atención y tipo de documento, evitando duplicados.                                                                              | Alta          |
| **RF-09**                                               | Precios, convenios y beneficios               | Calcular el importe aplicando tarifa, convenio, cobertura y beneficio configurados (motor de precios).                                                                   | Alta          |
| **RF-10**                                               | Consulta y reimpresión                        | Buscar comprobantes registrados y reimprimirlos sin crear una nueva operación.                                                                                           | Media         |
| **RF-11**                                               | Solicitud de anulación                        | Registrar la solicitud de anulación de un comprobante con motivo, solicitante y estado (PENDIENTE, APROBADO, RECHAZADO).                                                 | Alta          |
| **DESCUENTOS, NOTAS DE CRÉDITO Y AUTORIZACIONES**       |                                               |                                                                                                                                                                          |               |
| **RF-12**                                               | Descuentos                                    | Aplicar descuentos por monto o porcentaje respetando los límites configurados.                                                                                           | Alta          |
| **RF-13**                                               | Autorización de descuento                     | Solicitar y resolver un descuento (APROBADA / RECHAZADA); una autorización aprobada se consume en la venta (estado USADO).                                               | Alta          |
| **RF-14**                                               | Notas de crédito                              | Emitir una nota de crédito interna vinculada al comprobante de origen, con su importe y estado (EMITIDA / ANULADA).                                                      | Alta          |
| **RF-15**                                               | Autorización de notas y anulaciones           | Separar solicitante y aprobador, conservar la decisión y relacionarla con la operación afectada.                                                                         | Alta          |
| **TESORERÍA, CAJAS Y CUENTAS CORRIENTES**               |                                               |                                                                                                                                                                          |               |
| **RF-16**                                               | Caja general                                  | Registrar movimientos, consultar saldos por medio de pago y cerrar la caja general.                                                                                      | Alta          |
| **RF-17**                                               | Caja chica y arqueo                           | Abrir caja, registrar ingresos y egresos, realizar el arqueo y relacionar el cierre con la caja general.                                                                 | Alta          |
| **RF-18**                                               | Cuentas por cobrar                            | Gestionar créditos y planes (cred_ctacte / cred_planes): origen, abonos, saldo y vencimiento.                                                                            | Alta          |
| **RF-19**                                               | Convenios y coberturas                        | Configurar condiciones comerciales y de crédito, diferenciando la obligación del paciente y la del tercero.                                                              | Media         |
| **RF-20**                                               | Cuentas por pagar                             | Registrar obligaciones con proveedores, su cronograma de pagos y los pagos efectuados.                                                                                   | Alta          |
| **RF-21**                                               | Cuentas corrientes financieras                | Registrar bancos, cuentas, moneda y titular. No incluye conexión bancaria automática.                                                                                    | Media         |
| **LOGÍSTICA, COMPRAS E INVENTARIO**                     |                                               |                                                                                                                                                                          |               |
| **RF-22**                                               | Catálogo y niveles                            | Mantener el maestro de servicios y productos con código, unidad, precio, tipo, estado y clasificación por niveles.                                                       | Alta          |
| **RF-23**                                               | Productos y stock                             | Registrar productos con presentación, unidad, factor de conversión y existencias.                                                                                        | Alta          |
| **RF-24**                                               | Proveedores                                   | Registrar proveedores con identificación, datos comerciales, condiciones y estado.                                                                                       | Alta          |
| **RF-25**                                               | Órdenes de compra                             | Crear y aprobar órdenes de compra (BORRADOR → APROBADO → ENVIADO → RECIBIDO / ANULADO).                                                                                  | Alta          |
| **RF-26**                                               | Compras y recepción                           | Registrar la compra y su recepción, generando el movimiento de inventario y la obligación por pagar.                                                                     | Alta          |
| **RF-27**                                               | Ajustes, anulaciones y devoluciones de compra | Corregir o anular una compra conservando la referencia al registro de origen y el motivo.                                                                                | Media         |
| **RF-28**                                               | Kardex y valoración                           | Consultar entradas, salidas, transferencias y ajustes con cantidades, costos y saldos.                                                                                   | Alta          |
| **RECURSOS HUMANOS Y HONORARIOS**                       |                                               |                                                                                                                                                                          |               |
| **RF-29**                                               | Empleados y profesionales                     | Registrar personal y profesionales con rol, especialidad, vínculo y estado.                                                                                              | Alta          |
| **RF-30**                                               | Valorización médica                           | Calcular la valorización de prestaciones por profesional y período según la regla vigente.                                                                               | Alta          |
| **RF-31**                                               | Pagos de personal                             | Registrar pagos con período, conceptos, importes, sustento y estado.                                                                                                     | Media         |
| **RF-32**                                               | Honorarios médicos                            | Liquidar y pagar honorarios a partir de la valorización, distinguiendo liquidación interna de recibo tributario.                                                         | Alta          |
| **FINANZAS CORPORATIVAS Y CONFIGURACIÓN**               |                                               |                                                                                                                                                                          |               |
| **RF-33**                                               | Plan de cuentas PMT                           | Clasificar operaciones con el plan de cuentas administrativo.                                                                                                            | Baja          |
| **RF-34**                                               | Activos fijos                                 | Registrar activos con ubicación, responsable, costo, estado y depreciación.                                                                                              | Baja          |
| **RF-35**                                               | Compañías, empresas y sucursales              | Configurar la estructura organizacional y sus relaciones de ámbito.                                                                                                      | Alta          |
| **RF-36**                                               | Puntos de atención                            | Registrar puntos y asociarlos a sus documentos y series.                                                                                                                 | Alta          |
| **SEGURIDAD, ACCESOS Y AUDITORÍA**                      |                                               |                                                                                                                                                                          |               |
| **RF-37**                                               | Autenticación y sesión                        | Autenticar al usuario con JWT, controlar la vigencia de la sesión y el bloqueo por intentos fallidos.                                                                    | Alta          |
| **RF-38**                                               | Autorización granular                         | Controlar acceso por roles, permisos y opciones de menú, validado en la interfaz y en la API.                                                                            | Alta          |
| **RF-39**                                               | Auditoría dual                                | Registrar eventos técnicos (audit_log) y eventos de negocio (lab_auditoria) y consultarlos en una bitácora filtrable.                                                    | Alta          |
| **RF-40**                                               | Segregación y delegación                      | Definir funciones incompatibles y delegaciones temporales de permisos.                                                                                                   | Media         |
| **RF-41**                                               | Claves y contexto de sucursal                 | Cambiar o restablecer contraseña y seleccionar la sucursal o punto de trabajo autorizado.                                                                                | Alta          |
| **REPORTES, INTELIGENCIA DE NEGOCIOS Y NOTIFICACIONES** |                                               |                                                                                                                                                                          |               |
| **RF-42**                                               | Comprobantes PDF                              | Generar tickets de 58 y 80 mm y facturas A4 y A5 con JasperReports.                                                                                                      | Alta          |
| **RF-43**                                               | Listados operativos                           | Consultar ventas, compras, descuentos, comprobantes anulados, honorarios, pagos, valorización y correlatividad.                                                          | Alta          |
| **RF-44**                                               | Exportación a Excel                           | Exportar el listado visible con los criterios de filtro aplicados.                                                                                                       | Media         |
| **RF-45**                                               | Dashboard financiero                          | Mostrar indicadores de ventas, egresos, valor de inventario y utilidad por período (hoy, mes, año).                                                                      | Media         |
| **RF-46**                                               | Notificaciones en tiempo real                 | Comunicar eventos del sistema al usuario mediante WebSocket.                                                                                                             | Media         |
| **INTERFAZ**                                            |                                               |                                                                                                                                                                          |               |
| **RF-47**                                               | Interfaz web responsive                       | Adaptar menú, tablas y formularios al ancho de pantalla (escritorio y móvil) con estados de carga, vacío, error y confirmación.                                          | Alta          |

### 1.3 Requerimientos no funcionales

Se clasifican según las características de calidad de ISO/IEC 25010. Cuando el valor proviene de la configuración del proyecto se indica; las metas de rendimiento y disponibilidad son objetivos de diseño que se medirán en el Entregable 4, no resultados ya comprobados.

| **ID**                                          | **Característica** | **Requisito**                                                                                                    | **Criterio / verificación**                            |
|-------------------------------------------------|--------------------|------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------|
| **SEGURIDAD**                                   |                    |                                                                                                                  |                                                        |
| **RNF-01**                                      | Seguridad          | Autenticación con JWT firmado con RSA (RS256); las llaves se leen desde archivo externo y no se versionan.       | Token inválido o vencido → HTTP 401                    |
| **RNF-02**                                      | Seguridad          | Contraseñas almacenadas con hash BCrypt; nunca en texto plano ni en logs.                                        | Inspección de la tabla de usuarios                     |
| **RNF-03**                                      | Seguridad          | La autorización de cada operación se valida en el backend; los guards del frontend solo orientan la navegación.  | Llamada directa a la API sin permiso → HTTP 403        |
| **RNF-04**                                      | Seguridad          | Bloqueo temporal tras 5 intentos fallidos durante 15 minutos (configurable).                                     | security.max-failed-attempts=5; block-duration=15      |
| **RNF-05**                                      | Seguridad          | Vigencia de sesión de 30 minutos y máximo 3 sesiones concurrentes por usuario (configurables).                   | jwt.expiration=1 800 000 ms; max-concurrent-sessions=3 |
| **RNF-06**                                      | Seguridad          | Toda comunicación en producción viaja por HTTPS/WSS; CORS restringido a los orígenes autorizados.                | Certificado válido; origen no autorizado rechazado     |
| **RNF-07**                                      | Seguridad          | Secretos (BD, llaves, Twilio, SMTP) provistos por variables de entorno, no por el repositorio.                   | Revisión del repositorio y del archivo .env            |
| **RENDIMIENTO Y ESCALABILIDAD**                 |                    |                                                                                                                  |                                                        |
| **RNF-08**                                      | Rendimiento        | Los listados se consultan con paginación y filtros en el servidor.                                               | Parámetros page/size en cada listado                   |
| **RNF-09**                                      | Rendimiento        | Meta de diseño: consultas paginadas y registro de una venta en ≤ 2 s con hasta 10 usuarios concurrentes.         | Por verificar con pruebas de carga (Entregable 4)      |
| **RNF-10**                                      | Escalabilidad      | API sin estado de sesión en el servidor (JWT), lo que permite replicarla detrás de un proxy.                     | Revisión de arquitectura                               |
| **DISPONIBILIDAD Y CONFIABILIDAD**              |                    |                                                                                                                  |                                                        |
| **RNF-11**                                      | Integridad         | Operaciones que afectan varias tablas se ejecutan en una transacción única (todo o nada).                        | Prueba de reversión ante error                         |
| **RNF-12**                                      | Integridad         | Claves primarias, foráneas y restricciones de unicidad (serie + número) en el esquema relacional.                | Intento de duplicar un comprobante → rechazo           |
| **RNF-13**                                      | Disponibilidad     | Meta de operación: servicio disponible en horario de atención con reinicio automático del contenedor ante caída. | Política restart: always; por medir                    |
| **RNF-14**                                      | Recuperación       | Copia de seguridad de la base de datos con pg_dump y de los archivos subidos, con procedimiento de restauración. | Por documentar y ensayar (Entregable 2)                |
| **RNF-15**                                      | Observabilidad     | Logs de aplicación separados de la bitácora de negocio y endpoint de salud (actuator).                           | /actuator/health responde UP                           |
| **USABILIDAD, COMPATIBILIDAD Y MANTENIBILIDAD** |                    |                                                                                                                  |                                                        |
| **RNF-16**                                      | Usabilidad         | Diseño adaptativo para anchos desde 360 px hasta pantallas de escritorio; mensajes de validación claros.         | Revisión en móvil y escritorio                         |
| **RNF-17**                                      | Compatibilidad     | Funcionamiento en versiones recientes de Chrome, Edge y Firefox.                                                 | Prueba manual en cada navegador                        |
| **RNF-18**                                      | Mantenibilidad     | Separación en controlador → servicio → repositorio, con DTO para el contrato de la API.                          | Estructura de paquetes                                 |
| **RNF-19**                                      | Mantenibilidad     | Código fuente y base de datos versionados en Git.                                                                | Historial del repositorio                              |
| **RNF-20**                                      | Portabilidad       | Backend empaquetado como imagen Docker y desplegable con Docker Compose en cualquier servidor Linux.             | docker compose up -d levanta el sistema                |
| **RNF-21**                                      | Costo              | Despliegue de bajo costo (capa gratuita o servidor económico) para la fase de producción piloto.                 | Costo mensual del servicio                             |

### 1.4 Reglas de negocio

| **ID** | **Regla**                                                                                                                                           |
|--------|-----------------------------------------------------------------------------------------------------------------------------------------------------|
| RN-01  | La combinación serie + número de documento es única: no se registran dos comprobantes con el mismo correlativo.                                     |
| RN-02  | Una orden de atención solo puede tener una venta registrada.                                                                                        |
| RN-03  | El importe de una venta se calcula en este orden: valor de venta, descuento, cobertura, base imponible, IGV y total.                                |
| RN-04  | Un descuento especial requiere una autorización aprobada; al usarse en una venta pasa a estado USADO y no puede reutilizarse.                       |
| RN-05  | El solicitante y el aprobador de un descuento, nota de crédito o anulación deben ser personas distintas (segregación de funciones).                 |
| RN-06  | No se puede anular un comprobante que ya tiene una nota de crédito EMITIDA ni uno que ya está anulado.                                              |
| RN-07  | Cobrar una orden o devolver dinero exige que el usuario tenga una caja chica abierta.                                                               |
| RN-08  | Una venta registra la prestación; el cobro registra la recepción de dinero; la cuenta por cobrar conserva la deuda pendiente. Son hechos distintos. |
| RN-09  | Una orden de compra expresa lo solicitado; la recepción, lo realmente recibido; el kardex, el efecto sobre las existencias.                         |
| RN-10  | La recepción de una compra genera movimientos de kardex y una obligación en cuentas por pagar.                                                      |
| RN-11  | Los honorarios se liquidan a partir de prestaciones valorizadas del profesional en el período.                                                      |
| RN-12  | Toda operación se asocia a un usuario, sucursal y punto autorizados; el usuario solo ve y opera dentro de su ámbito.                                |
| RN-13  | Los datos no se eliminan físicamente cuando existe historial: se usa borrado lógico o cambio de estado.                                             |
| RN-14  | Los documentos generados son internos; no equivalen a comprobantes electrónicos aceptados por SUNAT.                                                |

### 1.5 Restricciones del sistema

| **Tipo**          | **Restricción**                                                                                                                                               |
|-------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Tecnológicas**  | Frontend Angular 20 + TypeScript + Tailwind CSS; backend Java 21 con Spring Boot 4.0.1; base de datos relacional PostgreSQL; reportes con JasperReports 6.21. |
| **De alcance**    | Los datos de pacientes son administrativos: el sistema no implementa historia clínica electrónica ni resultados biomédicos.                                   |
| **Tributarias**   | No existe integración con SUNAT (sin XML, firma digital ni CDR). Las notas de crédito y anulaciones son flujos internos.                                      |
| **Financieras**   | No hay conexión bancaria ni pasarelas de pago: las cuentas y movimientos se registran manualmente.                                                            |
| **De proceso**    | Los requerimientos fueron comunicados verbalmente por el ingeniero responsable; este documento los formaliza retrospectivamente.                              |
| **De despliegue** | Producción piloto de una semana con infraestructura gratuita o de muy bajo costo: frontend en Vercel y backend con base de datos en un único servidor Linux.  |
| **Normativas**    | Debe considerarse la Ley N.° 29733 de Protección de Datos Personales (Perú) por el tratamiento de datos de pacientes y personal.                              |
| **Operativas**    | Idioma de la interfaz: español. Moneda: soles (S/). Zona horaria: America/Lima.                                                                               |

### 1.6 Casos de uso e historias de usuario

El diagrama muestra los actores y los casos de uso principales. Cada caso de uso se desarrolla en historias de usuario con el formato «Como \[rol\], quiero \[acción\] para \[beneficio\]».

<img src="../../img_e1-requerimientos-y-diseno/media/12a89052d425eed82eb6ae1e4248de090c4dcce1.png" title="Figura 1.1 — Diagrama de casos de uso del sistema FINANZAS" style="width:5.83333in;height:4.9375in" alt="Figura 1.1 — Diagrama de casos de uso del sistema FINANZAS" />

*Figura 1.1 — Diagrama de casos de uso del sistema FINANZAS*

| **ID** | **Como…**     | **Quiero…**                                        | **Para…**                                           | **Requisitos**             |
|--------|---------------|----------------------------------------------------|-----------------------------------------------------|----------------------------|
| HU-01  | Cajero        | registrar una venta de servicios a un paciente     | cobrar y emitir el comprobante en pocos pasos       | RF-06, RF-08, RF-09, RF-42 |
| HU-02  | Cajero        | solicitar un descuento especial                    | atender casos excepcionales con control             | RF-12, RF-13, RF-15        |
| HU-03  | Aprobador     | aprobar o rechazar una solicitud pendiente         | dejar constancia de la decisión                     | RF-13, RF-15, RF-46        |
| HU-04  | Cajero        | anular un comprobante o emitir una nota de crédito | corregir errores sin perder trazabilidad            | RF-11, RF-14               |
| HU-05  | Cajero        | abrir, arquear y cerrar mi caja chica              | cuadrar el dinero cobrado del día                   | RF-17, RF-16               |
| HU-06  | Logística     | registrar una orden de compra y su recepción       | mantener actualizado el inventario y lo que se debe | RF-25, RF-26, RF-20        |
| HU-07  | Logística     | consultar el kardex de un producto                 | conocer entradas, salidas y saldo                   | RF-28                      |
| HU-08  | Finanzas      | liquidar honorarios de un médico por período       | pagar según lo valorizado                           | RF-30, RF-32               |
| HU-09  | Finanzas      | ver cuentas por cobrar y por pagar                 | priorizar cobros y pagos                            | RF-18, RF-20               |
| HU-10  | Administrador | crear usuarios y asignar roles y permisos          | cada persona acceda solo a lo que le corresponde    | RF-37, RF-38, RF-41        |
| HU-11  | Administrador | consultar la bitácora de auditoría                 | investigar quién hizo qué y cuándo                  | RF-39                      |
| HU-12  | Finanzas      | emitir reportes y exportarlos a Excel              | analizar ventas, compras y honorarios               | RF-43, RF-44, RF-45        |

### 1.7 Criterios de aceptación

Los criterios se redactan en formato Dado / Cuando / Entonces para que sean verificables en las pruebas de aceptación.

#### HU-01

**Escenario 1.** Dado que el cajero tiene una caja abierta y un paciente seleccionado, cuando agrega servicios, elige el medio de pago y confirma la venta, entonces el sistema registra la venta con su serie y correlativo, calcula IGV y total y permite imprimir el comprobante.

**Escenario 2.** Dado que ya existe un comprobante con la misma serie y número, cuando se intenta registrar otro igual, entonces el sistema rechaza la operación con un mensaje claro y no guarda datos.

**Escenario 3.** Dado que el cajero no tiene caja abierta, cuando intenta cobrar una orden, entonces el sistema le indica que debe abrir caja antes de continuar.

#### HU-02 / HU-03

**Escenario 1.** Dado un descuento especial solicitado, cuando el aprobador lo aprueba, entonces la autorización queda APROBADA y el solicitante recibe una notificación.

**Escenario 2.** Dado una autorización aprobada, cuando se usa en una venta, entonces pasa a USADO y no puede aplicarse a otra venta.

**Escenario 3.** Dado que el solicitante y el aprobador son la misma persona, cuando intenta aprobar, entonces el sistema lo impide.

#### HU-04

**Escenario 1.** Dado un comprobante con nota de crédito EMITIDA, cuando se solicita su anulación, entonces el sistema la rechaza e indica que debe usarse la nota de crédito.

**Escenario 2.** Dado un comprobante vigente y caja abierta, cuando la anulación es aprobada, entonces el comprobante queda anulado, se registra la devolución en caja y se guarda el motivo.

#### HU-06

**Escenario 1.** Dada una orden de compra enviada, cuando se registra su recepción, entonces se generan los movimientos de kardex y la cuenta por pagar correspondiente.

#### HU-10

**Escenario 1.** Dado un usuario sin el permiso requerido, cuando invoca una operación restringida, entonces la API responde 403 aunque la opción esté oculta o visible en la interfaz.

**Escenario 2.** Dados 5 intentos de inicio de sesión fallidos, cuando se intenta un sexto, entonces la cuenta queda bloqueada temporalmente por 15 minutos.

## Sección 2: Prototipo navegable

### 2.1 Tipo de prototipo

El prototipo es de alta fidelidad: está construido directamente con la tecnología final (Angular 20 y Tailwind CSS) y conectado a datos de prueba, por lo que se puede navegar y operar como el sistema real. La referencia visual inicial fue una idea general proporcionada por el ingeniero responsable; la organización de pantallas, formularios, tablas, navegación y componentes fue diseñada y construida por el autor.

| **Característica**      | **Detalle**                                                                                                            |
|-------------------------|------------------------------------------------------------------------------------------------------------------------|
| **Fidelidad**           | Alta: interfaz final con datos de prueba y navegación funcional.                                                       |
| **Herramienta**         | Angular 20 (componentes standalone), TypeScript, Tailwind CSS, ApexCharts para gráficos.                               |
| **Estructura**          | Menú lateral por módulos, encabezado con sesión y sucursal, área de trabajo con filtros, tablas y formularios modales. |
| **Responsive**          | Reorganización de menú, columnas y tablas según el ancho (escritorio y móvil).                                         |
| **Estados de pantalla** | Carga, sin registros, error de validación y confirmación mediante avisos.                                              |

### 2.2 Flujo de navegación

Tras iniciar sesión, el usuario llega al dashboard y accede a los módulos por el menú lateral, que muestra solo las opciones permitidas por sus permisos. Cada módulo sigue el mismo patrón: listado con filtros y paginación, y formulario o modal para crear o editar.

<img src="../../img_e1-requerimientos-y-diseno/media/9cc33609826035bec49c038deb25f10741d37b06.png" title="Figura 2.1 — Flujo de navegación general del sistema" style="width:6.66667in;height:3.21875in" alt="Figura 2.1 — Flujo de navegación general del sistema" />

*Figura 2.1 — Flujo de navegación general del sistema*

### 2.3 Pantallas principales

Las capturas corresponden al sistema en ejecución con datos de prueba. Se muestran las pantallas más representativas de cada grupo de requisitos.

<img src="../../img_e1-requerimientos-y-diseno/media/de38c611d6dfca85fc0c9900ad811e1570da58e2.jpg" title="Figura 2.2 — Dashboard financiero con indicadores de ventas, egresos, inventario y utilidad (RF-45)" style="width:6.25in;height:3.30208in" alt="Figura 2.2 — Dashboard financiero con indicadores de ventas, egresos, inventario y utilidad (RF-45)" />

*Figura 2.2 — Dashboard financiero con indicadores de ventas, egresos, inventario y utilidad (RF-45)*

<img src="../../img_e1-requerimientos-y-diseno/media/195e42029f6a5c35279a146ea785fa10e233107b.jpg" title="Figura 2.3 — Venta de servicios: paciente, servicios, resumen de pago y formato de impresión (RF-06, RF-42)" style="width:6.25in;height:3.33333in" alt="Figura 2.3 — Venta de servicios: paciente, servicios, resumen de pago y formato de impresión (RF-06, RF-42)" />

*Figura 2.3 — Venta de servicios: paciente, servicios, resumen de pago y formato de impresión (RF-06, RF-42)*

<img src="../../img_e1-requerimientos-y-diseno/media/be9dee108fc47cc3f26d3390fbc506b97e826592.jpg" title="Figura 2.4 — Gestión de usuarios: estados, roles y permisos asignados (RF-37, RF-38)" style="width:6.25in;height:3.3125in" alt="Figura 2.4 — Gestión de usuarios: estados, roles y permisos asignados (RF-37, RF-38)" />

*Figura 2.4 — Gestión de usuarios: estados, roles y permisos asignados (RF-37, RF-38)*

<img src="../../img_e1-requerimientos-y-diseno/media/786266ae195996d03e5ebe00ee5db45252761f41.jpg" title="Figura 2.5 — Gestión comercial y de servicios: catálogo con filtros y estados (RF-22)" style="width:6.25in;height:3.33333in" alt="Figura 2.5 — Gestión comercial y de servicios: catálogo con filtros y estados (RF-22)" />

*Figura 2.5 — Gestión comercial y de servicios: catálogo con filtros y estados (RF-22)*

<img src="../../img_e1-requerimientos-y-diseno/media/a579cb42417e8b9311284e462c2d07dd37d0fd8f.jpg" title="Figura 2.6 — Gestión de operaciones de caja: saldos por medio de pago y movimientos (RF-16)" style="width:6.25in;height:2.67708in" alt="Figura 2.6 — Gestión de operaciones de caja: saldos por medio de pago y movimientos (RF-16)" />

*Figura 2.6 — Gestión de operaciones de caja: saldos por medio de pago y movimientos (RF-16)*

<img src="../../img_e1-requerimientos-y-diseno/media/47d1634f772e8e9e7016ede724f3ab72eaefa2dd.jpg" title="Figura 2.7 — Bitácora y auditoría con filtros por usuario, acción y fecha (RF-39)" style="width:6.25in;height:2.64583in" alt="Figura 2.7 — Bitácora y auditoría con filtros por usuario, acción y fecha (RF-39)" />

*Figura 2.7 — Bitácora y auditoría con filtros por usuario, acción y fecha (RF-39)*

### 2.4 Validación con usuarios y stakeholders

La validación se hizo de forma incremental durante el desarrollo, mediante reuniones virtuales (Zoom y Microsoft Teams) con el ingeniero responsable y los encargados de la empresa, normalmente los domingos, martes y jueves. En cada sesión se presentaba el avance, se probaban manualmente los flujos y se registraban observaciones que se corregían en la siguiente iteración.

| **Aspecto**       | **Descripción**                                                                                         |
|-------------------|---------------------------------------------------------------------------------------------------------|
| **Método**        | Demostración en vivo del prototipo y prueba manual guiada de los flujos del módulo.                     |
| **Participantes** | Ingeniero responsable, encargados de la empresa y desarrollador.                                        |
| **Frecuencia**    | Sesiones interdiarias o consecutivas según el avance (domingos, martes y jueves).                       |
| **Resultado**     | Observaciones incorporadas como ajustes de pantallas, validaciones y reglas en iteraciones posteriores. |
| **Evidencia**     | Capturas de reuniones y acta de validación (Anexo A).                                                   |

|                                                                                                                                                                                                                                          |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Alcance de la validación: son pruebas manuales de aceptación funcional. No equivalen a una suite automatizada ni a un estudio formal de usabilidad. El acta del Anexo A debe completarse con las fechas y firmas reales de las sesiones. |

## Sección 3: Diseño arquitectónico

La arquitectura se documenta con el modelo C4: contexto (C1), contenedores (C2, que cumple el rol de vista de despliegue) y componentes (C3). Es una aplicación web modular por capas, no una arquitectura de microservicios.

### 3.1 Vista de contexto (C1)

El sistema es utilizado por cuatro grupos de usuarios internos de Reumacenter a través del navegador. Se comunica únicamente con un servidor de correo SMTP; el envío por WhatsApp (Twilio) está deshabilitado y no existe integración con SUNAT.

<img src="../../img_e1-requerimientos-y-diseno/media/d954be0df2742a298c63508bad0c66e8d420359f.png" title="Figura 3.1 — Diagrama de contexto del sistema (C4 nivel 1)" style="width:6.25in;height:3.20833in" alt="Figura 3.1 — Diagrama de contexto del sistema (C4 nivel 1)" />

*Figura 3.1 — Diagrama de contexto del sistema (C4 nivel 1)*

### 3.2 Vista de contenedores y despliegue (C2)

La arquitectura objetivo separa el frontend estático, servido desde la red de borde de Vercel, del backend, que corre en un servidor Linux con Docker Compose junto con la base de datos y un proxy inverso que gestiona el certificado HTTPS.

<img src="../../img_e1-requerimientos-y-diseno/media/0034e73dbb45c7a50ed3cb2a0b98634e647dbc52.png" title="Figura 3.2 — Diagrama de contenedores y despliegue (C4 nivel 2)" style="width:3.75in;height:4.75in" alt="Figura 3.2 — Diagrama de contenedores y despliegue (C4 nivel 2)" />

*Figura 3.2 — Diagrama de contenedores y despliegue (C4 nivel 2)*

| **Contenedor**           | **Tecnología**                                             | **Responsabilidad**                                                                                     |
|--------------------------|------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| **SPA web**              | Angular 20, TypeScript, Tailwind CSS, ApexCharts           | Pantallas, formularios, navegación, guards de ruta, cliente HTTP y cliente WebSocket (STOMP).           |
| **Proxy inverso**        | Caddy 2                                                    | Termina TLS (HTTPS/WSS) con certificado automático y reenvía al puerto 8084 de la API.                  |
| **API REST + WebSocket** | Java 21, Spring Boot 4.0.1, Spring Security, JasperReports | Autenticación JWT RS256, autorización, reglas de negocio, transacciones, reportes PDF y notificaciones. |
| **Base de datos**        | PostgreSQL 16 (laboratorio_db)                             | Persistencia relacional con integridad referencial; eventos LISTEN/NOTIFY.                              |
| **Almacén de archivos**  | Volumen Docker /app/uploads                                | Archivos adjuntos subidos (límite de 10 MB por archivo).                                                |

### 3.3 Vista de componentes del backend (C3)

Dentro de la API, cada dominio sigue el patrón controlador → servicio → repositorio. Los controladores exponen las rutas bajo /api; los servicios concentran las reglas de negocio y definen la frontera transaccional; los repositorios Spring Data JPA acceden a la base de datos. La seguridad y la auditoría son transversales.

<img src="../../img_e1-requerimientos-y-diseno/media/e7714da82f1deb3b6e66a531f8cf749ed0e962a3.png" title="Figura 3.3 — Diagrama de componentes de la API (C4 nivel 3)" style="width:6.66667in;height:5.80208in" alt="Figura 3.3 — Diagrama de componentes de la API (C4 nivel 3)" />

*Figura 3.3 — Diagrama de componentes de la API (C4 nivel 3)*

| **Componente**                 | **Responsabilidad**                                                                                      |
|--------------------------------|----------------------------------------------------------------------------------------------------------|
| **JwtAuthFilter / JwtService** | Valida el token Bearer en cada petición, carga la identidad y firma los tokens con la llave privada RSA. |
| **SecurityConfig**             | Define rutas públicas (login), reglas de acceso y política CORS tomada de la variable CORS_ORIGINS.      |
| **Controladores de dominio**   | Reciben y validan la solicitud, traducen a DTO y devuelven ApiResponse\<T\> o Page\<T\>.                 |
| **Servicios de dominio**       | Aplican reglas (cálculo de venta, numeración, autorizaciones, kardex) dentro de transacciones.           |
| **Repositorios y entidades**   | Mapeo objeto-relacional con JPA/Hibernate y consultas del dominio.                                       |
| **Reportes (JasperReports)**   | Genera tickets 58/80 mm y facturas A4/A5 a partir de plantillas JRXML.                                   |
| **Notificaciones (WebSocket)** | Publica eventos a los usuarios por STOMP; la interfaz consulta el estado persistido.                     |
| **Auditoría**                  | Registra eventos técnicos (audit_log) y de negocio (lab_auditoria).                                      |

### 3.4 Registro de decisiones arquitectónicas (ADR)

| **ID**     | **Decisión**                                                   | **Contexto**                                                                      | **Alternativas**                                                               | **Consecuencias**                                                                                                                   | **Estado**          |
|------------|----------------------------------------------------------------|-----------------------------------------------------------------------------------|--------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------|---------------------|
| **ADR-01** | Monolito modular por capas                                     | Un solo desarrollador, despliegue simple y bajo costo operativo.                  | Microservicios.                                                                | Despliegue y depuración simples; escala verticalmente. Los límites entre dominios se mantienen por paquetes.                        | Aceptada            |
| **ADR-02** | SPA con Angular 20 y Tailwind CSS                              | Interfaz rica con muchas pantallas de formularios y tablas.                       | Plantillas del servidor (Thymeleaf).                                           | Frontend desacoplado y desplegable como archivos estáticos; requiere API bien definida.                                             | Aceptada            |
| **ADR-03** | Spring Boot 4 / Java 21 / Spring Security                      | Ecosistema maduro para API, seguridad, JPA y reportes.                            | Node.js (NestJS), .NET.                                                        | Código tipado y transaccional; mayor consumo de memoria que otras opciones.                                                         | Aceptada            |
| **ADR-04** | PostgreSQL como base de datos                                  | Datos altamente relacionales con necesidad de transacciones e integridad.         | MySQL, NoSQL.                                                                  | Integridad referencial y LISTEN/NOTIFY; requiere administración de copias de seguridad.                                             | Aceptada            |
| **ADR-05** | Autenticación JWT RS256 sin estado                             | Simplicidad para escalar la API y separar frontend y backend.                     | Sesión de servidor con cookie.                                                 | No hay sesión en memoria; el control de sesiones concurrentes se apoya en la tabla Session.                                         | Aceptada            |
| **ADR-06** | Reportes PDF con JasperReports                                 | Necesidad de tickets y facturas con formato fijo.                                 | Generar PDF en el navegador.                                                   | Plantillas JRXML reutilizables; requiere fuentes instaladas en el contenedor.                                                       | Aceptada            |
| **ADR-07** | Notificaciones por WebSocket (STOMP)                           | Avisar autorizaciones y eventos sin recargar.                                     | Polling periódico.                                                             | Menos tráfico; el WebSocket no pasa por el proxy de Vercel y se conecta directo al dominio del backend.                             | Aceptada            |
| **ADR-08** | Despliegue: Vercel + servidor Linux con Docker Compose y Caddy | Producción piloto de una semana, sin que el sistema se «duerma» y a costo mínimo. | Render gratuito (se suspende por inactividad), Railway con créditos de prueba. | Sin arranques en frío; el mismo compose es portable a cualquier VPS.                                                                | Aceptada            |
| **ADR-09** | Auditoría dual (técnica y de negocio)                          | Distinguir quién llamó a la API de qué evento de negocio ocurrió.                 | Un único registro.                                                             | Consultas de auditoría más claras; duplica algo de información.                                                                     | Aceptada            |
| **ADR-10** | Esquema con ddl-auto=update y Flyway deshabilitado             | Las migraciones versionadas estaban incompletas durante el desarrollo.            | Migraciones Flyway completas.                                                  | Agiliza el desarrollo pero no deja historial de cambios del esquema. Deuda técnica: pasar a migraciones versionadas (Entregable 2). | Aceptada con riesgo |

## Sección 4: Diseño detallado

### 4.1 Diagrama de clases del dominio

El diagrama resume las entidades centrales de los flujos de seguridad, venta, caja, compras e inventario, y honorarios. Los nombres y atributos corresponden a las entidades JPA del backend; por legibilidad se muestran solo los atributos principales.

<img src="../../img_e1-requerimientos-y-diseno/media/20bcd35bf6e656480045e2f048ad2dd77e646a95.png" title="Figura 4.1a — Seguridad y organización: usuarios, roles, permisos, sucursales, puntos y sesiones" style="width:3.75in;height:3.51042in" alt="Figura 4.1a — Seguridad y organización: usuarios, roles, permisos, sucursales, puntos y sesiones" />

*Figura 4.1a — Seguridad y organización: usuarios, roles, permisos, sucursales, puntos y sesiones*

<img src="../../img_e1-requerimientos-y-diseno/media/765372ba797d41e9b6ea86e83bd3a3986d54cffc.png" title="Figura 4.1b — Ventas y caja: paciente, venta, detalle, notas de crédito, autorizaciones y movimientos de caja" style="width:6.25in;height:4.63542in" alt="Figura 4.1b — Ventas y caja: paciente, venta, detalle, notas de crédito, autorizaciones y movimientos de caja" />

*Figura 4.1b — Ventas y caja: paciente, venta, detalle, notas de crédito, autorizaciones y movimientos de caja*

<img src="../../img_e1-requerimientos-y-diseno/media/6a040d3fb5d649167ec807ee671120d1f8aaad28.png" title="Figura 4.1c — Compras, inventario y honorarios: proveedor, orden, compra, kardex y pagos" style="width:3.54167in;height:5.94792in" alt="Figura 4.1c — Compras, inventario y honorarios: proveedor, orden, compra, kardex y pagos" />

*Figura 4.1c — Compras, inventario y honorarios: proveedor, orden, compra, kardex y pagos*

### 4.2 Diagrama de secuencia: registrar una venta

El flujo muestra cómo una venta atraviesa interfaz, controlador, servicio, motor de precios y base de datos dentro de una sola transacción. Si falla la validación de correlativo o de orden duplicada, no se guarda nada.

<img src="../../img_e1-requerimientos-y-diseno/media/520490fc16463be39a8440f40b117f74ced4f9de.png" title="Figura 4.2 — Secuencia de registro de una venta (POST /api/ventas)" style="width:6.66667in;height:3.125in" alt="Figura 4.2 — Secuencia de registro de una venta (POST /api/ventas)" />

*Figura 4.2 — Secuencia de registro de una venta (POST /api/ventas)*

### 4.3 Diagramas de estados

Tres entidades tienen un ciclo de vida con reglas de transición: la venta, la autorización de descuento y la orden de compra.

<table>
<colgroup>
<col style="width: 50%" />
<col style="width: 50%" />
</colgroup>
<tbody>
<tr class="odd">
<td><p><img src="../../img_e1-requerimientos-y-diseno/media/4bb105928e49af257b25295526b14d1dbee7b275.png" title="Figura 4.3a — Estados de la venta" style="width:2.60417in;height:3.22917in" alt="Figura 4.3a — Estados de la venta" /></p>
<p><em>Figura 4.3a — Estados de la venta</em></p></td>
<td><p><img src="../../img_e1-requerimientos-y-diseno/media/9689e5a7bed30ff724922e78b5bb0086dd4af412.png" title="Figura 4.3b — Estados de la autorización de descuento" style="width:2.60417in;height:3.34375in" alt="Figura 4.3b — Estados de la autorización de descuento" /></p>
<p><em>Figura 4.3b — Estados de la autorización de descuento</em></p></td>
</tr>
<tr class="even">
<td><p><img src="../../img_e1-requerimientos-y-diseno/media/bcf922b37fd4f5872664857f971f36bd8c553eb8.png" title="Figura 4.3c — Estados de la orden de compra" style="width:1.77083in;height:3.14583in" alt="Figura 4.3c — Estados de la orden de compra" /></p>
<p><em>Figura 4.3c — Estados de la orden de compra</em></p></td>
<td><p><strong>Transiciones relevantes</strong></p>
<ul>
<li><p>Venta: solo se anula si no tiene nota de crédito EMITIDA (RN-06).</p></li>
<li><p>Autorización: una vez USADA no se reutiliza (RN-04).</p></li>
<li><p>Orden de compra: la recepción genera kardex y cuenta por pagar (RN-10).</p></li>
</ul></td>
</tr>
</tbody>
</table>

### 4.4 Modelo de diseño detallado

#### 4.4.1 Estructura del backend

El código se organiza por dominio funcional en el paquete com.pe.finanzas. Cada dominio (por ejemplo venta_registro, compra, caja_chica, kardex) contiene sus controladores, servicios, repositorios, entidades y DTO. Los paquetes transversales son:

- **config / security / auth:** configuración de Spring Security, filtro JWT, CORS, WebSocket y servicios de autenticación.

- **exception:** excepciones del dominio (recurso no encontrado, solicitud inválida) y manejo global de errores.

- **audit:** registro técnico de peticiones y eventos de negocio.

- **shared / reports:** utilidades compartidas, almacenamiento de archivos y generación de reportes con plantillas JRXML.

- **jobs:** tareas programadas y escucha de eventos de la base de datos (LISTEN/NOTIFY).

#### 4.4.2 Contrato de la API

- Recursos REST bajo el prefijo /api, agrupados por dominio y con métodos HTTP estándar.

- Autenticación con cabecera Authorization: Bearer \<JWT\>; las rutas de /api/auth son públicas.

- Respuestas envueltas en ApiResponse\<T\>; los listados devuelven Page\<T\> con parámetros page y size más filtros.

- DTO de entrada y salida para no exponer directamente las entidades JPA.

- Errores traducidos a códigos HTTP: 400 (solicitud inválida), 401 (sin sesión), 403 (sin permiso), 404 (no existe) y 409/422 para reglas de negocio.

| **Dominio**   | **Rutas base (/api/…)**                                                                                                                                                                     | **Operaciones principales**                                                                |
|---------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------|
| **Seguridad** | auth, users, roles, permissions, menu, restrictions, segregations, delegations, approvals, security                                                                                         | Login, usuarios, roles, permisos, menú por permisos, delegaciones.                         |
| **Maestros**  | servicios, categorias, niveles, paquetes, beneficios, convenios, tipos-paciente, tipos-atencion, tipos-documento, grupo-familiar, empresas, companias, sucursales, puntos, puntos-documento | CRUD de catálogos y estructura organizacional.                                             |
| **Ventas**    | ventas, notas-credito, solicitudes-anulacion, autorizaciones-descuento, autorizacion-nc, reimpresiones                                                                                      | Crear/consultar/anular venta, imprimir PDF, pagar orden pendiente, autorizar y reimprimir. |
| **Tesorería** | caja-general, caja-chica, arqueos, cred-ctacte, cuentas-corrientes, cuentas-proveedor, cronograma-pagos, otros-pagos                                                                        | Apertura, movimientos, arqueo y cierre; créditos y pagos a proveedores.                    |
| **Logística** | proveedores, ordenes-compra, compras, productos, inventario, kardex, tipo-temperatura                                                                                                       | Órdenes, recepción de compras, stock y consulta de kardex.                                 |
| **RR. HH.**   | rrhh/empleados, rrhh/datos-medico, pagos-personal, pagos-honorarios, reportes/valorizacion                                                                                                  | Personal, valorización y liquidación de honorarios.                                        |
| **Finanzas**  | plan-cuentas, activos                                                                                                                                                                       | Plan de cuentas PMT y activos fijos.                                                       |
| **Reportes**  | reportes/ventas, compras, descuentos, honorarios, pagos-personal, comprobantes-anulados, correlatividad; rep-empresas-atendidas; reports/audit; dashboard; notificaciones                   | Listados, exportaciones, bitácora, indicadores y notificaciones.                           |

#### 4.4.3 Estructura del frontend

El frontend Angular se organiza en core (servicios transversales, interceptores y guards), models (interfaces tipadas), services (clientes HTTP por dominio), shared (componentes reutilizables: tablas, filtros, modales, avisos) y features, con un directorio por pantalla o módulo (venta-servicios, arqueo-caja, orden-compra, kardex, usuarios, rep-ventas, entre otros). El enrutamiento usa carga diferida y guards que adaptan la navegación al perfil; la autorización efectiva permanece en el backend.

#### 4.4.4 Seguridad en el diseño

- Flujo de login: credenciales → validación y control de intentos → emisión de JWT RS256 → registro de sesión.

- Cada petición pasa por el filtro JWT; luego se evalúan roles, permisos y ámbito (sucursal/punto) en el servicio.

- Las llaves RSA y credenciales se inyectan por archivo y variables de entorno; no se versionan.

- Las operaciones sensibles (descuentos, notas de crédito, anulaciones) requieren aprobador distinto del solicitante y dejan registro de auditoría.

## Anexos

### Anexo A: Acta de validación con stakeholders

Formato para registrar cada sesión de validación. Complete fechas, participantes y firmas con los datos reales de las reuniones realizadas.

| **N.°** | **Fecha**              | **Módulos / funcionalidades revisadas**                            | **Observaciones** | **Estado**                   |
|---------|------------------------|--------------------------------------------------------------------|-------------------|------------------------------|
| 1       | \_\_\_\_/\_\_\_\_/2026 | Maestros, seguridad y acceso (RF-01 a RF-05, RF-35 a RF-41)        |                   | Conforme / Con observaciones |
| 2       | \_\_\_\_/\_\_\_\_/2026 | Ventas, descuentos, notas de crédito y anulaciones (RF-06 a RF-15) |                   | Conforme / Con observaciones |
| 3       | \_\_\_\_/\_\_\_\_/2026 | Caja, cuentas por cobrar y por pagar (RF-16 a RF-21)               |                   | Conforme / Con observaciones |
| 4       | \_\_\_\_/\_\_\_\_/2026 | Compras, inventario y kardex (RF-22 a RF-28)                       |                   | Conforme / Con observaciones |
| 5       | \_\_\_\_/\_\_\_\_/2026 | Honorarios, personal y reportes (RF-29 a RF-34, RF-42 a RF-47)     |                   | Conforme / Con observaciones |

| **Rol**                 | **Nombre y apellidos** | **Cargo**                  | **Firma** |
|-------------------------|------------------------|----------------------------|-----------|
| Ingeniero responsable   |                        |                            |           |
| Encargado de la empresa |                        |                            |           |
| Desarrollador           | Omar Condori Pachauri  | Estudiante / desarrollador |           |

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<tbody>
<tr class="odd">
<td><p><strong>[ Espacio para evidencia — capturas de las reuniones virtuales (Zoom / Teams) ]</strong></p>
<p><em>Inserte aquí las capturas de pantalla de las sesiones de validación.</em></p></td>
</tr>
</tbody>
</table>

### Anexo B: Matriz de trazabilidad de requerimientos

Relaciona cada grupo de requisitos con su historia de usuario, la pantalla (carpeta de features del frontend), la ruta de la API y las entidades que lo soportan.

| **Requisitos**                 | **Historia / CU** | **Pantalla (features)**                                                     | **API (/api/…)**                                                 | **Entidades**                                                                        |
|--------------------------------|-------------------|-----------------------------------------------------------------------------|------------------------------------------------------------------|--------------------------------------------------------------------------------------|
| **RF-01 a RF-03**              | CU-03             | venta-servicios (alta de paciente)                                          | grupo-familiar, tipos-paciente                                   | DatosPersonales, DatosPaciente, GrupoFamiliar                                        |
| **RF-04, RF-35, RF-36**        | CU-02             | gestion-empresas, gestion-companias, sucursales                             | empresas, companias, sucursales, puntos                          | Empresa, Compania, Sucursal, Punto                                                   |
| **RF-06 a RF-09**              | HU-01             | venta-servicios, gestion-paquetes, correlatividad                           | ventas, paquetes, convenios                                      | VentaRegistro, VentaDetalle, Paquete, SerieCorrelativo                               |
| **RF-10**                      | HU-01             | reimprimir                                                                  | reimpresiones, ventas/{id}/imprimir                              | VentaRegistro                                                                        |
| **RF-11, RF-15**               | HU-04             | anular-comprobante, autorizar-anularcomprobante                             | solicitudes-anulacion                                            | SolicitudAnulacion                                                                   |
| **RF-12, RF-13**               | HU-02, HU-03      | autorizar-descuento                                                         | autorizaciones-descuento                                         | AutorizacionDescuento                                                                |
| **RF-14**                      | HU-04             | nota-credito, listar-notas-credito, autorizar-notacredito                   | notas-credito, autorizacion-nc                                   | NotaCredito, AutorizacionNotaCredito                                                 |
| **RF-16, RF-17**               | HU-05             | registro-movimientos, arqueo-caja, ingresos-gastos                          | caja-general, caja-chica, arqueos                                | CajaGeneral, CajaChica, CajaChicaMovimiento, ArqueoCaja                              |
| **RF-18, RF-19**               | HU-09             | cuentas-convenio, cuentas-corrientes                                        | cred-ctacte, convenios                                           | CredCtaCte, CredPlan                                                                 |
| **RF-20, RF-21**               | HU-09             | cuentas-pagar, cuentas-corrientes                                           | cuentas-proveedor, cronograma-pagos, cuentas-corrientes          | CuentaProveedor, CronogramaPago, CuentaCorriente                                     |
| **RF-22, RF-23**               | HU-06             | gestion-catalogo, niveles, productos                                        | servicios, niveles, productos                                    | Catalogo, Nivel, Producto                                                            |
| **RF-24 a RF-27**              | HU-06             | proveedores, orden-compra, registrar-compras, ajustar-compra, anular-compra | proveedores, ordenes-compra, compras                             | Proveedor, OrdenCompra, Compra, DetalleCompra                                        |
| **RF-28**                      | HU-07             | kardex                                                                      | kardex, inventario                                               | Kardex                                                                               |
| **RF-29 a RF-32**              | HU-08             | gestion-empleados, valorizacion, pago-personal, pagorec-honorarios          | rrhh/\*, pagos-personal, pagos-honorarios, reportes/valorizacion | DatosMedico, PagoComprobante, PagoConceptoDetalle                                    |
| **RF-33, RF-34**               | —                 | plan-cuentasPMT                                                             | plan-cuentas, activos                                            | Activo                                                                               |
| **RF-37, RF-38, RF-40, RF-41** | HU-10             | auth, usuarios, gestion-claves                                              | auth, users, roles, permissions, segregations, delegations       | DatosPersonales, Role, Permission, Session, DelegacionTemporal, SegregacionFunciones |
| **RF-39**                      | HU-11             | reports (bitácora)                                                          | reports/audit                                                    | AuditLog                                                                             |
| **RF-42 a RF-45**              | HU-12             | rep-ventas, rep-compras, rep-descuentos, rep-honorarios, dashboard          | reportes/\*, dashboard                                           | Consultas sobre las entidades de ventas, compras y pagos                             |
| **RF-46**                      | HU-03             | notificaciones-service                                                      | notificaciones (WebSocket)                                       | —                                                                                    |
| **RF-47, RNF-16**              | —                 | shared, core (layout, menú, componentes)                                    | —                                                                | —                                                                                    |

Los requisitos no funcionales se verifican de forma transversal: RNF-01 a RNF-07 en la capa de seguridad (JwtService, SecurityConfig), RNF-08 y RNF-09 en los listados paginados, RNF-11 y RNF-12 en servicios transaccionales y restricciones del esquema, y RNF-20 en el Dockerfile y el docker-compose.

### Anexo C: Diagramas complementarios

Los diagramas de casos de uso, secuencia, estados y clases se encuentran en las Secciones 1 y 4. Para complementar la entrega se recomienda adjuntar los archivos fuente (Graphviz/Mermaid o Draw.io) y los diagramas del modelo de datos del Informe de Prácticas (Figuras 11 a 14, Sección 8), que agrupan las entidades por dominio.

### Anexo D: Evidencia de sustentación

Exposición breve defendiendo el entregable ante el docente.

| **Dato**                             | **Detalle**            |
|--------------------------------------|------------------------|
| **Fecha de sustentación**            | \_\_\_\_/\_\_\_\_/2026 |
| **Docente evaluador**                |                        |
| **Duración**                         |                        |
| **Enlace o archivo de la grabación** |                        |

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<tbody>
<tr class="odd">
<td><p><strong>[ Espacio para evidencia — captura o enlace de la sustentación ]</strong></p>
<p><em>Adjunte la captura de la exposición o el enlace a la grabación.</em></p></td>
</tr>
</tbody>
</table>
