# GeoStock (GeoMiel) - Especificaciones Técnicas y de Diseño

Este repositorio contiene el informe técnico de análisis, el esquema lógico completo de base de datos y la guía de desarrollo detallada para replicar el sistema de trazabilidad de tambores de miel **GeoStock** (referenciado comercialmente como **GeoMiel**).

---

## 1. Esquema Completo de la Base de Datos

A partir del análisis de las llamadas de la API en el cliente (`/api/auth/me`, `/api/apicultores`, `/api/renapa`, `/api/barrels`, `/api/estivas`, `/api/recepciones`, `/api/alerts`), se deduce que el sistema utiliza un modelo relacional (por ejemplo, PostgreSQL o MySQL) mapeado mediante un ORM como Prisma (debido a la estructura de campos como `_count` y la convención de IDs tipo CUID o UUID).

El esquema de base de datos completo en formato Prisma se encuentra en [schema.prisma](backend/prisma/schema.prisma). A continuación se detalla su estructura lógica:

### Tabla: `User` (Usuarios / Personal)
Representa a los usuarios del sistema.
* **`id`**: String (UUID / CUID con prefijo `usr_`) - Clave primaria.
* **`email`**: String - Email o usuario, único (ej. `hespinosa`).
* **`role`**: Enum (`ADMIN`, `CLIENT`, `VIEWER`) - Rol que determina los permisos en los módulos.
* **`enabledModules`**: Array de Strings (ej. `["dashboard", "apicultores", "recepciones", "tambores"]`).

### Tabla: `Apicultor` (Proveedores de Miel)
Almacena la información de los apicultores (proveedores) y su estado de validación con RENAPA.
* **`id`**: String (CUID) - Clave primaria.
* **`cuit`**: String (11 dígitos, único) - Identificación fiscal.
* **`nombre`**: String - Razón social o nombre completo.
* **`codigoAp`**: String (Único, opcional) - Código interno de apicultor (ej. `AP00031`).
* **`dni`**: String (Opcional)
* **`codigoPostal`**: String (Opcional)
* **`localidad`**: String (Opcional)
* **`provincia`**: String (Opcional)
* **`direccion`**: String (Opcional)
* **`email`**: String (Opcional)
* **`telefono`**: String (Opcional)
* **`status`**: Enum (`PENDIENTE_VALIDACION`, `VALIDADO`) - Estado interno en GeoStock.
* **`numeroRenapa`**: String (Opcional) - Número oficial otorgado.
* **`idSolicitudRenapa`**: Int (Opcional) - Número de solicitud de RENAPA.
* **`renapaStatus`**: Enum (`VIGENTE`, `POR_VENCER`, `VENCIDO`, `NO_ENCONTRADO`, `ERROR`) - Estado oficial de RENAPA.
* **`renapaVigenciaAt`**: DateTime (Opcional) - Fecha de vencimiento del registro RENAPA.
* **`renapaConsultadoAt`**: DateTime (Opcional) - Fecha de la última consulta en vivo.
* **`notas`**: Text (Opcional)
* **`createdAt`**: DateTime
* **`updatedAt`**: DateTime
* **`createdById`**: String (Opcional) - Clave foránea que referencia a `User(id)`.

### Tabla: `RenapaConsulta` (Historial de Consultas de RENAPA)
Historial de las consultas oficiales realizadas al padrón nacional.
* **`id`**: String (CUID) - Clave primaria.
* **`apicultorId`**: String - Clave foránea que referencia a `Apicultor(id)`.
* **`cuit`**: String
* **`status`**: String
* **`vigenciaAt`**: DateTime (Opcional)
* **`consultadoAt`**: DateTime
* **`certificadoDisponible`**: Boolean - Indica si existe un PDF oficial descargable.
* **`errorMessage`**: String (Opcional) - En caso de error en la consulta externa.

### Tabla: `RangoColor` (Clasificación de Colores de Miel)
* **`id`**: String (UUID) - Clave primaria.
* **`codigo`**: String (Único) - Código de rango (ej. "41-45", "+70").
* **`etiqueta`**: String - Etiqueta legible (ej. "41–45").

### Tabla: `Barrel` (Tambores de Miel)
Tambores que contienen la miel recolectada con sus pesos, análisis físico-químicos y ubicación.
* **`id`**: String (CUID) - Clave primaria.
* **`internalCode`**: String (Único) - Código de barra o etiqueta interna.
* **`senasaCode`**: String (Opcional) - Código oficial de SENASA.
* **`height`**: String (Opcional) - Altura del tambor (ej. "NORMAL").
* **`product`**: String (Opcional)
* **`origin`**: String (Opcional) - Origen del lote (default "Argentina").
* **`lot`**: String (Opcional) - Lote de producción.
* **`drumNumber`**: Int (Opcional)
* **`pesoOrigen`**: Float (Opcional)
* **`grossWeight`**: Float (Opcional) - Peso bruto.
* **`netWeight`**: Float (Opcional) - Peso neto.
* **`tara`**: Float (Opcional) - Peso del tambor vacío.
* **`romaneo`**: String (Opcional) - Número de romaneo.
* **`destination`**: String (Opcional)
* **`supplier`**: String (Opcional)
* **`apicultorId`**: String (Opcional) - Clave foránea que referencia a `Apicultor(id)`.
* **`recepcionId`**: String (Opcional) - Clave foránea que referencia a `Recepcion(id)`.
* **`ordenCompraId`**: String (Opcional)
* **`calidad`**: String (Opcional)
* **`color`**: Float (Opcional) - Medida de color.
* **`humedad`**: Float (Opcional) - Porcentaje de humedad.
* **`hmf`**: Float (Opcional) - Porcentaje de HMF.
* **`fechaDeposito`**: DateTime (Opcional)
* **`fechaRemuestreo`**: DateTime (Opcional)
* **`rangoColorId`**: String (Opcional) - Clave foránea que referencia a `RangoColor(id)`.
* **`clasificacion`**: String (Opcional) - Clasificación del depósito (ej. "GEO_EXPO").
* **`fechaIngresoDeposito`**: DateTime (Opcional)
* **`status`**: Enum (`DISPONIBLE`, `EN_ESTIVA`, `PROCESADO`, `DESPACHADO`)
* **`senasaStatus`**: String (Opcional)
* **`senasaConsultadoAt`**: DateTime (Opcional)
* **`createdAt`**: DateTime
* **`updatedAt`**: DateTime

### Tabla: `Estiva` (Estructuras de Apilamiento / Celdas de Almacén)
Estructuras físicas organizadas en celdas de almacenamiento donde se colocan los tambores en formato de pares (dos por nivel y posición).
* **`id`**: String (CUID) - Clave primaria.
* **`code`**: String (Único) - Identificador de la estiba (ej. E-080).
* **`type`**: String - Tipo de estiba.
* **`capacity`**: Integer - Capacidad máxima.
* **`warehouseId`**: String - Clave foránea que referencia a `Warehouse(id)`.
* **`createdAt`**: DateTime
* **`updatedAt`**: DateTime

### Tabla: `EstivaPosition` (Posición de Tambores en Estiva)
Coordenada de almacenamiento físico exacta para un tambor y su fecha de colocación y retiro.
* **`id`**: String (CUID) - Clave primaria.
* **`estivaId`**: String - Clave foránea a `Estiva(id)`.
* **`barrelId`**: String - Clave foránea a `Barrel(id)`.
* **`pairId`**: String - Agrupador UUID para los dos tambores de la misma celda.
* **`level`**: Int - Piso/Nivel (1-indexed).
* **`position`**: Int - Columna/Celda (1-indexed).
* **`placedAt`**: DateTime - Fecha de colocación.
* **`removedAt`**: DateTime (Opcional) - Fecha de retiro.

### Tabla: `Recepcion` (Ingresos de Mercadería)
Documentos de recepción de lotes de tambores de apicultores.
* **`id`**: String (CUID) - Clave primaria.
* **`apicultorId`**: String - Clave foránea que referencia a `Apicultor(id)`.
* **`status`**: Enum (`PENDIENTE`, `FINALIZADA`)
* **`pdfDisponible`**: Boolean
* **`createdAt`**: DateTime
* **`updatedAt`**: DateTime

### Tabla: `Warehouse` (Depósitos / Bodegas)
* **`id`**: String (CUID) - Clave primaria.
* **`codigo`**: String (Único) - Código abreviado (ej. "PI").
* **`name`**: String - Nombre del depósito (ej. "Parque Industrial").

### Tabla: `ScanEvent` (Auditoría de Escaneo)
Historial de eventos de escaneo.
* **`id`**: String (CUID) - Clave primaria.
* **`barrelId`**: String - Clave foránea a `Barrel(id)`.
* **`type`**: String - Tipo de evento ("INGRESO", "ARMADO", "DESARMADO").
* **`payload`**: String (Opcional) - Detalles en formato JSON.
* **`createdAt`**: DateTime

### Tabla: `ConsultaSenasa` (Auditoría de Consultas SENASA)
* **`id`**: String (CUID) - Clave primaria.
* **`barrelId`**: String - Clave foránea a `Barrel(id)`.
* **`status`**: String
* **`errorMessage`**: String (Opcional)
* **`createdAt`**: DateTime

---

## 2. Lógica de Armado y Desarmado de Estivas

El almacenamiento físico en las estivas se organiza de manera **agrupada en pares** y con fuertes restricciones de orden físico (tipo LIFO de apilado vertical).

### A. Lógica de Armado (Asociar Tambores a la Estiva)
1. **Carga y Preparación**:
   * Al seleccionar una estiba (`estivaId`), el frontend carga de manera concurrente:
     * `getEstiva(estivaId)`: Posiciones ocupadas actualmente.
     * `getEstivaStructure(estivaId)`: Niveles y columnas de la estiba.
     * `suggestPosition(estivaId)`: La API del servidor retorna de manera predictiva el siguiente nivel (`level`) y columna/posición (`position`) libre ideal donde se debería colocar el siguiente par de tambores.
2. **Ciclo de Escaneo (Pares)**:
   * El operario escanea el **primer tambor** (guardando su código interno o código SENASA). La UI valida que no sea un tambor duplicado.
   * El operario escanea el **segundo tambor** para conformar el **par** que se apilará en la misma celda de la estiba.
3. **Persistencia**:
   * Una vez completados los dos escaneos de tambor, la posición (nivel y columna) sugeridos, se llama a:
     `POST /api/scan/armado` enviando `{ estivaId, barrelCodes: [código1, código2], level, position }`.
   * Si es exitoso, el servidor asocia la relación en la base de datos de los tambores con la estiba en esa coordenada y marca el estado de los tambores como `EN_ESTIVA`.

### B. Lógica de Desarmado (Retiro de Tambores de la Estiva)
1. **Regla de Apilamiento Físico (LIFO)**:
   * No se pueden retirar tambores que estén en niveles inferiores si hay tambores colocados encima.
   * Al cargar la estiba en modo desarmado, el backend devuelve un arreglo `removableBarrelIds` con las IDs de los tambores que se encuentran en la cima o que están libres de obstrucciones superiores.
2. **Ciclo de Validación y Escaneo**:
   * **Escaneo del Tambor 1**: El sistema busca el código en un mapa de la estiba. Si el tambor escaneado no está en el listado de permitidos, la UI arroja el error: **"Retirá primero tambores superiores"**.
   * **Escaneo del Tambor 2 (Búsqueda del Compañero)**: Al escanear el primer tambor, el sistema recupera su `pairId` (el par asociado en la celda). Al escanear el segundo tambor, valida que pertenezca al mismo par. Si no es así, arroja el error: **"[Código] no es el par del primero. Escaneá el compañero."**.
3. **Persistencia**:
   * Una vez escaneado y validado el par correspondiente de la celda removible, se llama a:
     `POST /api/scan/desarmado` enviando `{ estivaId, barrelCodes: [código1, código2] }`.
   * Si es exitoso, se desvinculan los tambores de la estiba, permitiendo que los tambores del nivel inferior pasen a ser removibles en el siguiente paso.

---

## 3. Tecnologías Detectadas en el Frontend

El frontend de GeoStock cuenta con una interfaz premium, oscura, minimalista y de alto rendimiento. Las tecnologías clave identificadas son:

1. **Framework Principal**: **Next.js 14+** (App Router: utilizando Server Components y Client Components).
2. **Biblioteca de Renderizado**: **React 18** (hooks como `useState`, `useEffect`, `useMemo`, `useCallback`, y Context API para Auth, Búsqueda global y Estados del Escáner).
3. **Estilos y Maquetación**:
   * **Tailwind CSS v3/v4** (clases utilitarias oscuras, bordes con transparencias `border-white/[0.05]` y sombras personalizadas).
   * **Tailwind-Merge** y **Clsx** agrupados en una función de utilidad `cn(...)`.
   * **Radix UI** (primitivas headless de React como `@radix-ui/react-select` para menús de estivas, y `@radix-ui/react-slot` para renderizado polimórfico).
4. **Iconografía**: **Lucide React** (`lucide-react`) para los iconos interactivos y de estado.
5. **Animaciones**: **Framer Motion** (`framer-motion`) para la transición suave de elementos como modales y paneles interactivos.

---

## 4. Guía de Desarrollo para Agentes de IA

Para desarrollar este proyecto de forma automática o asistida por un agente de IA, se debe seguir la siguiente especificación:

### Estructura Recomendada del Proyecto
```text
├── backend/
│   ├── prisma/
│   │   └── schema.prisma        # Definición de Base de Datos
│   ├── src/
│   │   ├── controllers/         # Lógica de endpoints (Auth, Scan, Apicultores, Tambores)
│   │   ├── middleware/          # Validación de Roles y Cookies de sesión
│   │   └── index.js             # Entrada del servidor (Express)
│   ├── package.json
│   └── .env
├── frontend/
│   ├── src/
│   │   ├── components/          # Topbar, Sidebar, UI Elements (Radix/Tailwind)
│   │   ├── context/             # AuthContext, ScanContext
│   │   ├── pages/               # Login, Apicultores, Scan, Estivas, Tambores
│   │   └── App.tsx
│   ├── tailwind.config.js
│   ├── package.json
│   └── vite.config.ts
```

### Rutas API Backend a Desarrollar
El agente de IA deberá proveer controladores Express que implementen:
1. `POST /api/auth/login` y `POST /api/auth/logout`: Gestión de sesiones mediante cookies HTTP-only (`geomiel_sid`).
2. `GET /api/auth/me`: Retorna la sesión activa: `{ id, email, role, enabledModules }`.
3. `GET /api/apicultores?search=&status=&skip=&take=`: Búsqueda y listado paginado de apicultores.
4. `POST /api/apicultores`: Creación de un apicultor, con validación de CUIT usando el algoritmo de módulo 11.
5. `POST /api/apicultores/:id/validar`: Realiza la validación cruzada con RENAPA y pasa el estado a `VALIDADO`.
6. `GET /api/barrels?search=&status=&skip=&take=`: Búsqueda y listado de tambores.
7. `GET /api/barrels/:id`: Retorna detalles de un tambor, incluyendo su apicultor, rango de color e historial de posiciones.
8. `POST /api/scan/armado`: `{ estivaId, barrelCodes: [c1, c2], level, position }`. Asocia un par de tambores a la celda.
9. `POST /api/scan/desarmado`: `{ estivaId, barrelCodes: [c1, c2] }`. Desvincula los tambores de la estiba.
10. `GET /api/scan/suggest/:estivaId`: Retorna la siguiente celda disponible `{ level, position }`.
11. `GET /api/scan/desarmado/suggest/:estivaId`: Sugiere el par superior a retirar.

### Componentes Frontend Críticos
El agente de IA deberá generar los componentes React con soporte para TypeScript:
1. **`LoginForm`**: Captura de credenciales, control de errores y redirección condicional según los módulos habilitados del usuario (`user.enabledModules`).
2. **`Sidebar`**: Menú lateral dinámico que lee `user.enabledModules` y filtra los accesos permitidos.
3. **`ScanArmado`**: Interfaz de escaneo que simula el lector de códigos de barra (con inputs manuales y listeners de teclas), valida el escaneo de a pares y muestra el layout gráfico de las celdas de la estiba.
4. **`ScanDesarmado`**: Interfaz de retiro que implementa la lógica LIFO validando contra `removableBarrelIds` y arrojando los errores informativos correspondientes.
