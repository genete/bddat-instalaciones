# ADR-007 — Agrupación lógica de líneas (conjunto de tramos)

**Estado:** Aceptado
**Fecha:** 2026-06-14

## Contexto

Lo que modelamos como `linea` es el **tramo** de las compañías: un conductor que une dos
elementos de otro tipo (nodos). Pero distribuidoras y REE nombran **líneas con identidad
propia** —"LA JANDA", "SET A – SET B"— que son **conjuntos de tramos** bajo una misma
denominación. Esa denominación es como se lee el esquema unifilar y como se redacta la
resolución; enumerar "SL435, SL436…" en vez de "Línea LA JANDA" es ininteligible para quien
la recibe (P2).

Hace falta un **agrupador lógico de tramos**, distinto de la contención física.

## Decisión

Se introduce **`agrupacion_logica`**, un contenedor **sin datos propios salvo el nombre**:

| Columna | Tipo | Descripción |
|---|---|---|
| id | UUID PK | |
| nombre | TEXT | "LA JANDA", "SET A – SET B" |
| padre_id | UUID FK → agrupacion_logica | anidamiento (nullable) |

La pertenencia es **1:N**: una FK nullable **`agrupacion_id → agrupacion_logica.id`** en
`activo_red`. Reglas:

- Cada activo cuelga del **nivel más específico al que pertenece en exclusiva**; la
  pertenencia a los niveles superiores es **transitiva** por `padre_id` (no se declara).
- **No es `activo_red`**: la línea-con-nombre no es tangible (no se monta, no se pone en
  servicio, no tiene geometría propia). Su traza es **derivada** (`ST_Union` de los tramos).
- **Anidable** (`padre_id`): cubre desde el tramo único hasta el haz de circuitos.

### Cómo cubre los casos

- **Circuito mixto (tramos encadenados):** una agrupación plana con sus tramos en serie.
- **Doble/múltiple circuito en paralelo (mismo origen y final):** la línea-nombre como
  **padre** que anida una agrupación por **circuito**; el "mismo origen/final" emerge de la
  topología (los tramos extremos comparten los nodos de las SET), no se codifica.

**Ejemplo — doble circuito "SET A–SET B"** (G0) con Circuito 1 (G1) y 2 (G2), tramos L1/L2
y apoyos compartidos A1–A3:

| activo | agrupacion_id |
|---|---|
| L1 (terna 1) | G1 |
| L2 (terna 2) | G2 |
| A1, A2, A3 (apoyos comunes) | G0 |

El apoyo común cuelga de **G0** (donde es común); el conductor, de su circuito. "L1 es de
SET A–SET B" se infiere por `G1 → G0`. Sin duplicar filas.

## Frontera normativa: agrupación (líneas) vs envolvente (instalaciones)

Son **dos mecanismos ortogonales** con respaldo normativo distinto:

| Mecanismo | Qué agrupa | Norma |
|---|---|---|
| **`agrupacion_logica`** | **líneas**: conjuntos de tramos | **RD 223/2008** (Reglamento de líneas de AT) |
| **`envolvente`** (contención física) | **instalaciones**: recintos (CT, subestación, posición) | **RD 337/2014** (Reglamento de instalaciones de AT) |

Un mismo activo puede tener **ambas**: un seccionador puede estar *contenido* en un CT
(`envolvente_id`, RD 337/2014) y *pertenecer* a la línea LA JANDA (`agrupacion_id`,
RD 223/2008). No se mezclan.

## Alternativas descartadas

| Alternativa | Problema |
|---|---|
| Tabla intermedia **N:M** (`agrupacion_activo`) | **Overkill** para el caso normal: duplica los apoyos comunes que el árbol ya hereda. Solo se justifica con **compartición parcial** (un activo en unos circuitos sí y en otros no), infrecuente. **Migrable a N:M sin pérdida** si aparece. |
| Reutilizar `envolvente_id` | Mezcla contención **física** (recinto) con agrupación **lógica** (sin recinto); ensucia ambas semánticas. |
| Hacer la agrupación un `activo_red` | No es tangible: sin terminales, sin geometría propia, sin puesta en servicio. |

## Consecuencias

- `activo_red` gana una segunda referencia de agrupación, **ortogonal** a `envolvente_id`:
  `agrupacion_id` (lógica, líneas, RD 223/2008) vs `envolvente_id` (física, instalaciones,
  RD 337/2014).
- El descriptor humano (P2) recorre la jerarquía de `agrupacion_logica` para nombrar "Línea
  LA JANDA, doble circuito" y enumerar sus tramos.
- La traza de una línea-nombre es una consulta (`ST_Union`), no un dato.
- Si aparece compartición parcial de activos entre circuitos, la FK se migra a tabla enlace
  N:M sin cambios conceptuales.
