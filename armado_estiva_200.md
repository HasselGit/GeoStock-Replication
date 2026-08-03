# Secuencia de Armado de una Estiba desde 0 (GeoStock)

Este documento detalla la **secuencia física exacta de las primeras 20 posiciones (10 parejas de tambores)** para construir/armar una estiba de miel desde cero (ejemplo `E39`), comenzando sobre el suelo en el **Piso 1 (`P1`)**.

---

## 1. Regla de Construcción de Base Físico-Mecánica

1.  **Fundación en Piso 1 (Piso Base)**:
    *   No se puede colocar ningún tambor en elevación (Piso 2, 3, 4 o 5) si no existen los tambores del Piso 1 en el suelo que le sirvan de apoyo.
    *   La API `GET /api/scan/suggest/:estivaId` para una estiba vacía sugiere iniciar obligatoriamente en `{ level: 1, position: 1 }`.
2.  **Colocación en Pares Obligatorios**:
    *   Cada celda requiere colocar 2 tambores simultáneamente (Lado Derecho `D` y Lado Izquierdo `I`) compartiendo un identificador de pareja `pairId`.
3.  **Avance de Ubicación (`U`) en Piso 1**:
    *   El Piso 1 es un piso impar (`1 % 2 == 1`), por lo que las Ubicaciones `U` se colocan en orden **directo** desde la entrada del pasillo (`U1` al `U22`).

---

## 2. Lista Exacta de las Primeras 20 Posiciones (10 Pares) para Armar desde 0

```mermaid
flowchart LR
    P1U1["1. E39-D-P1-U1 & E39-I-P1-U1"] --> P1U2["2. E39-D-P1-U2 & E39-I-P1-U2"]
    P1U2 --> P1U3["3. E39-D-P1-U3 & E39-I-P1-U3"]
    P1U3 --> P1U4["4. E39-D-P1-U4 & E39-I-P1-U4"]
    P1U4 --> P1U10["... 10. E39-D-P1-U10 & E39-I-P1-U10"]
```

### Detalle de las 20 Posiciones:

| N° Orden | Posición Exacta en Sistema | Nivel / Elevación | Lado | Unidad / Ubicación | Estado Operativo |
| :---: | :---: | :---: | :---: | :---: | :---: |
| **1** | `E39-D-P1-U1` | **Piso 1 (Base)** | Derecho | `U1` | Primer tambor sobre el suelo |
| **2** | `E39-I-P1-U1` | **Piso 1 (Base)** | Izquierdo | `U1` | Compañero del Par 1 |
| **3** | `E39-D-P1-U2` | **Piso 1 (Base)** | Derecho | `U2` | Par 2 en el suelo |
| **4** | `E39-I-P1-U2` | **Piso 1 (Base)** | Izquierdo | `U2` | Compañero del Par 2 |
| **5** | `E39-D-P1-U3` | **Piso 1 (Base)** | Derecho | `U3` | Par 3 en el suelo |
| **6** | `E39-I-P1-U3` | **Piso 1 (Base)** | Izquierdo | `U3` | Compañero del Par 3 |
| **7** | `E39-D-P1-U4` | **Piso 1 (Base)** | Derecho | `U4` | Par 4 en el suelo |
| **8** | `E39-I-P1-U4` | **Piso 1 (Base)** | Izquierdo | `U4` | Compañero del Par 4 |
| **9** | `E39-D-P1-U5` | **Piso 1 (Base)** | Derecho | `U5` | Par 5 en el suelo |
| **10** | `E39-I-P1-U5` | **Piso 1 (Base)** | Izquierdo | `U5` | Compañero del Par 5 |
| **11** | `E39-D-P1-U6` | **Piso 1 (Base)** | Derecho | `U6` | Par 6 en el suelo |
| **12** | `E39-I-P1-U6` | **Piso 1 (Base)** | Izquierdo | `U6` | Compañero del Par 6 |
| **13** | `E39-D-P1-U7` | **Piso 1 (Base)** | Derecho | `U7` | Par 7 en el suelo |
| **14** | `E39-I-P1-U7` | **Piso 1 (Base)** | Izquierdo | `U7` | Compañero del Par 7 |
| **15** | `E39-D-P1-U8` | **Piso 1 (Base)** | Derecho | `U8` | Par 8 en el suelo |
| **16** | `E39-I-P1-U8` | **Piso 1 (Base)** | Izquierdo | `U8` | Compañero del Par 8 |
| **17** | `E39-D-P1-U9` | **Piso 1 (Base)** | Derecho | `U9` | Par 9 en el suelo |
| **18** | `E39-I-P1-U9` | **Piso 1 (Base)** | Izquierdo | `U9` | Compañero del Par 9 |
| **19** | `E39-D-P1-U10` | **Piso 1 (Base)** | Derecho | `U10` | Par 10 en el suelo |
| **20** | `E39-I-P1-U10` | **Piso 1 (Base)** | Izquierdo | `U10` | Compañero del Par 10 |

---

> [!TIP]
> **Fundamentación Física**: Todas estas 20 posiciones pertenecen al **Piso 1 (`P1`)** porque constituyen la base firme del suelo sobre la cual se asentará la pirámide de tambores de miel.
