# Directrices de Desarrollo de GeoStock (GeoMiel)

Este archivo contiene las reglas y directrices obligatorias para todos los agentes de IA que trabajen en este repositorio. Cualquier cambio, adición o refactorización del código debe alinearse estrictamente con estas reglas.

---

## 1. Arquitectura y Stack Tecnológico

*   **Frontend**: React (Vite) con TypeScript.
*   **Backend**: Node.js con Express.
*   **Base de Datos**: Base relacional mapeada únicamente con **Prisma ORM** utilizando el archivo `backend/prisma/schema.prisma` como única fuente de verdad para el esquema.
*   **Gestión del Monorepositorio**: Las dependencias y scripts de desarrollo conjuntos se deben administrar desde el archivo `package.json` en la raíz del proyecto usando `concurrently`.

---

## 2. Directrices de Base de Datos y Modelado

*   **Esquema Prisma**: Queda terminantemente prohibido modificar el esquema de base de datos de manera que rompa las relaciones clave:
    *   `Apicultor` debe poseer una relación obligatoria de validación con `RenapaConsulta` para el historial de auditoría.
    *   `Barrel` (Tambor) debe estar opcionalmente asociado a `Estiva` (celda física), `Recepcion` (lote de ingreso) y `OrdenCompra` (orden de compra asignada).
    *   `EstivaPosition` debe coordinar la celda física exacta de un tambor (`level` y `position`) y agruparlos en parejas mediante un `pairId`.
    *   `Factura` debe estar obligatoriamente relacionada a una `OrdenCompra`.
*   **IDs**: Todos los identificadores únicos (`id`) deben seguir el estándar CUID. Los usuarios deben llevar el prefijo `usr_`.

---

## 3. Lógica de Negocio Obligatoria (Core Domain)

### A. Validación de CUIT (Argentina)
Toda entrada de CUIT en altas de apicultores o consultas debe ser validada mediante el algoritmo de **módulo 11**:
*   Los coeficientes de multiplicación son `[5, 4, 3, 2, 7, 6, 5, 4, 3, 2]`.
*   El CUIT debe formatearse y almacenarse sin guiones (11 caracteres numéricos) y validarse contra su último dígito verificador.
*   No se deben permitir caracteres no numéricos en la persistencia.

### B. Módulo de Escáner y Visor 3D: Armado y Nomenclatura de Estivas
*   **Geometría Piramidal de Estiva**: La capacidad total de la estiba se calcula reduciendo 2 tambores (1 par) por nivel de elevación para estabilidad física:
    $$\text{Capacidad Nivel } s = \max(0, \text{baseCapacity} - (s - 1) \times 2)$$
    *Para `maxLevels: 5` y `baseCapacity: 44`, el total ocupado es $44 + 42 + 40 + 38 + 36 = 200 \text{ tambores}$.*
*   **Nomenclatura Humana de Posición (`[Estiba]-[Lado]-P[Piso]-U[Ubicación]`)**:
    *   **Estiba** (ej. `E39`).
    *   **Lado/Cara**: `D` (Derecho, para posiciones impares) o `I` (Izquierdo, para posiciones pares).
    *   **P (Piso / Nivel)**: `P1`, `P2`, `P3`, `P4`, `P5` (indica el piso de elevación 1 al 5).
    *   **U (Ubicación / Unidad)**: Índice del tambor en la fila.
*   **Trabado Alternado en Pisos Pares (Zig-Zag)**: En los pisos impares (`P1`, `P3`, `P5`), la Ubicación `U` se cuenta en orden directo (`1, 2, 3...`). En los **pisos pares (`P2`, `P4`)**, la Ubicación `U` se invierte de sentido (`totalNivel - t + 1`) para representar el trabamiento físico cruzado de los tambores.

### C. Módulo de Escáner: Armado y Desarmado (Regla de Soporte de 4 Tambores de Base)
*   **Regla Física de Apoyo (4 Tambores de Base por 1 Par Superior)**: Para poder colocar un par en un nivel superior (ej. `P2-U1`), se requiere obligatoriamente tener colocados los **2 pares contiguos del piso inferior (4 tambores de base)** que forman la cuna o valle de apoyo (ej. `P1-U22` y `P1-U21` para apoyar `P2-U1`).
*   **Secuencia de Armado Intercalado (Triangular)**:
    1.  `E39-D-P1-U22` y `E39-I-P1-U22` (Par Base 22 en el fondo)
    2.  `E39-D-P1-U21` y `E39-I-P1-U21` (Par Base 21 en el fondo)
    3.  `E39-D-P2-U1` y `E39-I-P2-U1` (Par 1 del Piso 2, apoyado sobre los 4 tambores de `P1-U22` y `P1-U21`)
    4.  `E39-D-P1-U20` y `E39-I-P1-U20` (Par Base 20)
    5.  `E39-D-P2-U2` y `E39-I-P2-U2` (Par 2 del Piso 2, apoyado sobre `P1-U21` y `P1-U20`)
    6.  `E39-D-P3-U1` y `E39-I-P3-U1` (Par 1 del Piso 3, apoyado sobre los 4 tambores de `P2-U1` y `P2-U2`)
*   **Validación de Obstrucción Física**: No se permite colocar ni retirar tambores si no se cumplen las condiciones físicas de apoyo inferior o despeje de cabecera. La UI debe arrojar el error: `"Retirás primero tambores superiores"`.
*   **Validación de Compañero**: Requiere el ingreso de ambos tambores del par (`pairId`). Si el segundo tambor no coincide con el compañero del primero, la UI arremete el error: `"[Código] no es el par del primero. Escaneá el compañero."`.

### D. Módulo de Compras (Liquidación e Integración ERP)
*   **Liquidación de Precios**: La liquidación se debe calcular sumando el importe de los tambores según su clasificación por calidad físico-química (`CLARA`, `INTERMEDIA`, `OSCURA`), multiplicando el precio respectivo definido en la orden por los kilos netos del tambor.
*   **Integración Finnegans**: Las solicitudes de push deben registrar el estado exacto del ERP (`finnegansEstado`, `finnegansId`, `finnegansError`) en la base de datos de facturas para auditoría fiscal.

---

## 4. Estética y Experiencia de Usuario (UI/UX)

*   **Paleta de Colores**: Tema oscuro moderno y premium. El color de fondo principal debe ser `#08090A`.
*   **Tipografía**:
    *   Textos generales: **Inter**.
    *   Códigos de barras, CUITs, números, importes y celdas: **Commit Mono** (para fácil lectura en entornos de almacén).
*   **Componentes UI**: Utilizar primitivas headless de **Radix UI** para interactivos complejos (Dropdowns, Modales, Comboboxes) estilizados con **Tailwind CSS**.
*   **Lector**: Las interfaces de escáner deben admitir tanto entrada por escáner de hardware (mediante emulación de teclado con evento Enter) como la introducción manual de códigos en caso de fallas de lectura.
