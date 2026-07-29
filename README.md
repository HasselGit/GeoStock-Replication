# GeoStock (GeoMiel) - Especificaciones Técnicas y de Diseño

Este repositorio contiene el informe técnico de análisis, el esquema lógico completo de base de datos y la guía de desarrollo detallada para replicar el sistema de trazabilidad de tambores de miel **GeoStock** (referenciado comercialmente como **GeoMiel**).

---

## 1. Esquema Completo de la Base de Datos

El sistema utiliza un modelo relacional de base de datos robusto (PostgreSQL) mapeado con Prisma ORM.

El esquema de base de datos completo en formato Prisma se encuentra en [schema.prisma](backend/prisma/schema.prisma). A continuación se detalla su estructura lógica:

### Módulo: Control de Accesos y Usuarios
*   **`User`**: Id (`cuid`), email (ej. `hespinosa`), rol (`ADMIN`, `CLIENT`, `VIEWER`), y módulos habilitados (`enabledModules`, ej. `["apicultores", "dashboard", "tambores", "estivas", "scan", "recepciones", "senasa", "laboratorio", "compra", "almacenes", "logistica"]`).
*   **`idGeomielObligatorio`**: Flag/Bandera de sesión que determina que el ingreso del código interno de Geomiel en tambores es obligatorio.

### Módulo: Proveedores y Padrón
*   **`Apicultor`**: Cuit (único, 11 caracteres), nombre completo, código apicultor (`codigoAp`, ej. `AP00031`), DNI, código postal, localidad, provincia, dirección, email, teléfono, status interno (`PENDIENTE_VALIDACION`, `VALIDADO`), número de RENAPA, ID de solicitud de RENAPA, y auditorías de consulta de RENAPA.
*   **`RenapaConsulta`**: Historial de las consultas oficiales realizadas al padrón nacional para validar la vigencia de los apicultores.

### Módulo: Recepciones e Ingresos
*   **`Recepcion`**: ID (`cuid`), número autogenerado (ej. `R-2026-0004`), número de remito, fecha de ingreso, transportista, origen, persona que entrega, observaciones, status (`BORRADOR`, `PENDIENTE`, `FINALIZADA`), tambores esperados y disponibilidad del remito firmado digitalmente (PDF).
*   **`Alert`**: Registro de control de calidad y vencimientos (ej. `RENAPA_VENCIDO`, `PARAMETRO_FUERA_DE_RANGO`), con estados `ACTIVA` o `RESUELTA`.

### Módulo: Tambores y Ubicación Física
*   **`Barrel` (Tambor)**: Código de barra interno (ej. `G-2026-000029`), código SENASA, altura, producto, origen, lote, número de tambor, pesos (`grossWeight`, `netWeight`, `tara`), número de romaneo, calidad clasificada (`calidad`, ej. `CLARA`), parámetros químicos (`color` en mm Pfund, `humedad`, `hmf`), remuestreos y relaciones de estiba.
*   **`RangoColor`**: Rangos comerciales oficiales de coloración (ej. `0-15`, `41-45`, `+70`), conteniendo límites específicos (`min` y `max`).
*   **`Estiva`**: Códigos físicos de estiba (ej. `E-080`), tipo, capacidad máxima y depósito al que pertenece.
*   **`EstivaPosition`**: Coordenada de almacenamiento físico exacta por nivel (`level`) y columna (`position`) agrupadas en parejas mediante un UUID `pairId` (regla de apilamiento en pares). Contiene fechas de ingreso (`placedAt`) y retiro (`removedAt`).
*   **`Warehouse` (`Almacen`)**: Depósitos de la empresa. Contiene código (`PI`), nombre (ej. `Parque Industrial`), tipo (`FISICO`, `VIRTUAL`, `TERCERO`), ubicación y si está activo.

### Módulo: Compras y Facturación
*   **`Emisor`**: Proveedor fiscal de la orden de compra. Contiene CUIT, razón social, estado en ARCA (`inscriptoArca`), número de RENAPA, ID en Finnegans y datos de contacto.
*   **`OrdenCompra`**: Código (`OC-2024024`), workflow (`W1`, `W2`), tipo (`MIEL`), estado (`BORRADOR`, `LIQUIDADA`, `CANCELADA`), precios estipulados por calidad (`precioClara`, `precioIntermedia`, `precioOscura`), tara teórica por tambor, kg, comisionista y observaciones.
*   **`Factura`**: Involucra el número de factura, tipo (`UNICA`), importe, kg, tambores facturados, y campos de control de integración con Finnegans (`finnegansEstado`, `finnegansId`, `finnegansError`).

### Módulo: Almacenes e Inventarios (Actualizado)
*   **`Referencia`**: Catálogo de insumos o materiales de stock (ej. `R-009` para "Cera Operculo", unidad `kg`).
*   **`Movimiento`**: Transacciones de inventario en depósitos (tipo `INGRESO`, `EGRESO`, `AJUSTE`, cantidad, fecha y observaciones).
*   **`EnvaseCera`** (Nuevo): Registro de envases de cera procesados en depósitos (código, tipo `PALLET`/`BOLSON`, pesoBruto, tara, pesoNeto, estado `EN_DEPOSITO`/`EXPORTADO`, fecha y observaciones).

### Módulo: Logística y Traslados
*   **`Chofer`**: Conductor del camión (nombre, CUIT, licencia, teléfono y activo).
*   **`SolicitudLogistica`**: Pedido de traslado (número, fecha, tipo de carga, origen, destino y estado).
*   **`Carga`**: Precinto y patente del camión (patente, acoplado, fecha y estado).
*   **`Viaje`**: Planificación de viaje que asocia chofer, carga y múltiples solicitudes logísticas.
*   **`Remito`**: Remisión de transporte oficial.

### Módulo: Laboratorio y Canastos (Actualizado)
*   **`LaboratorioOrden`**: Órdenes de análisis clínico de calidad.
*   **`LaboratorioLote`**: Lotes agrupados para análisis de miel.
*   **`LaboratorioMuestra`**: Parámetros de laboratorio del tambor (color, humedad, HMF, aprobado, fecha) asociado opcionalmente a un canasto.
*   **`Canasto`** (Nuevo): Agrupación física de muestras clínicas (canastos/racks de ensayo) rotulado con un código predictivo sugerido por el sistema (`sugerirRotuloCanasto`).

---

## 2. Contratos de la API y Flujos de Integración

### A. Impresión Física de Etiquetas (ZPL)
El backend cuenta con el endpoint `GET /api/barrels/:id/etiqueta` que devuelve una cadena en formato **ZPL** (Zebra Programming Language) lista para impresoras térmicas Zebra, la cual dibuja los códigos de barra y detalles de pesaje del tambor.

### B. Módulo de Cera y Canastos
*   **Cera**: `/api/almacenes/envases-cera` (POST para crear y GET para listar con cálculos consolidados de cantidades y kilos de bolsones/pallets), `/api/almacenes/envases-cera/:id/exportar` (POST).
*   **Canastos**: `/api/laboratorio/canastos` (GET, POST), `/api/laboratorio/canastos/:id` (GET, PATCH), `/api/laboratorio/canastos/:id/archivar` (POST), y `/api/laboratorio/canastos/:id/muestras/:muestraId` (DELETE).

### C. Lógica de Pesaje de Tambores
El flujo de recepción incluye `/api/barrels/pesaje/buscar?code=...` para buscar tambores ingresados y `POST /api/barrels/pesaje/:id/confirmar` para asentar el peso neto y bruto final.

---

## 3. Guía de Desarrollo para Agentes de IA

### Estructura de Rutas API a Implementar
1.  **Auth**: `/api/auth/login`, `/api/auth/logout`, `/api/auth/me`.
2.  **Compras**: `/api/compra/emisores`, `/api/compra/ordenes` (GET, POST, PATCH), `/api/compra/ordenes/:id/tambores` (POST, DELETE), `/api/compra/tambores/:id/calidad` (PATCH), `/api/compra/ordenes/:id/facturas` (POST, DELETE), `/api/compra/ordenes/:id/finnegans/push` (POST).
3.  **Almacenes**: `/api/almacenes`, `/api/almacenes/referencias` (GET, POST, PATCH), `/api/almacenes/existencias` (GET), `/api/almacenes/referencias/:id/movimientos` (GET), `/api/almacenes/movimientos` (POST), `/api/almacenes/envases-cera` (GET, POST), `/api/almacenes/envases-cera/:id/exportar` (POST), `/api/almacenes/envases-cera/:id` (DELETE).
4.  **Logística**: `/api/logistica/choferes`, `/api/logistica/solicitudes`, `/api/logistica/cargas`, `/api/logistica/viajes` (GET, POST, PATCH), `/api/logistica/viajes/:id/solicitudes` (POST, DELETE), `/api/logistica/remitos`.
5.  **Laboratorio**: `/api/laboratorio/ordenes`, `/api/laboratorio/iniciar` (POST), `/api/laboratorio/muestras/:id` (PATCH), `/api/laboratorio/ordenes/:id/validar-muestra` (POST), `/api/laboratorio/canastos` (GET, POST), `/api/laboratorio/canastos/:id` (GET, PATCH, DELETE, POST `/archivar`).
