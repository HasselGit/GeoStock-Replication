# Plan de Implementación Paso a Paso para GeoStock (GeoMiel)

Este documento sirve como la **hoja de ruta secuencial obligatoria** para que cualquier agente de IA o desarrollador pueda construir y desplegar la réplica exacta del sistema GeoStock desde cero, asegurando que las dependencias de base de datos y lógica de negocio se implementen en el orden más conveniente.

---

## Fases de Construcción Secuencial

```mermaid
flowchart TD
    Fase1["Fase 1: Infraestructura y Base de Datos (Prisma)"] --> Fase2["Fase 2: Autenticación y Usuarios"]
    Fase2 --> Fase3["Fase 3: Catálogos Base (Depósitos y Referencias)"]
    Fase3 --> Fase4["Fase 4: Módulo Apicultores (CUIT Módulo 11 + RENAPA)"]
    Fase4 --> Fase5["Fase 5: Módulo Recepciones, Pesaje y ZPL"]
    Fase5 --> Fase6["Fase 6: Módulo Estivas y Escáner (Regla LIFO Pares)"]
    Fase6 --> Fase7["Fase 7: Módulo Laboratorio y Canastos"]
    Fase7 --> Fase8["Fase 8: Módulo Compras e Integración Finnegans"]
    Fase8 --> Fase9["Fase 9: Módulo Logística, Reservas y Remitos"]
    Fase9 --> Fase10["Fase 10: Dashboard Real-Time y Pulido UI/UX"]
```

---

### Paso 1: Base de Datos e Infraestructura
1. Configurar la conexión a **PostgreSQL** en el archivo `.env`.
2. Ejecutar `npx prisma db push` sobre [schema.prisma](backend/prisma/schema.prisma) para generar automáticamente las **25 tablas**, tipos enum y relaciones relacionales.
3. Ejecutar `npx prisma generate` para tener el cliente de `@prisma/client` completamente tipado.

### Paso 2: Autenticación y Gestión de Usuarios
1. Crear el controlador de login (`POST /api/auth/login`) con hashing de contraseñas (bcrypt) y generación de cookie de sesión (`geomiel_sid`).
2. Crear middleware de autenticación que inyecte `req.user` conteniendo `enabledModules`, `role`, `area` y la bandera `idGeomielObligatorio`.
3. Implementar `/api/auth/me` y `/api/auth/logout`.

### Paso 3: Catálogos Base e Inventario de Referencia
1. Sembrar o poblar vía API la tabla `RangoColor` con los rangos comerciales (`0-15`, `41-45`, `+70`) incluyendo sus límites `min` y `max`.
2. Implementar el ABM de **Depósitos** (`Warehouse`) (`/api/almacenes`).
3. Implementar el ABM de **Referencias** (`Referencia`) (`/api/almacenes/referencias`).

### Paso 4: Módulo de Apicultores
1. Implementar la validación algorítmica de **CUIT Módulo 11** en el backend para denegar CUITs inválidos antes de tocar la base de datos.
2. Implementar la creación y consulta de Apicultores (`/api/apicultores`).
3. Conectar el servicio de consulta y auditoría de **RENAPA** (`/api/apicultores/:id/validar` y `/api/renapa/consultar/:cuit`).

### Paso 5: Recepción, Registro de Tambores, Pesaje y ZPL
1. Crear el ABM de **Recepciones** (`/api/recepciones`).
2. Implementar el registro de tambores (`/api/barrels`) asociando el código de barra interno (ej: `G-2026-000029`) e ID de recepción.
3. Implementar el submódulo de pesaje de balanza (`/api/barrels/pesaje/buscar` y `/api/barrels/pesaje/:id/confirmar`).
4. Implementar el generador de etiquetas térmicas **ZPL** (`GET /api/barrels/:id/etiqueta`) para impresoras Zebra.

### Paso 6: Armado y Desarmado de Estivas (Escáner LIFO)
1. Implementar la creación de celdas físicas de estiba (`/api/estivas`).
2. Implementar la API predictiva de sugerencia de celda libre (`/api/scan/suggest/:estivaId`).
3. Implementar el endpoint `/api/scan/armado` obligando a recibir tambores **de a pares (2 tambores)** asignando un `pairId`.
4. Implementar la regla LIFO de desarmado `/api/scan/desarmado/suggest/:estivaId` y la validación en `/api/scan/desarmado` arrojando los errores estrictos: `"Retirás primero tambores superiores"` y `"[Código] no es el par del primero. Escaneá el compañero."`.

### Paso 7: Módulo de Laboratorio y Canastos
1. Crear órdenes de laboratorio (`/api/laboratorio/iniciar`).
2. Muestreo físico-químico (color, humedad, HMF) y aprobación (`/api/laboratorio/muestras/:id`).
3. Módulo de agrupar muestras en **Canastos** (`/api/laboratorio/canastos`).

### Paso 8: Módulo de Compras, Liquidaciones e Integración ERP
1. Registro de Emisores fiscales (`/api/compra/emisores`).
2. Asignación de tambores a Órdenes de Compra (`/api/compra/ordenes/:id/tambores`).
3. Cálculo dinámico de la liquidación cruzando kg netos y precios por calidad (`CLARA`, `INTERMEDIA`, `OSCURA`).
4. Emisión de facturas (`/api/compra/ordenes/:id/facturas`).
5. Push de integración al ERP **Finnegans** (`/api/compra/ordenes/:id/finnegans/push`).

### Paso 9: Logística, Reservas de Stock y Remitos Digitales
1. ABM de Choferes, Solicitudes Logísticas y Cargas (`/api/logistica/...`).
2. Motor de **Reservas de Stock** (`/api/logistica/reservas`) y cálculo de disponible real (`/api/logistica/stock-disponible`).
3. Emisión de remitos y captura de **Firma Digital** (`/api/logistica/remitos/:id/firma`).

### Paso 10: Dashboard Real-Time y Estética UI/UX
1. Mapear las telemetrías y estadísticas en `/api/dashboard/stats`.
2. Estilizar la aplicación en React siguiendo el tema oscuro `#08090A`, la tipografía **Commit Mono** para códigos/pesos, e interactivos con Radix UI + Tailwind CSS.
