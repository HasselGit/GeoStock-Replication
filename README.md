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
*   **`OrdenCompra`**: Código (`OC-2024024`), workflow (`W1`, `W2`), tipo (`MIEL`), estado (`BORRADOR`, `LIQUIDADA`, `CANCELADA`), precios estipulados por calidad (`precioClara`, `precioIntermedia`, `precioOscura`), tara teórica por tambor, kg, comisionista y observaciones. Permite borrado explícito (`DELETE /compra/ordenes/:id`).
*   **`Factura`**: Involucra el número de factura, tipo (`UNICA`), importe, kg, tambores facturados, y campos de control de integración con Finnegans (`finnegansEstado`, `finnegansId`, `finnegansError`).

### Módulo: Almacenes e Inventarios
*   **`Referencia`**: Catálogo de insumos o materiales de stock (ej. `R-009` para "Cera Operculo", unidad `kg`).
*   **`Movimiento`**: Transacciones de inventario en depósitos (tipo `INGRESO`, `EGRESO`, `AJUSTE`, cantidad, fecha y observaciones).
*   **`EnvaseCera`**: Registro de envases de cera procesados en depósitos (código, tipo `PALLET`/`BOLSON`, pesoBruto, tara, pesoNeto, estado `EN_DEPOSITO`/`EXPORTADO`, fecha y observaciones).

### Módulo: Logística, Reservas de Stock y Remitos (Actualizado)
*   **`Chofer`**: Conductor del camión (nombre, CUIT, licencia, teléfono y activo).
*   **`SolicitudLogistica`**: Pedido de traslado (número, fecha, tipo de carga, origen, destino y estado).
*   **`Carga`**: Precinto y patente del camión (patente, acoplado, fecha y estado).
*   **`Viaje`**: Planificación de viaje que asocia chofer, carga y múltiples solicitudes logísticas.
*   **`Remito`**: Remisión de transporte oficial con soporte para firma digital (`firmaUrl`, `firmadoAt`, `firmadoPor`).
*   **`ReservaStock`** (Nuevo): Control de apartado de existencias previas al despacho (`referenciaId`, `warehouseId`, `cantidad`, `estado` `ACTIVA`/`CONSUMIDA`/`CANCELADA`, opcionalmente asociada a una `SolicitudLogistica`).

### Módulo: Laboratorio y Canastos
*   **`LaboratorioOrden`**: Órdenes de análisis clínico de calidad.
*   **`LaboratorioLote`**: Lotes agrupados para análisis de miel.
*   **`LaboratorioMuestra`**: Parámetros de laboratorio del tambor (color, humedad, HMF, aprobado, fecha) asociado opcionalmente a un canasto.
*   **`Canasto`**: Agrupación física de muestras clínicas (canastos/racks de ensayo) rotulado con un código predictivo sugerido por el sistema (`sugerirRotuloCanasto`).

---

## 2. Contratos de la API y Flujos de Integración

### A. Reservas de Stock y Cálculo de Disponible (Nuevo 30 Julio)
*   `GET /api/logistica/reservas`: Listado de reservas activas de inventario.
*   `POST /api/logistica/reservas`: `{ referenciaId, warehouseId, cantidad, solicitudId, observaciones }` $\rightarrow$ Registra el apartado de stock.
*   `POST /api/logistica/reservas/:id/consumir`: Descuenta físicamente la reserva al concretar la carga.
*   `POST /api/logistica/reservas/:id/cancelar`: Libera el stock apartado.
*   `GET /api/logistica/stock-disponible?referenciaId=...&warehouseId=...`: Retorna la fórmula de cálculo del inventario usable real: `{ total, reservado, enCarga, comprometido, disponible }`.

### B. Firma Digital y Contexto de Remito (Nuevo 30 Julio)
*   `GET /api/logistica/viajes/:id/contexto-remito`: Datos consolidados para confeccionar la documentación del viaje.
*   `POST /api/logistica/remitos/:id/firma`: `{ firmaUrl, firmadoPor }` $\rightarrow$ Registra la conformidad de entrega.

### C. Impresión Física de Etiquetas (ZPL)
El backend cuenta con el endpoint `GET /api/barrels/:id/etiqueta` que devuelve una cadena en formato **ZPL** (Zebra Programming Language) lista para impresoras térmicas Zebra.

### D. Módulo de Cera y Canastos
*   **Cera**: `/api/almacenes/envases-cera` (POST, GET), `/api/almacenes/envases-cera/:id/exportar` (POST).
*   **Canastos**: `/api/laboratorio/canastos` (GET, POST), `/api/laboratorio/canastos/:id` (GET, PATCH), `/api/laboratorio/canastos/:id/archivar` (POST).

---

## 3. Guía de Desarrollo para Agentes de IA

### Estructura de Rutas API a Implementar
1.  **Auth**: `/api/auth/login`, `/api/auth/logout`, `/api/auth/me`.
2.  **Compras**: `/api/compra/emisores`, `/api/compra/ordenes` (GET, POST, PATCH, DELETE), `/api/compra/ordenes/:id/tambores` (POST, DELETE), `/api/compra/tambores/:id/calidad` (PATCH), `/api/compra/ordenes/:id/facturas` (POST, DELETE), `/api/compra/ordenes/:id/finnegans/push` (POST).
3.  **Almacenes**: `/api/almacenes`, `/api/almacenes/referencias` (GET, POST, PATCH), `/api/almacenes/existencias` (GET), `/api/almacenes/referencias/:id/movimientos` (GET), `/api/almacenes/movimientos` (POST), `/api/almacenes/envases-cera` (GET, POST), `/api/almacenes/envases-cera/:id/exportar` (POST).
4.  **Logística y Reservas**: `/api/logistica/choferes`, `/api/logistica/solicitudes` (GET, POST, DELETE), `/api/logistica/cargas`, `/api/logistica/viajes` (GET, POST, PATCH), `/api/logistica/viajes/:id/solicitudes` (POST, DELETE), `/api/logistica/remitos` (GET, POST, DELETE, POST `/firma`), `/api/logistica/reservas` (GET, POST, POST `/consumir`, POST `/cancelar`), `/api/logistica/stock-disponible` (GET).
5.  **Laboratorio**: `/api/laboratorio/ordenes`, `/api/laboratorio/iniciar` (POST), `/api/laboratorio/muestras/:id` (PATCH), `/api/laboratorio/ordenes/:id/validar-muestra` (POST), `/api/laboratorio/canastos` (GET, POST), `/api/laboratorio/canastos/:id` (GET, PATCH, DELETE, POST `/archivar`).
