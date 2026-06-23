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
    *   `Barrel` (Tambor) debe estar opcionalmente asociado a `Estiva` (celda física) y `Recepcion` (lote de ingreso).
    *   `EstivaPosition` debe coordinar la celda física exacta de un tambor (`level` y `position`) y agruparlos en parejas mediante un `pairId`.
*   **IDs**: Todos los identificadores únicos (`id`) deben seguir el estándar CUID. Los usuarios deben llevar el prefijo `usr_`.

---

## 3. Lógica de Negocio Obligatoria (Core Domain)

### A. Validación de CUIT (Argentina)
Toda entrada de CUIT en altas de apicultores o consultas debe ser validada mediante el algoritmo de **módulo 11**:
*   Los coeficientes de multiplicación son `[5, 4, 3, 2, 7, 6, 5, 4, 3, 2]`.
*   El CUIT debe formatearse y almacenarse sin guiones (11 caracteres numéricos) y validarse contra su último dígito verificador.
*   No se deben permitir caracteres no numéricos en la persistencia.

### B. Módulo de Escáner: Armado de Estivas
*   Los tambores se colocan exclusivamente **de a pares (2 tambores)** por celda.
*   Al armar una posición en la estiba, la API `/api/scan/suggest/:estivaId` debe proponer de forma predictiva la siguiente celda libre (`level` y `position`).
*   La operación de colocación en estiva (`scanArmado`) requiere recibir dos códigos de tambor (`barrelCodes: [código1, código2]`), la ID de la estiba, el nivel y la posición.

### C. Módulo de Escáner: Desarmado de Estivas (Regla LIFO)
*   **Validación de Obstrucción Física**: No se permite retirar tambores ubicados en niveles inferiores si hay tambores colocados encima de ellos en la misma estiba.
*   La API del backend debe calcular dinámicamente qué tambores son removibles (`removableBarrelIds`). Si el operario escanea un tambor que no se encuentra en esta lista, la UI **debe** arrojar el error: `"Retirás primero tambores superiores"`.
*   **Validación de Compañero**: El escaneo requiere el retiro del par completo. Si el operario escanea el primer tambor de una celda, el segundo tambor escaneado **debe** corresponder al compañero del mismo `pairId` registrado. Si no coincide, la UI **debe** arrojar el error: `"[Código] no es el par del primero. Escaneá el compañero."`.

---

## 4. Estética y Experiencia de Usuario (UI/UX)

*   **Paleta de Colores**: Tema oscuro moderno y premium. El color de fondo principal debe ser `#08090A`.
*   **Tipografía**:
    *   Textos generales: **Inter**.
    *   Códigos de barras, CUITs, números y celdas: **Commit Mono** (para fácil lectura en entornos de almacén).
*   **Componentes UI**: Utilizar primitivas headless de **Radix UI** para interactivos complejos (Dropdowns, Modales, Comboboxes) estilizados con **Tailwind CSS**.
*   **Lector**: Las interfaces de escáner deben admitir tanto entrada por escáner de hardware (mediante emulación de teclado con evento Enter) como la introducción manual de códigos en caso de fallas de lectura.
