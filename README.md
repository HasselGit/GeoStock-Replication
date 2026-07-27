# GeoStock (GeoMiel) - Especificaciones Técnicas y de Diseño

Este repositorio contiene el informe técnico de análisis, el esquema lógico completo de base de datos y la guía de desarrollo detallada para replicar el sistema de trazabilidad de tambores de miel **GeoStock** (referenciado comercialmente como **GeoMiel**).

---

## 1. Esquema Completo de la Base de Datos

A partir del análisis detallado de los módulos habilitados y las llamadas de la API en el cliente (`/api/auth/me`, `/api/apicultores`, `/api/renapa/*`, `/api/barrels/*`, `/api/estivas/*`, `/api/recepciones/*`, `/api/compra/*`, `/api/almacenes/*`, `/api/logistica/*`, `/api/laboratorio/*`, `/api/alerts/*`), se deduce que el sistema utiliza un modelo relacional robusto mapeado con Prisma ORM.

El esquema de base de datos completo en formato Prisma se encuentra en [schema.prisma](backend/prisma/schema.prisma). A continuación se detalla su estructura lógica:

### Módulo: Control de Accesos y Usuarios
*   **`User`**: Id (`cuid`), email (ej. `hespinosa`), rol (`ADMIN`, `CLIENT`, `VIEWER`), y módulos habilitados (`enabledModules`, ej. `["dashboard", "compra", "apicultores", "recepciones", "tambores", "almacenes"]`).

### Módulo: Proveedores y Padrón
*   **`Apicultor`**: Cuit (único, 11 caracteres), nombre completo, código apicultor (`codigoAp`, ej. `AP00031`), DNI, código postal, localidad, provincia, dirección, email, teléfono, status interno (`PENDIENTE_VALIDACION`, `VALIDADO`), número de RENAPA, ID de solicitud de RENAPA, y auditorías de consulta de RENAPA.
*   **`RenapaConsulta`**: Historial de las consultas oficiales realizadas al padrón nacional para validar la vigencia de los apicultores.

### Módulo: Recepciones e Ingresos
*   **`Recepcion`**: ID (`cuid`), número autogenerado (ej. `R-2026-0004`), número de remito, fecha de ingreso, transportista, origen, persona que entrega, observaciones, status (`BORRADOR`, `PENDIENTE`, `FINALIZADA`), tambores esperados y disponibilidad del remito firmado digitalmente (PDF).
*   **`Alert`**: Registro de control de calidad y vencimientos (ej. `RENAPA_VENCIDO`, `PARAMETRO_FUERA_DE_RANGO`), con estados `ACTIVA` o `RESUELTA`.

### Módulo: Tambores y Ubicación Física
*   **`Barrel` (Tambor)**: Código de barra interno (ej. `G-2026-000029`), código SENASA, altura, producto, origen, lote, número de tambor, pesos (`grossWeight`, `netWeight`, `tara`), número de romaneo, calidad clasificada (`calidad`, ej. `CLARA`), parámetros químicos (`color` en mm Pfund, `humedad`, `hmf`), remuestreos y relaciones de estiba.
*   **`RangoColor`**: Rangos comerciales oficiales de coloración (ej. `41-45`, `+70`).
*   **`Estiva`**: Códigos físicos de estiba (ej. `E-080`), tipo, capacidad máxima y depósito al que pertenece.
*   **`EstivaPosition`**: Coordenada de almacenamiento físico exacta por nivel (`level`) y columna (`position`) agrupadas en parejas mediante un UUID `pairId` (regla de apilamiento en pares). Contiene fechas de ingreso (`placedAt`) y retiro (`removedAt`).
*   **`Warehouse` (`Almacen`)**: Depósitos de la empresa. Contiene código (`PI`), nombre (ej. `Parque Industrial`), tipo (`FISICO`, `VIRTUAL`, `TERCERO`), ubicación y si está activo.

### Módulo: Compras y Facturación (Nuevo)
*   **`Emisor`**: Proveedor fiscal de la orden de compra. Contiene CUIT, razón social, estado en ARCA (`inscriptoArca`), número de RENAPA, ID en Finnegans y datos de contacto.
*   **`OrdenCompra`**: Código (`OC-2024024`), workflow (`W1`, `W2`), tipo (`MIEL`), estado (`BORRADOR`, `LIQUIDADA`, `CANCELADA`), precios estipulados por calidad (`precioClara`, `precioIntermedia`, `precioOscura`), tara teórica por tambor, kg, comisionista y observaciones.
*   **`Factura`**: Involucra el número de factura, tipo (`UNICA`), importe, kg, tambores facturados, y campos de control de integración con Finnegans (`finnegansEstado`, `finnegansId`, `finnegansError`).

### Módulo: Almacenes e Inventarios (Nuevo)
*   **`Referencia`**: Catálogo de insumos o materiales de stock.
*   **`Movimiento`**: Transacciones de inventario en depósitos (tipo `INGRESO`, `EGRESO`, `AJUSTE`, cantidad, fecha y observaciones).

### Módulo: Logística y Traslados (Nuevo)
*   **`Chofer`**: Conductor del camión (nombre, CUIT, licencia, teléfono y activo).
*   **`SolicitudLogistica`**: Pedido de traslado (número, fecha, tipo de carga, origen, destino y estado).
*   **`Carga`**: Precinto y patente del camión (patente, acoplado, fecha y estado).
*   **`Viaje`**: Planificación de viaje que asocia chofer, carga y múltiples solicitudes logísticas.
*   **`Remito`**: Remisión de transporte oficial.

### Módulo: Laboratorio y Control Químico (Nuevo)
*   **`LaboratorioOrden`**: Órdenes de análisis clínico de calidad.
*   **`LaboratorioLote`**: Lotes agrupados para análisis de miel.
*   **`LaboratorioMuestra`**: Parámetros de laboratorio del tambor (color, humedad, HMF, aprobado, fecha).

---

## 2. Contratos de la API y Flujos de Integración

### A. Integración con ERP Finnegans
Las órdenes de compra liquidadas se sincronizan mediante `POST /api/compra/ordenes/:id/finnegans/push`. El backend envía los detalles de facturación y asignación de tambores, y almacena las respuestas del ERP en la factura para auditoría.

### B. Liquidación Predictiva de Miel
El backend expone en `GET /api/compra/ordenes/:id` un objeto `liquidacion` dinámico. Este calcula en tiempo real los importes a pagar basándose en los kilos netos de los tambores asociados, clasificados por calidad (color en mm Pfund y humedad) y multiplicados por los precios definidos de la orden.

### C. Lógica LIFO de Escaneo por Pares (Estivas)
*   **Armado**: `/api/scan/armado` requiere `barrelCodes: [código1, código2]` para asegurar la colocación física en pares.
*   **Desarmado**: `/api/scan/desarmado` valida que no se retiren tambores de niveles inferiores si hay tambores colocados encima, y exige escanear consecutivamente el par compañero de la celda.

---

## 3. Guía de Desarrollo para Agentes de IA

### Estructura de Rutas API a Implementar
1.  **Auth**: `/api/auth/login`, `/api/auth/logout`, `/api/auth/me`.
2.  **Compras**: `/api/compra/emisores`, `/api/compra/ordenes` (GET, POST, PATCH), `/api/compra/ordenes/:id/tambores` (POST, DELETE), `/api/compra/tambores/:id/calidad` (PATCH), `/api/compra/ordenes/:id/facturas` (POST, DELETE), `/api/compra/ordenes/:id/finnegans/push` (POST).
3.  **Almacenes**: `/api/almacenes`, `/api/almacenes/referencias` (GET, POST, PATCH), `/api/almacenes/existencias` (GET), `/api/almacenes/referencias/:id/movimientos` (GET), `/api/almacenes/movimientos` (POST).
4.  **Logística**: `/api/logistica/choferes`, `/api/logistica/solicitudes`, `/api/logistica/cargas`, `/api/logistica/viajes` (GET, POST, PATCH), `/api/logistica/viajes/:id/solicitudes` (POST, DELETE), `/api/logistica/remitos`.
5.  **Laboratorio**: `/api/laboratorio/ordenes`, `/api/laboratorio/iniciar` (POST), `/api/laboratorio/muestras/:id` (PATCH), `/api/laboratorio/ordenes/:id/validar-muestra` (POST).

### UI/UX y Estilo
*   Fondo `#08090A`, tema oscuro, tipografía **Inter** para UI, y **Commit Mono** para datos numéricos y códigos de barra. Componentes headless de **Radix UI** estilizados con **Tailwind CSS**.
