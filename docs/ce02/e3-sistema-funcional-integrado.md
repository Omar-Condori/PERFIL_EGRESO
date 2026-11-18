# CE023 - S2 - E3: Sistema de Software Funcional Integrado

**Equipo:** Omar Condori Pachauri (201810211) · Merry Elizabeth Flores Llanque (202211953) · Jaqueline Huahuaccapa Ccama (202210712) — Ciclo 10.

## Resumen ejecutivo

Reumacenter E.I.R.L. es un laboratorio clínico de Juliaca que manejaba sus ventas, cajas, compras, inventario y pagos de honorarios en procesos separados, con poca trazabilidad de quién autorizó un descuento, anuló un comprobante o movió dinero. FINANZAS es el sistema que unifica esas operaciones: cada venta de servicios se cobra, afecta la caja y queda registrada y auditada en un solo flujo, y la gerencia ve los resultados en un tablero y en reportes.

El sistema es una aplicación web de tres capas: un frontend Angular 20 que el personal usa desde el navegador, un backend Spring Boot 4 (Java 21) que expone una API REST protegida con JWT y una base de datos PostgreSQL 16. Un cajero inicia sesión, elige al paciente y los servicios, cobra en efectivo, tarjeta, Yape/Plin o transferencia, imprime el comprobante en PDF y la caja se actualiza sola; un supervisor aprueba descuentos o anulaciones desde su propia bandeja; el contador consulta compras, pagos y reportes.

#### El sistema en cifras (medido sobre el repositorio)

| **Componente**                     | **Cifra**                                                                                                                                  |
|------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------|
| Backend (Java 21 + Spring Boot 4)  | ≈ 44 200 líneas · 61 módulos de negocio · 67 controladores · 325 endpoints REST · 128 servicios · 72 repositorios · 76 entidades · 157 DTO |
| Frontend (Angular 20 + TypeScript) | ≈ 42 800 líneas · 143 componentes · 66 servicios · 51 módulos funcionales (features) con carga diferida                                    |
| Seguridad                          | JWT RS256, 173 anotaciones @PreAuthorize, bloqueo por intentos, sesiones controladas, auditoría de escrituras                              |
| Datos                              | 76 tablas en PostgreSQL 16 (Entregable 2)                                                                                                  |
| Historial del repositorio          | 115 commits entre el 15/01/2026 y el 17/11/2026                                                                                            |

|                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Estado de la evidencia.** Las capturas de funcionamiento (sección 2) provienen del sistema en ejecución local del estudiante, con datos de prueba. El despliegue en la nube está preparado (Docker Compose, Caddy, Vercel) pero la evidencia de publicación se completa en el Anexo D cuando la máquina virtual esté activa. La sección 3.5 declara con honestidad las deudas técnicas encontradas, incluidas dos de seguridad que deben corregirse antes de abrir el sistema a internet. |

## 1. Arquitectura e integración

### 1.1 Arquitectura general

FINANZAS sigue una arquitectura cliente-servidor de tres capas con un backend monolítico modular. Se eligió un monolito modular (un solo despliegue, paquetes por dominio de negocio) y no microservicios porque el sistema lo opera un equipo de una persona y una sola empresa: un único artefacto simplifica el despliegue, las transacciones entre ventas, caja e inventario son locales a la base de datos y el costo de operación es mínimo. Los paquetes por dominio dejan abierta la puerta a separar servicios más adelante.

<img src="../../img_e3-sistema-funcional-integrado/media/a97c55887a17e7afe1c21dbbbe29123961a52532.png" title="Figura 1. Arquitectura de despliegue objetivo: SPA en Vercel; API, base de datos y proxy en una VM con Docker Compose." style="width:4.16667in;height:4.33333in" alt="Figura 1. Arquitectura de despliegue objetivo: SPA en Vercel; API, base de datos y proxy en una VM con Docker Compose." />

*Figura 1. Arquitectura de despliegue objetivo: SPA en Vercel; API, base de datos y proxy en una VM con Docker Compose.*

| **Componente** | **Tecnología**                                                                   | **Responsabilidad**                                                                                    |
|----------------|----------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------|
| Frontend       | Angular 20, TypeScript, Tailwind CSS 4, ApexCharts, SweetAlert2                  | Interfaz, validación de formularios, control de rutas por permiso, gráficos, exportación a Excel y PDF |
| Backend        | Spring Boot 4.0.1, Java 21, Spring Security, Spring Data JPA, JasperReports 6.21 | Reglas de negocio, API REST, seguridad, auditoría, reportes y comprobantes PDF                         |
| Base de datos  | PostgreSQL 16                                                                    | Persistencia, integridad, funciones y triggers (Entregable 2)                                          |
| Proxy inverso  | Caddy 2                                                                          | HTTPS automático, compresión y enrutamiento hacia el backend                                           |
| Tiempo real    | WebSocket STOMP + LISTEN/NOTIFY de PostgreSQL                                    | Notificaciones a usuarios (por ejemplo, solicitudes de autorización)                                   |
| Contenedores   | Docker Compose (db, backend, caddy)                                              | Despliegue reproducible con un solo comando                                                            |

### 1.2 Capas y módulos del backend

Cada petición atraviesa las mismas capas, en el mismo orden. La separación hace que el controlador no conozca la base de datos y que las reglas de negocio vivan solo en los servicios.

<img src="../../img_e3-sistema-funcional-integrado/media/a3cd7d2680a3f9457286c8a06d8773cdd5ce29c9.png" title="Figura 2. Capas del backend y elementos transversales (conteos medidos en el código fuente)." style="width:4.16667in;height:3.51042in" alt="Figura 2. Capas del backend y elementos transversales (conteos medidos en el código fuente)." />

*Figura 2. Capas del backend y elementos transversales (conteos medidos en el código fuente).*

El código está organizado «por función» y no «por tipo»: cada dominio de negocio es un paquete con sus propias subcarpetas controller, service, repository, entity y dto. Los 61 paquetes se agrupan así:

| **Grupo**             | **Paquetes principales**                                                                                                                             | **Responsabilidad**                                                                |
|-----------------------|------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------|
| Seguridad e identidad | auth, security, users, roles, permissions, accesos, audit, sucursales                                                                                | Login, JWT, roles y permisos, contexto de sucursal, bitácora                       |
| Ventas y cobros       | venta_registro, motor_precios, descuentos, autorizar_descuento, autorizar_notacredito, nota_credito, solicitud_anulacion, reimprimir, correlatividad | Comprobantes, precios, descuentos y anulaciones con autorización                   |
| Caja y tesorería      | caja_chica, caja_general, arqueo_caja, otros_pagos, cuentas_corrientes, cuentas_convenio                                                             | Efectivo, transferencias entre cajas, arqueos, créditos                            |
| Compras e inventario  | compra, orden_compra, proveedores, productos, kardex, valorizacion, activos_fijos                                                                    | Abastecimiento, stock y valorización                                               |
| Personal y honorarios | empleados, pago_personal, pagorec_honorarios, datos_medico, plan_cuentas_pmt                                                                         | Planilla, pago a médicos y honorarios                                              |
| Catálogos y pacientes | catalogo, niveles, paquetes, paciente, grupo_familiar, compania, empresa                                                                             | Servicios, tarifas, tipos de atención, pacientes y empresas                        |
| Reportes y panel      | dashboard, reports, rep_ventas, rep_compras, rep_descuentos, rep_honorarios, rep_pago_personal, rep_comprobantes_anulados, rep_empresas_atendidas    | Indicadores y reportes exportables                                                 |
| Infraestructura       | config, exception, jobs, service, shared, system, menu                                                                                               | Configuración, errores globales, tareas programadas, notificaciones, menú dinámico |

### 1.3 Estructura del frontend

El frontend es una aplicación de una sola página con componentes independientes (standalone) y rutas cargadas bajo demanda (loadComponent): el navegador descarga el código de un módulo solo cuando el usuario entra a él, lo que reduce el tiempo de la primera carga. Se organiza en cuatro zonas:

- **core:** guards (AuthGuard con permisos por ruta, PuntoVentaCheckGuard), interceptor HTTP (inyecta el token y trata 401/403) y servicios transversales.

- **features:** 51 módulos funcionales, uno por pantalla de negocio (venta-servicios, caja general, honorarios, reportes…).

- **shared:** 13 componentes reutilizables (tabla general, paginación, modales, selectores, filtros), layout, cabecera y menú lateral dinámico.

- **models / pipes / services:** contratos de datos, formateadores y servicios compartidos que hablan con la API.

### 1.4 Integración con la base de datos

- **Acceso:** Spring Data JPA con Hibernate; 72 repositorios; las consultas complejas usan JPQL. La conexión se configura por variables de entorno (DB_URL, DB_USER, DB_PASSWORD), con valores por defecto para desarrollo local.

- **Transacciones:** 332 usos de @Transactional en la capa de servicio; las operaciones de solo lectura se marcan readOnly para optimizar.

- **Concurrencia:** el correlativo de cada serie de comprobantes se obtiene con bloqueo pesimista (SELECT … FOR UPDATE), de modo que dos cajeros no pueden emitir el mismo número.

- **Esquema:** lo genera Hibernate (ddl-auto=update); Flyway está presente pero desactivado. El Entregable 2 documenta el modelo, la integridad adicional, los triggers, los roles y las pruebas de rendimiento.

- **Eventos hacia la interfaz:** una tabla cola_eventos y un hilo que ejecuta LISTEN sobre el canal «nuevo_evento_outbox» (patrón outbox): cuando se inserta un evento, PostgreSQL avisa al backend, que lo procesa y lo envía por WebSocket. Si la conexión cae, el hilo reintenta cada 10 s, y un respaldo programado cada 30 s evita perder eventos.

### 1.5 Integración con APIs y servicios externos

El sistema no depende de ninguna API de terceros para operar: la autenticación, la facturación interna, los reportes y las notificaciones se resuelven con componentes propios. Esto es una decisión consciente de costo y disponibilidad. Hay dos integraciones internas relevantes:

- **Documentación de la API:** springdoc-openapi publica la especificación OpenAPI y la interfaz Swagger UI, útil para probar los endpoints durante la sustentación.

- **Comprobantes:** JasperReports genera PDF en cuatro formatos (ticket 58 mm, ticket 80 mm, A5 y A4) a partir de plantillas .jrxml; el cajero elige el formato al cobrar.

Observación técnica: el repositorio contiene la carpeta notificaciones-service, pero hoy solo tiene archivos de compilación y del IDE, sin código fuente; las notificaciones realmente funcionan dentro del backend (WebSocket + outbox). Del mismo modo, las propiedades de Twilio aparecen en la configuración pero no hay código ni dependencia que las use. Ambos son restos que conviene retirar para evitar confusión.

### 1.6 Decisiones arquitectónicas

| **Decisión**                      | **Motivo**                                                                                | **Contrapartida asumida**                                                                         |
|-----------------------------------|-------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Monolito modular                  | Un equipo, una empresa; transacciones locales; despliegue simple                          | Todo se escala junto; requiere disciplina para no acoplar paquetes                                |
| API REST sin estado con JWT RS256 | Cualquier instancia valida el token con la clave pública; permite escalar horizontalmente | Un token emitido es válido hasta que expira (30 min); por eso existe además una tabla de sesiones |
| PostgreSQL con LISTEN/NOTIFY      | Una sola base cubre datos y eventos; sin broker adicional                                 | No sustituye a un broker si hubiera varias instancias con mucho tráfico                           |
| Angular con rutas diferidas       | Primera carga rápida en equipos modestos                                                  | Mayor número de archivos que gestionar                                                            |
| Docker Compose + Caddy            | Despliegue reproducible con HTTPS automático y costo cero                                 | Un único servidor: sin alta disponibilidad                                                        |

## 2. Funcionalidad del sistema

### 2.1 Módulos implementados

El menú lateral organiza la aplicación en ocho áreas. La columna de estado distingue lo que tiene captura de pantalla en este documento de lo que está implementado en el código pero aún sin captura.

| **Área**           | **Pantallas (features)**                                                                                                                                          | **Estado**                                  |
|--------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------|
| Configuraciones    | usuarios, roles y permisos, claves, niveles y tipos de atención, catálogo de servicios, paquetes, empresas, compañías, correlatividad, sucursales, autorizaciones | Capturas: usuarios, niveles, catálogo       |
| Procesos           | productos, cuentas por pagar, pago de personal, pago de honorarios, otros pagos, plan de cuentas, valorización                                                    | Captura: honorarios                         |
| RR. HH.            | empleados, médicos, pago de personal                                                                                                                              | Implementado; sin captura en este documento |
| Venta de servicios | venta, anular comprobante, anular por falta de autorización, notas de crédito, reimprimir, pre-ventas, solicitudes                                                | Captura: venta                              |
| Compras            | registrar compras, órdenes de compra, ajustar y anular compra, proveedores, kardex                                                                                | Implementado; sin captura en este documento |
| Caja general       | gestión de operaciones, arqueo, cuentas                                                                                                                           | Captura: gestión de operaciones             |
| Caja chica         | ingresos y gastos, registro de movimientos                                                                                                                        | Implementado; sin captura en este documento |
| Reportes           | ventas, compras, descuentos, honorarios, pagos, otros pagos, comprobantes anulados, empresas atendidas, bitácora                                                  | Captura: bitácora y auditoría               |

### 2.2 Flujo de autenticación

El acceso está protegido de extremo a extremo. El backend verifica credenciales con BCrypt, bloquea la cuenta tras 5 intentos fallidos durante 15 minutos, limita a 3 las sesiones simultáneas y marca como sospechosa una sesión con cambio de IP en menos de una hora. El frontend guarda el token, lo envía en cada petición y reacciona a los errores: un 401 cierra la sesión y lleva al login; un 403 lleva a la pantalla de acceso denegado.

<img src="../../img_e3-sistema-funcional-integrado/media/daf337e38955c0d3d4525689ed35c80450b782cc.png" title="Figura 3. Secuencia de inicio de sesión y manejo de 401/403." style="width:4.89583in;height:3.3125in" alt="Figura 3. Secuencia de inicio de sesión y manejo de 401/403." />

*Figura 3. Secuencia de inicio de sesión y manejo de 401/403.*

### 2.3 Flujo principal: venta de servicios y cobro

Es el flujo central del sistema. En una sola transacción de base de datos, el backend genera el correlativo, calcula los importes, consume la autorización de descuento (si la hay), guarda la cabecera y el detalle de la venta y registra el ingreso en la caja abierta del cajero. Si cualquier paso falla, no se guarda nada. Si el cajero no tiene una caja abierta, el sistema rechaza el cobro con un mensaje claro.

<img src="../../img_e3-sistema-funcional-integrado/media/f55b1020158e246a7ccebd4345c07a92d8905397.png" title="Figura 4. Secuencia de registro de una venta (VentaController → VentaServiceImpl → base de datos y caja)." style="width:6.35417in;height:3.17708in" alt="Figura 4. Secuencia de registro de una venta (VentaController → VentaServiceImpl → base de datos y caja)." />

*Figura 4. Secuencia de registro de una venta (VentaController → VentaServiceImpl → base de datos y caja).*

<img src="../../img_e3-sistema-funcional-integrado/media/a6f628450bdc8282db09a7b9791753aee01d5df7.jpg" title="Figura 5. Pantalla de venta de servicios: paciente, servicios del catálogo, pago mixto y formato de impresión." style="width:5.41667in;height:2.89583in" alt="Figura 5. Pantalla de venta de servicios: paciente, servicios del catálogo, pago mixto y formato de impresión." />

*Figura 5. Pantalla de venta de servicios: paciente, servicios del catálogo, pago mixto y formato de impresión.*

### 2.4 Flujos con autorización

Tres operaciones sensibles exigen que un segundo usuario con permiso las apruebe antes de ejecutarse. Cada una tiene su propio ciclo de estados, almacenado en base de datos y auditado:

| **Operación**            | **Estados**                              | **Regla clave**                                                                              |
|--------------------------|------------------------------------------|----------------------------------------------------------------------------------------------|
| Descuento                | PENDIENTE → APROBADA / RECHAZADA → USADO | La autorización se consume al cobrar; el monto final debe cuadrar con original − descuento   |
| Nota de crédito          | EMITIDA → ANULADA                        | Una venta con nota de crédito emitida no puede anularse                                      |
| Anulación de comprobante | PENDIENTE → APROBADO / RECHAZADO         | Exige motivo; si se devolvió dinero, requiere caja abierta; la venta anulada queda inmutable |

### 2.5 Gestión de usuarios, roles y permisos

El control de acceso es por roles y permisos (RBAC). Cada endpoint declara el permiso que exige, por ejemplo @PreAuthorize("hasAuthority('VENTA_CREAR')"); el menú y las rutas del frontend se muestran u ocultan según los mismos permisos. Además existen roles jerárquicos, segregación de funciones (roles que no pueden coincidir en una persona), delegación temporal de un rol y contexto de sucursal (cada usuario opera en la sucursal que tiene asignada).

<img src="../../img_e3-sistema-funcional-integrado/media/6f1d6ca9f974482a40393828a98e43034f4bc981.jpg" title="Figura 6. Gestión de usuarios: estado, roles y acciones disponibles." style="width:5.41667in;height:2.875in" alt="Figura 6. Gestión de usuarios: estado, roles y acciones disponibles." />

*Figura 6. Gestión de usuarios: estado, roles y acciones disponibles.*

### 2.6 Evidencias funcionales

Capturas del sistema en ejecución local con datos de prueba:

<img src="../../img_e3-sistema-funcional-integrado/media/a11704e7ac93f9f18e04ae8805ef2efe7ee05690.jpg" title="Figura 7. Dashboard financiero: ventas, egresos, inventario y utilidad con selector de período." style="width:5.20833in;height:2.75in" alt="Figura 7. Dashboard financiero: ventas, egresos, inventario y utilidad con selector de período." />

*Figura 7. Dashboard financiero: ventas, egresos, inventario y utilidad con selector de período.*

<img src="../../img_e3-sistema-funcional-integrado/media/0fbe1d4ca4b8421b180b5443feef8d87df223f52.jpg" title="Figura 8. Gestión comercial y de servicios (catálogo, tarifas y estado)." style="width:5.20833in;height:2.78125in" alt="Figura 8. Gestión comercial y de servicios (catálogo, tarifas y estado)." />

*Figura 8. Gestión comercial y de servicios (catálogo, tarifas y estado).*

<img src="../../img_e3-sistema-funcional-integrado/media/bb5ba8d2b1e9dc33b2a206bda1381c8f9db48a70.jpg" title="Figura 9. Gestión de niveles y tipos de atención." style="width:5.20833in;height:2.80208in" alt="Figura 9. Gestión de niveles y tipos de atención." />

*Figura 9. Gestión de niveles y tipos de atención.*

<img src="../../img_e3-sistema-funcional-integrado/media/3297a674487167b196a87bab55f1fe1b617da1ec.jpg" title="Figura 10. Caja general: saldos por medio de pago y operaciones." style="width:5.20833in;height:2.79167in" alt="Figura 10. Caja general: saldos por medio de pago y operaciones." />

*Figura 10. Caja general: saldos por medio de pago y operaciones.*

<img src="../../img_e3-sistema-funcional-integrado/media/a526f17bb94eb2ad8ea283ce9b478d8139230b05.jpg" title="Figura 11. Pago y registro de honorarios." style="width:5.20833in;height:2.22917in" alt="Figura 11. Pago y registro de honorarios." />

*Figura 11. Pago y registro de honorarios.*

<img src="../../img_e3-sistema-funcional-integrado/media/a02281d0da1aa0a7e9a6b47798c39f6a902b58d6.jpg" title="Figura 12. Bitácora y auditoría: acción, módulo, usuario, IP y estado, con exportación a Excel." style="width:5.20833in;height:2.19792in" alt="Figura 12. Bitácora y auditoría: acción, módulo, usuario, IP y estado, con exportación a Excel." />

*Figura 12. Bitácora y auditoría: acción, módulo, usuario, IP y estado, con exportación a Excel.*

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<tbody>
<tr class="odd">
<td><p><strong>[ Demo en vivo ]</strong></p>
<p><em>Enlace o código QR de la demo desplegada y fecha de la prueba (se completa tras el despliegue)</em></p></td>
</tr>
</tbody>
</table>

## 3. Código y diseño técnico

### 3.1 Estructura del repositorio

Un solo repositorio (monorepo) contiene todos los componentes, lo que permite versionar juntos el frontend, el backend y el despliegue.

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<tbody>
<tr class="odd">
<td><p>ERP-CLINICO/</p>
<p>├── finanzas-backend/ Spring Boot 4 · Java 21 · Maven</p>
<p>│ ├── Dockerfile imagen multi-etapa (build → JRE)</p>
<p>│ └── src/main/</p>
<p>│ ├── java/com/pe/finanzas/</p>
<p>│ │ ├── &lt;dominio&gt;/ 61 paquetes: controller · service(+impl) ·</p>
<p>│ │ │ repository · entity · dto</p>
<p>│ │ ├── config/ Security, CORS, WebSocket, OpenAPI, Jackson</p>
<p>│ │ ├── exception/ 9 excepciones + GlobalExceptionHandler</p>
<p>│ │ ├── jobs/ ... tareas programadas (@Scheduled)</p>
<p>│ │ └── service/ PostgresEventListenerService (outbox)</p>
<p>│ └── resources/</p>
<p>│ ├── application.properties (variables de entorno)</p>
<p>│ ├── db/migration/ migraciones Flyway V1–V23</p>
<p>│ └── reports/ plantillas .jrxml y logo</p>
<p>├── finanzas-frontend/ Angular 20 · Tailwind 4</p>
<p>│ ├── vercel.json rewrites /api y fallback SPA</p>
<p>│ └── src/app/</p>
<p>│ ├── core/ guards · interceptors · servicios</p>
<p>│ ├── features/ 51 módulos (carga diferida)</p>
<p>│ ├── shared/ componentes, layout, validadores</p>
<p>│ └── environments/ environment.ts · environment.prod.ts</p>
<p>├── deploy/ docker-compose.yml · Caddyfile ·</p>
<p>│ .env.example · generar-llaves.sh</p>
<p>└── notificaciones-service/ sin código fuente (ver 1.5)</p></td>
</tr>
</tbody>
</table>

### 3.2 Patrones y buenas prácticas aplicadas

| **Práctica**                              | **Dónde se aplica**                                           | **Beneficio**                                                                   |
|-------------------------------------------|---------------------------------------------------------------|---------------------------------------------------------------------------------|
| Arquitectura en capas                     | Controller → Service → Repository en los 61 módulos           | Responsabilidades separadas; el negocio se prueba sin HTTP                      |
| Interfaz + implementación                 | VentaService / VentaServiceImpl y equivalentes (carpeta impl) | Desacopla contrato de implementación; facilita pruebas con simulaciones         |
| DTO                                       | 157 clases de petición y respuesta                            | La API no expone las entidades; el contrato puede evolucionar sin tocar la base |
| Inyección de dependencias por constructor | @RequiredArgsConstructor en servicios y controladores         | Dependencias explícitas, inmutables y fáciles de reemplazar                     |
| Respuesta uniforme                        | ApiResponse\<T\> con success, message y data                  | El frontend maneja todas las respuestas con el mismo formato                    |
| Manejo centralizado de errores            | GlobalExceptionHandler (@RestControllerAdvice)                | Cero try/catch repetidos en controladores; códigos HTTP coherentes              |
| Outbox + LISTEN/NOTIFY                    | cola_eventos + PostgresEventListenerService                   | Eventos no se pierden aunque falle la notificación en vivo                      |
| Bloqueo pesimista                         | Correlativo de comprobantes                                   | Numeración sin duplicados con varios cajeros a la vez                           |
| Builder y Lombok                          | Entidades y DTO                                               | Menos código repetitivo, construcción legible                                   |
| Configuración externa                     | application.properties con variables de entorno               | Mismo artefacto en desarrollo y producción; secretos fuera del código           |
| Componentes reutilizables                 | shared/components en Angular (tabla, paginación, modales)     | Interfaz consistente y menos duplicación                                        |

### 3.3 Modularidad y desacoplamiento

- **Paquete por dominio:** ventas, caja, compras y seguridad viven en paquetes distintos y se comunican por servicios, no por tablas compartidas. Ejemplo: VentaServiceImpl registra el ingreso a caja llamando a CajaChicaService, sin tocar sus repositorios.

- **Contratos estables:** los controladores solo conocen interfaces de servicio y DTO; cambiar la implementación o la base no los afecta.

- **Seguridad como capa aparte:** filtros e interceptores (JWT, auditoría, contexto de sucursal) se aplican sin que el código de negocio los invoque.

- **Frontend desacoplado del backend:** solo conoce la URL base de la API y del WebSocket (environment); en producción la API se alcanza por una ruta relativa /api que Vercel reenvía al servidor.

Punto de mejora: algunos servicios concentran demasiadas responsabilidades; VentaServiceImpl tiene 973 líneas (cálculo, correlativo, caja, autorizaciones y anulación). El siguiente refactor recomendado es extraer el cálculo de importes y la integración con caja a componentes propios.

### 3.4 Manejo de errores y validaciones

Las validaciones están en tres niveles, de modo que un dato incorrecto se detenga lo antes posible:

| **Nivel**     | **Mecanismo**                                                                                                                      | **Evidencia**                                          |
|---------------|------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------|
| Interfaz      | Formularios reactivos, validadores propios (unique-id.validator) y mensajes con SweetAlert2                                        | shared/validators y componentes de formulario          |
| API           | Bean Validation en los DTO: @NotNull (67), @NotBlank (58), @Size (42), @DecimalMin (20), @Pattern, @Email; @Valid en 64 parámetros | Respuesta 400 con el detalle por campo                 |
| Base de datos | FK, CHECK y UNIQUE (Entregable 2)                                                                                                  | Rechazo aun si alguien escribe directamente en la base |

| **Excepción**                                | **Código HTTP** | **Mensaje al usuario**                                    |
|----------------------------------------------|-----------------|-----------------------------------------------------------|
| MethodArgumentNotValid / ConstraintViolation | 400             | Error de validación con el detalle por campo              |
| BadRequestException / reglas de negocio      | 400             | Texto específico (p. ej., «No tienes una caja abierta…»)  |
| AuthenticationException                      | 401             | Error de autenticación                                    |
| ExpiredJwtException                          | 401             | Su sesión ha expirado, por favor inicie sesión nuevamente |
| AccessDenied / UnauthorizedException         | 403             | No tienes permisos para acceder a este recurso            |
| ResourceNotFoundException                    | 404             | Recurso no encontrado                                     |
| Exception (no controlada)                    | 500             | Se registra con la causa raíz en el log                   |

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<tbody>
<tr class="odd">
<td><p>@RestControllerAdvice</p>
<p>public class GlobalExceptionHandler {</p>
<p>@ExceptionHandler(MethodArgumentNotValidException.class)</p>
<p>public ResponseEntity&lt;ApiResponse&lt;Map&lt;String,String&gt;&gt;&gt; handleValidationErrors(...) {</p>
<p>// arma {campo: mensaje} y responde 400 con success=false</p>
<p>}</p>
<p>@ExceptionHandler(io.jsonwebtoken.ExpiredJwtException.class) // 401</p>
<p>@ExceptionHandler({ AccessDeniedException.class, UnauthorizedException.class }) // 403</p>
<p>@ExceptionHandler(Exception.class) // 500: registra el error y busca la causa raíz</p>
<p>}</p></td>
</tr>
</tbody>
</table>

### 3.5 Calidad, pruebas y deuda técnica

La revisión del código encontró aspectos sólidos y otros que se declaran abiertamente para corregirlos. Las dos primeras filas son de seguridad y deben resolverse antes de publicar el sistema en internet.

| **Hallazgo**                                                                               | **Riesgo**                                                                       | **Acción recomendada**                                                                                   |
|--------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| El endpoint público POST /api/auth/register acepta una lista de roles en la petición       | Alto: cualquier persona podría crearse una cuenta con rol administrador          | Exigir autenticación y permiso de administrador, o ignorar el campo de roles                             |
| Las claves JWT (private_key.pem) están versionadas en src/main/resources/keys              | Alto: con la clave privada se pueden falsificar tokens si algún día se reutiliza | Quitarlas del repositorio y de su historial; el despliegue ya usa claves nuevas generadas en el servidor |
| Credenciales de una base local escritas en CheckDb.java                                    | Medio                                                                            | Eliminar el archivo y cambiar la contraseña                                                              |
| /api/test-notifications/\*\* es público                                                    | Medio                                                                            | Retirarlo o protegerlo en producción                                                                     |
| Solo 3 archivos de prueba (carga del contexto, Jasper y liquidación de seguros)            | Medio: sin red de seguridad para refactorizar                                    | Agregar pruebas de VentaService, caja y autorizaciones                                                   |
| Flyway desactivado; el esquema lo crea Hibernate                                           | Medio: sin historial de cambios de base                                          | Usar el pg_dump -s como migración base y activar Flyway                                                  |
| Carpeta notificaciones-service sin código; propiedades Twilio sin uso                      | Bajo: confusión                                                                  | Eliminarlas del repositorio                                                                              |
| AuditInterceptor guarda en la misma transacción de la petición y confía en X-Forwarded-For | Bajo                                                                             | Guardar de forma asíncrona y fijar el proxy confiable (Caddy)                                            |

### 3.6 Desempeño y escalabilidad

Desempeño. En base de datos, el Entregable 2 midió el efecto de los índices (de 15,5 ms a 2,0 ms en una consulta de reporte sobre 400 000 filas). En el backend, el despliegue fija la memoria de la JVM (Xms 512 MB, Xmx 2 GB, recolector G1), usa paginación (Pageable) en los listados y consultas de solo lectura marcadas readOnly. En el frontend, la carga diferida de rutas reduce el peso inicial. Todavía no se ha ejecutado una prueba de carga del backend; se propone una con 20 usuarios simultáneos sobre login, listado y registro de ventas (herramienta k6 o JMeter) y se anota como pendiente.

Escalabilidad. El backend no guarda estado de usuario en memoria (JWT firmado con RS256), por lo que se puede ejecutar en varias instancias detrás de un balanceador. Para que eso funcione completamente quedan tres ajustes: (1) reemplazar el broker WebSocket en memoria (enableSimpleBroker) por un broker externo, (2) mover los archivos subidos del volumen local a almacenamiento compartido y (3) separar la lectura de reportes hacia una réplica de PostgreSQL. Mientras tanto, la capacidad se aumenta verticalmente (más CPU y memoria en la VM).

## 4. Despliegue e instalación

### 4.1 Requisitos

| **Componente**          | **Versión**          | **Uso**                                |
|-------------------------|----------------------|----------------------------------------|
| Java (JDK)              | 21                   | Compilar y ejecutar el backend         |
| Maven                   | 3.9 (incluye ./mvnw) | Construir el backend                   |
| Node.js y npm           | Node 20 o superior   | Construir el frontend (Angular CLI 20) |
| PostgreSQL              | 16                   | Base de datos laboratorio_db           |
| Docker y Docker Compose | Versión reciente     | Despliegue en servidor                 |

### 4.2 Instalación en entorno local

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<tbody>
<tr class="odd">
<td><p># 1. Base de datos</p>
<p>createdb -U postgres laboratorio_db</p>
<p># 2. Backend (puerto 8084; crea las tablas al arrancar)</p>
<p>cd finanzas-backend</p>
<p>./mvnw spring-boot:run</p>
<p># Variables opcionales: DB_URL, DB_USER, DB_PASSWORD</p>
<p># Documentación de la API: http://localhost:8084/swagger-ui.html</p>
<p># 3. Frontend (puerto 4200)</p>
<p>cd ../finanzas-frontend</p>
<p>npm install</p>
<p>npm start</p></td>
</tr>
</tbody>
</table>

### 4.3 Configuración requerida

| **Variable**                               | **Ejemplo**                              | **Descripción**                                                                                 |
|--------------------------------------------|------------------------------------------|-------------------------------------------------------------------------------------------------|
| API_DOMAIN                                 | miapi.duckdns.org                        | Dominio público del backend (Caddy emite el certificado)                                        |
| DB_PASSWORD                                | (clave larga)                            | Contraseña del usuario finanzas de PostgreSQL                                                   |
| CORS_ORIGINS                               | https://mi-app.vercel.app                | Origen del frontend autorizado                                                                  |
| DB_URL / DB_USER                           | jdbc:postgresql://db:5432/laboratorio_db | Conexión a la base (definidas en docker-compose.yml)                                            |
| JWT_PRIVATE_KEY_PATH / JWT_PUBLIC_KEY_PATH | file:/keys/private_key.pem               | Claves RS256 generadas con generar-llaves.sh (no se versionan: deploy/keys/ está en .gitignore) |
| FILES_DIR                                  | /var/laboratorio/archivos                | Carpeta de archivos subidos (volumen uploads)                                                   |

### 4.4 Despliegue en producción

Arquitectura objetivo: el frontend en Vercel y el backend con la base de datos y el proxy en una máquina virtual con Docker Compose. Se eligió esta combinación porque el costo es cero para el periodo de evaluación y porque el servidor tiene memoria suficiente (hasta 24 GB en el nivel gratuito de Oracle Cloud) para la JVM y PostgreSQL sin lentitud.

| **Paso** | **Acción**                                                                                                                      |
|----------|---------------------------------------------------------------------------------------------------------------------------------|
| 1        | Crear la VM (Ubuntu, Ampere A1), abrir los puertos 80 y 443 e instalar Docker                                                   |
| 2        | Registrar un subdominio gratuito en DuckDNS apuntando a la IP pública de la VM                                                  |
| 3        | git clone del repositorio; en deploy/ ejecutar ./generar-llaves.sh y copiar .env.example a .env con los valores reales          |
| 4        | docker compose up -d --build (construye el backend y levanta db, backend y caddy)                                               |
| 5        | En finanzas-frontend/vercel.json y environment.prod.ts reemplazar CAMBIA-ESTO.duckdns.org por el dominio real y subir el cambio |
| 6        | Importar el repositorio en Vercel con Root Directory finanzas-frontend; Vercel compila y publica con HTTPS                      |

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<tbody>
<tr class="odd">
<td><p>services:</p>
<p>db: image: postgres:16 # volumen pgdata, healthcheck pg_isready</p>
<p>backend: build: ../finanzas-backend # espera a db saludable; JVM -Xmx2g</p>
<p># DB_URL, JWT_*_PATH, CORS_ORIGINS por variables de entorno</p>
<p>caddy: image: caddy:2 # puertos 80/443; reverse_proxy backend:8084</p></td>
</tr>
</tbody>
</table>

### 4.5 Verificación posterior al despliegue

- GET https://\<dominio\>/api/auth/health responde 200.

- Swagger UI abre en https://\<dominio\>/swagger-ui.html.

- El inicio de sesión desde la URL de Vercel funciona sin error de CORS.

- Se registra una venta de prueba, se imprime el PDF y la caja refleja el ingreso.

- Las notificaciones llegan por WebSocket (wss://\<dominio\>/ws-notifications-native).

- docker compose ps muestra los tres servicios activos y el respaldo de la base se ejecuta (Entregable 2, sección 4.4).

### 4.6 Evidencia de despliegue

Material ya disponible: el sistema corriendo en entorno local (figuras 5 a 12) y los archivos de despliegue versionados en el repositorio (carpeta deploy y Dockerfile). Pendiente de completar cuando la máquina virtual esté activa:

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<tbody>
<tr class="odd">
<td><p><strong>[ Captura: docker compose ps con los tres servicios activos ]</strong></p>
<p><em>Pegar la salida del comando en el servidor</em></p></td>
</tr>
</tbody>
</table>

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<tbody>
<tr class="odd">
<td><p><strong>[ Captura: sistema abierto desde la URL pública de Vercel ]</strong></p>
<p><em>Con la barra de direcciones visible</em></p></td>
</tr>
</tbody>
</table>

## Anexos

### Anexo A. Repositorio fuente

| **Dato**                                 | **Valor**                                                           |
|------------------------------------------|---------------------------------------------------------------------|
| Repositorio                              | https://github.com/Omar-Condori/ERP-CLINICO                         |
| Último commit al preparar este documento | 022d52d — 17/11/2026 — Actualización de finanzas: cambios recientes |
| Primer commit del repositorio            | 15/01/2026 (115 commits en total)                                   |
| Ramas y estructura                       | Ver sección 3.1                                                     |

### Anexo B. Manual técnico breve

| **Tarea**                      | **Cómo hacerlo**                                                                                                                                     |
|--------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------|
| Arrancar todo en el servidor   | cd deploy && docker compose up -d --build                                                                                                            |
| Ver el estado                  | docker compose ps · docker compose logs -f backend                                                                                                   |
| Actualizar a una nueva versión | git pull && docker compose up -d --build backend                                                                                                     |
| Reiniciar el backend           | docker compose restart backend                                                                                                                       |
| Respaldo de la base            | docker compose exec db pg_dump -U finanzas -Fc laboratorio_db \> respaldo.dump                                                                       |
| Compilar el frontend           | cd finanzas-frontend && npm run build (salida en dist/laboratorio/browser)                                                                           |
| Probar la API                  | Swagger UI: /swagger-ui.html; el botón Authorize recibe el JWT de /api/auth/login                                                                    |
| Agregar un nuevo módulo        | Crear paquete \<dominio\> con controller, service(+impl), repository, entity y dto; en Angular, una carpeta en features y una ruta con loadComponent |

### Anexo C. Capturas de funcionamiento

Las capturas están en la sección 2: venta (figura 5), usuarios (6), dashboard (7), catálogo (8), niveles (9), caja general (10), honorarios (11) y bitácora (12). Para la sustentación se recomienda reemplazar o complementar con capturas del sistema desplegado, con fecha visible.

### Anexo D. Evidencia de despliegue

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<tbody>
<tr class="odd">
<td><p><strong>[ Evidencia de despliegue ]</strong></p>
<p><em>URL pública, fecha y capturas de docker compose ps, de la salud de la API y del sistema en Vercel</em></p></td>
</tr>
</tbody>
</table>

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

#### Guía de preguntas probables para la defensa técnica

| **Pregunta**                                                        | **Respuesta breve apoyada en este documento**                                                                                                                       |
|---------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| ¿Por qué un monolito y no microservicios?                           | Equipo de una persona, una empresa y transacciones entre ventas, caja e inventario que deben ser atómicas; los paquetes por dominio permiten separar después (1.6). |
| ¿Cómo evitan que dos cajeros emitan el mismo número de comprobante? | Bloqueo pesimista (SELECT FOR UPDATE) sobre la serie y restricción única en base de datos (2.3 y Entregable 2).                                                     |
| ¿Cómo protegen la API?                                              | JWT RS256 de 30 min, permisos por endpoint (173 @PreAuthorize), bloqueo tras 5 intentos, sesiones limitadas y auditoría de cada escritura (2.2, 2.5).               |
| ¿Qué pasa si falla un paso al cobrar?                               | Toda la operación es una transacción: se revierte completa y el usuario recibe un mensaje claro (2.3, 3.4).                                                         |
| ¿Escala el sistema?                                                 | El backend es sin estado y puede replicarse; faltan broker WebSocket externo y almacenamiento compartido (3.6).                                                     |
| ¿Qué mejorarían?                                                    | Cerrar los hallazgos de seguridad, activar Flyway, ampliar las pruebas automáticas y refactorizar VentaServiceImpl (3.5).                                           |
