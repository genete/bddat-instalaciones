# ADR-007 — Agrupación lógica de líneas mediante envolvente lógica

**Estado:** Aceptado
**Fecha:** 2026-06-14

> Sustituye a una primera versión que introducía una tabla aparte `agrupacion_logica` con
> FK propia `agrupacion_id`. Tras revisión, se descarta esa tabla: la agrupación lógica de
> líneas se modela **reutilizando `envolvente`** (ver "Alternativas descartadas").

## Contexto

Lo que modelamos como `linea` es el **tramo** de las compañías: un conductor que une dos
elementos de otro tipo (nodos). Pero distribuidoras y REE nombran **líneas con identidad
propia** —"LA JANDA", "SET A – SET B"— que son **conjuntos de tramos** bajo una misma
denominación. Esa denominación es como se lee el esquema unifilar y como se redacta la
resolución (P2); enumerar "SL435, SL436…" en vez de "Línea LA JANDA" es ininteligible.

Hace falta un agrupador de tramos que dé esa identidad. La cuestión es **cómo**: ¿tabla
nueva o el mecanismo de contención que ya existe?

## Decisión

La agrupación lógica de líneas se modela como **envolvente lógica**: una `envolvente`
**sin reflejo físico**, distinguida por el `tipo` (`linea`, `circuito`). Reutiliza el
mecanismo de contención existente —`activo_red.envolvente_id` (self-FK, anidable)— **sin
añadir tabla ni FK nuevas**.

- Una **línea con nombre** ("LA JANDA", "SET A – SET B") es una envolvente `tipo = linea`
  que contiene sus tramos (`linea`).
- Un **circuito** con varios tramos (mixto aéreo/subterráneo, o partido por empalmes) es una
  envolvente `tipo = circuito` anidada bajo la línea.
- La pertenencia de un tramo a la línea es `tramo.envolvente_id → envolvente-línea`; la
  pertenencia a niveles superiores es **transitiva** por la jerarquía.

### Regla del contenedor mínimo

**No se crean contenedores de un solo hijo.** Un nivel de envolvente (línea, circuito) existe
**solo si agrupa más de un activo**. El anidamiento **surge por necesidad**, no por plantilla.

**Doble circuito de tramos simples** (cada circuito = 1 tramo) — *sin* niveles intermedios:

| id | nombre | subtabla | envolvente_id |
|---|---|---|---|
| `e0` | "SET A–SET B" | envolvente (`linea`) | NULL |
| `L1` | tramo terna 1 | linea | `e0` |
| `L2` | tramo terna 2 | linea | `e0` |
| `A1` | apoyo común | apoyo | `e0` |

**Solo si un circuito tiene varios tramos** aparece su envolvente (aquí circuito 2 = aéreo +
subterráneo):

| id | nombre | subtabla | envolvente_id |
|---|---|---|---|
| `e0` | "SET A–SET B" | envolvente (`linea`) | NULL |
| `L1` | tramo circ. 1 | linea | `e0` | *(directo, sin contenedor)* |
| `eC2` | "Circuito 2" | envolvente (`circuito`) | `e0` | *(necesario: agrupa 2 tramos)* |
| `L2a` | tramo aéreo circ. 2 | linea | `eC2` |
| `L2b` | tramo subt. circ. 2 | linea | `eC2` |
| `EMP` | empalme aéreo/subt | empalme | `eC2` |
| `A1` | apoyo común | apoyo | `e0` |

### Geometría (potestativa)

Una **envolvente lógica no tiene geometría propia**: su representación es la **agregación**
(`ST_Union` / envolvente) de las geometrías de sus activos contenidos. La geometría se hereda
hacia arriba por la jerarquía de contención; no se almacena en el contenedor lógico (P3). Una
envolvente física (recinto) sí puede llevar shape propio, opcional.

## Frontera normativa (por `tipo`, no estructural)

`envolvente` pasa a ser un contenedor **físico o lógico**, distinguido por el `tipo`:

| `tipo` | Qué agrupa | Norma |
|---|---|---|
| físicos (CT, subestación, posición…) | **instalaciones**: recintos | **RD 337/2014** |
| lógicos (`linea`, `circuito`) | **líneas**: conjuntos de tramos | **RD 223/2008** |

## Alternativas descartadas

| Alternativa | Problema |
|---|---|
| **Tabla aparte `agrupacion_logica` + FK `agrupacion_id`** (primera propuesta) | No aporta capacidad sobre la envolvente. El "doble eje físico/lógico" que la justificaba **no se da**: los activos que se agrupan en una línea (tramos, y por contención sus apoyos) **no están en recintos** → su `envolvente_id` está libre para el eje lógico; y los activos de recinto (posiciones, celdas) se relacionan con su línea por **topología** (`terminal`–`nodo`), no por contención. Un artefacto y una FK de más. |
| Pertenencia **N:M** | Overkill: un activo cuelga del nivel más específico y el resto se hereda. Solo haría falta con compartición parcial (un activo en unos circuitos sí y en otros no), infrecuente. |
| Agrupación como entidad sin terminales pero con geometría propia | La línea-nombre no tiene shape propio; su traza es agregación de los tramos. |

## Consecuencias

- **No hay tabla `agrupacion_logica` ni columna `agrupacion_id`.** Un mecanismo único de
  contención (`envolvente_id`) cubre lo físico y lo lógico.
- `envolvente` se reinterpreta como **contenedor físico o lógico** (atributo `tipo`).
- "¿Qué línea pasa por esta posición de SET?" se responde por **topología**
  (`posición → terminal → nodo ← terminal ← tramo → envolvente-línea`), no por agrupación.
- La geometría de una línea/circuito es una **consulta** (`ST_Union`), no un dato.
- **Reaparición de una 2.ª referencia** solo si algún día un activo debiera pertenecer a dos
  jerarquías **no anidables por contención** a la vez. Hoy no ocurre; si surgiera, se añade
  entonces.
