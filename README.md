# GeoStock (GeoMiel) - Especificaciones Técnicas y de Diseño

Este repositorio contiene el informe técnico de análisis, el esquema lógico completo de base de datos y la guía de desarrollo detallada para replicar el sistema de trazabilidad de tambores de miel **GeoStock** (referenciado comercialmente como **GeoMiel**).

---

## 1. Esquema Completo de la Base de Datos

El sistema utiliza un modelo relacional de base de datos robusto (PostgreSQL) mapeado con Prisma ORM.

El esquema de base de datos completo en formato Prisma se encuentra en [schema.prisma](backend/prisma/schema.prisma). A continuación se detalla su estructura lógica:

### Módulo: Control de Accesos y Usuarios
*   **`User`**: Id (`cuid`), email (ej. `hespinosa`), nombre completo (`Espinosa Illas, Hassel`), área organizativa (`area`, ej. `"Stocks"`), rol (`ADMIN`, `CLIENT`, `VIEWER`), y módulos habilitados (`enabledModules`, ej. `["apicultores", "dashboard", "tambores", "estivas", "scan", "recepciones", "senasa", "laboratorio", "compra", "almacenes", "logistica"]`).
*   **`idGeomielObligatorio`**: Flag/Bandera de sesión que determina que el ingreso del código interno de Geomiel en tambores es obligatorio.

### Módulo: Proveedores y Padrón
*   **`Apicultor`**: Cuit (único, 11 caracteres), nombre completo, código apicultor (`codigoAp`, ej. `AP00031`), DNI, código postal, localidad, provincia, dirección, email, teléfono, status interno (`PENDIENTE_VALIDACION`, `VALIDADO`), número de RENAPA, ID de solicitud de RENAPA, y auditorías de consulta de RENAPA.
*   **`RenapaConsulta`**: Historial de las consultas oficiales realizadas al padrón nacional para validar la vigencia de los apicultores.

### Módulo: Recepciones e Ingresos
*   **`Recepcion`**: ID (`cuid`), número autogenerado (ej. `R-2026-0004`), número de remito, fecha de ingreso, transportista, origen, persona que entrega, observaciones, status (`BORRADOR`, `PENDIENTE`, `FINALIZADA`), tambores esperados y disponibilidad del remito firmado digitalmente (PDF).
*   **`Alert`**: Registro de control de calidad y vencimientos (ej. `RENAPA_VENCIDO`, `PARAMETRO_FUERA_DE_RANGO`), con estados `ACTIVA` o `RESUELTA`.

### Módulo: Tambores, Visor 3D y Ubicación Física (Actualizado)
*   **`Barrel` (Tambor)**: Código de barra interno (ej. `G-2026-000029`), código SENASA, altura, producto, origen, lote, número de tambor, pesos (`grossWeight`, `netWeight`, `tara`), número de romaneo, calidad clasificada (`calidad`, ej. `CLARA`), parámetros químicos (`color` en mm Pfund, `humedad`, `hmf`), remuestreos y relaciones de estiba. Soporta eliminación física (`DELETE /barrels/:id`).
*   **`Estiva`**: Códigos físicos de estiba (ej. `E39`), tipo (`DEFINITIVA`), accesos (`TWO_SIDES`), capacidad base (44 tambores) y 5 niveles de altura.
*   **Capacidad Piramidal Estricta**: La capacidad total decrece en 2 tambores por nivel para garantizar la estabilidad física:
    $$\text{Capacidad Nivel } s = \max(0, \text{baseCapacity} - (s - 1) \times 2)$$
    *Para `maxLevels: 5` y `baseCapacity: 44`, el total ocupado al 100% es $44 + 42 + 40 + 38 + 36 = 200 \text{ tambores}$.*
*   **Nomenclatura Humana de Posición (`[Estiba]-[Lado]-P[Piso]-U[Ubicación]`)**:
    *   `E39`: Código de Estiba.
    *   `D` / `I`: Lado Derecho (posiciones impares) / Izquierdo (posiciones pares).
    *   `P1` a `P5`: **Piso / Nivel de Elevación** (Piso 1 al 5).
    *   `U1` a `U22`: **Ubicación / Unidad** en la fila.
*   **Trabado Alternado de Pisos Pares (Zig-Zag)**: En los pisos impares (`P1`, `P3`, `P5`), la Ubicación `U` se cuenta en orden directo (`1, 2, 3...`). En los **pisos pares (`P2`, `P4`)**, la Ubicación `U` se invierte (`total - posicion + 1`) para representar el trabado físico cruzado de tambores en la pirámide (ejemplos reales: `E39-D-P1-U1`, `E39-D-P2-U21`, `E39-D-P4-U19`).

---

## 2. Contratos de la API y Flujos de Integración

### A. Reservas de Stock y Cálculo de Disponible
*   `GET /api/logistica/reservas`: Listado de reservas activas de inventario.
*   `POST /api/logistica/reservas`: `{ referenciaId, warehouseId, cantidad, solicitudId, observaciones }` $\rightarrow$ Registra el apartado de stock.
*   `GET /api/logistica/stock-disponible?referenciaId=...&warehouseId=...`: Retorna la fórmula de cálculo del inventario usable real: `{ total, reservado, enCarga, comprometido, disponible }`.

### B. Escáner de Estivas
*   `POST /api/scan/armado`: Exige recibir `barrelCodes: [código1, código2]` en pareja (`pairId`).
*   `POST /api/scan/desarmado`: Aplica LIFO estricto. Lanza los errores: `"Retirás primero tambores superiores"` y `"[Código] no es el par del primero. Escaneá el compañero."`.

---

## 3. Guía de Desarrollo para Agentes de IA

### Estructura de Rutas API a Implementar
1.  **Auth**: `/api/auth/login`, `/api/auth/logout`, `/api/auth/me`.
2.  **Compras y Padrón**: `/api/compra/emisores`, `/api/compra/padron/estado`, `/api/compra/padron/sincronizar`, `/api/compra/ordenes` (GET, POST, PATCH, DELETE), `/api/compra/ordenes/:id/tambores` (POST, DELETE), `/api/compra/tambores/:id/calidad` (PATCH), `/api/compra/ordenes/:id/facturas` (POST, DELETE), `/api/compra/ordenes/:id/finnegans/push` (POST).
3.  **Almacenes y Estivas**: `/api/almacenes`, `/api/estivas` (GET, POST, GET `/:id/structure`), `/api/scan/suggest/:id`, `/api/scan/armado` (POST), `/api/scan/desarmado` (POST).
4.  **Logística y Reservas**: `/api/logistica/choferes`, `/api/logistica/solicitudes`, `/api/logistica/cargas`, `/api/logistica/viajes`, `/api/logistica/remitos` (GET, POST, DELETE, POST `/firma`), `/api/logistica/reservas` (GET, POST, POST `/consumir`, POST `/cancelar`), `/api/logistica/stock-disponible` (GET).
5.  **Laboratorio y SENASA**: `/api/senasa/consultar` (POST), `/api/laboratorio/ordenes`, `/api/laboratorio/iniciar` (POST), `/api/laboratorio/canastos` (GET, POST, PATCH, DELETE, POST `/archivar`).
