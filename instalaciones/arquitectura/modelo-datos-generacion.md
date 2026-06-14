# Modelo de datos — Generación y almacenamiento

> Este documento describe la **parte técnica** de los activos de generación. Está
> integrado en el modelo de red ([modelo-datos.md](modelo-datos.md)) y se rige por
> los mismos principios ([ADR-000](../decisiones/ADR-000-principios-y-alcance.md)),
> topología ([ADR-001](../decisiones/ADR-001-topologia-terminal-nodo.md)),
> estructura ([ADR-002](../decisiones/ADR-002-herencia-tablas.md)) y criterios
> ([ADR-006](../decisiones/ADR-006-criterios-modelado.md)). La frontera técnico/BDDAT
> de generación se detalla en [ADR-003](../decisiones/ADR-003-limite-modelo-generacion.md).

## Principio rector — frontera en la interfaz AC del producto de generación

El modelo de generación **no es un subsistema con tablas y FK propias**: son
**activos de red** como cualquier otro, agrupados por **contención** (`envolvente_id`).

La **frontera** del modelo es la **interfaz AC del producto de generación** (donde el
fabricante entrega un equipo cerrado y certificado):

| Tecnología | Producto cerrado (frontera) | Tensión del terminal | Caja negra (fuera del modelo) |
|---|---|---|---|
| FV (string / central con CT aparte) | inversor | **BT** (400/690/800 V) | DC: paneles → strings → inversor |
| FV "AC block" / skid con trafo integrado | inversor + trafo | **MT** | DC + BT interna del skid |
| Eólica | aerogenerador (trafo integrado) | **MT** | góndola, su BT y su trafo |
| Almacenamiento | PCS / inversor bidireccional | BT o MT | celdas DC + BMS |

Reglas que se derivan:

- **DC fuera, AC dentro aunque sea BT.** El lado DC (REBT, ITC-BT) es la *caja negra*
  de la unidad y no se modela. La **AC de evacuación sí se modela**, incluida la **BT**
  entre el inversor y el centro elevador (cuadros AC y puentes del transformador entran
  en el RD 337/2014; ver [ADR-003](../decisiones/ADR-003-limite-modelo-generacion.md)).
- **La unidad de generación es una hoja del grafo:** **1 terminal, sin multiconexión**
  (fuente). Tiene tensión propia (la de su salida AC) y **define el nodo** al que conecta
  (R3). Es el primer activo de 1 terminal sin multiconexión del modelo.
- **Conecta siempre por línea**, nunca por terminal directo a otro activo: en FV/batería
  por una **línea BT** hasta el cuadro AC del CT elevador; en eólica por una **línea MT**
  de la red de evacuación interna del parque.

## Encaje en el modelo de red

```
activo_red (base)
  ├── envolvente            tipo = planta_fotovoltaica / parque_eolico /
  │                                planta_almacenamiento / planta_hibrida
  ├── unidad_fv             1 terminal (BT/MT) — inversor + isla DC (caja negra)
  ├── unidad_eolica         1 terminal (MT)    — aerogenerador (trafo integrado)
  └── unidad_almacenamiento 1 terminal (BT/MT) — PCS bidireccional + batería (caja negra)

planta (envolvente) ──contiene──▶ unidad_*, CT elevador, líneas BT/MT, embarrados…
unidad_* ── terminal ──▶ nodo ── terminal ──▶ línea (BT/MT) ──▶ resto de la red
```

- **La planta es una `envolvente` pura**, sin tabla ni atributos propios. Tras enviar a
  BDDAT lo administrativo y a las vistas lo agregable, **no le queda ningún dato técnico
  propio**: solo `nombre` (de `activo_red`) y `tipo` (de `envolvente`). Ejerce contención
  (C8). Su geometría poligonal va por `geometria`+`activo_geometria` (compartible), no
  como campo. Ver [modelo-datos.md](modelo-datos.md) (tabla `envolvente`).
- **Las unidades son subtipos directos de `activo_red`** (herencia de un nivel, ADR-002).
  **No hay capa común `unidad_generacion`**: `activo_red` ya es el ancla de toda relación
  polimórfica (terminal, geometría, contención). La clasificación "es generación" se
  resuelve en el **backend** con `es_generacion()` (enumera estas tablas), coherente con
  que el tipo es derivable de la subtabla (ADR-002/006).
- **La BT AC es red normal** en BT: `linea` (tipo aérea/subterránea, `tension_id` BT) y
  `elemento_corte` (cuadro AC = seccionador-fusible) operando en BT. Requiere el catálogo
  `tension` ampliado con valores BT (ver modelo-datos.md, catálogo `tension`).

## Marco normativo (conservado)

### Terminología RD 647/2020 (Códigos de Red)

| Término | Definición | En este modelo |
|---|---|---|
| **MGE** | Módulo de Generación — instalación completa | `envolvente` (planta) + vista de agregación |
| **UGE** | Unidad de Generación — cada generador individual | `unidad_fv` / `unidad_eolica` / `unidad_almacenamiento` |
| **CAMGE** | Componentes Adicionales del MGE | activos auxiliares de red (trafos, líneas, celdas) |

- En FV la UGE es **solo el inversor** (el lado DC no forma parte de la planta a efectos
  de conexión); en eólica, el **aerogenerador completo**.
- Almacenamiento híbrido: comparte punto de conexión con la generación (RD 997/2025) →
  se modela como `unidad_fv` + `unidad_almacenamiento` en la **misma envolvente**
  (`tipo = planta_hibrida`), conectadas al mismo nodo/embarrado.

### Normativa de referencia

| Norma | Contenido |
|---|---|
| **RD 997/2025** | Potencia instalada: bifacial ×1,15, almacenamiento, híbridas |
| **RD 647/2020** | Códigos de red UE: MGE/UGE/CAMGE; frontera en el inversor |
| **RD 337/2014 (ITC-RAT)** | Incluye cuadros BT y puentes del transformador → justifica modelar la BT AC |
| **REBT (RD 842/2002, ITC-BT)** | Lado DC/BT interno (caja negra de la unidad) |
| **RD 413/2014 (RAIPEE)** | Registro: trabaja a nivel **instalación → fase → división** (ver "Frontera con BDDAT") |
| **RD 1183/2020** | Acceso y conexión (punto de conexión) |
| **PO 12.2 (REE)** | Requisitos en el punto de conexión (reactiva, tensión, control) — *pendiente* |

### Cobertura en IEC CIM 61970

`PowerElectronicsUnit` (base de `BatteryUnit`/`PhotovoltaicUnit`/`PowerElectronicsWindUnit`)
aporta **muy poco en común** (solo `maxP`/`minP`): confirma que no procede una capa común
de tecnologías. Lo rico que CIM separa es **`PowerElectronicsConnection`** — el punto de
conexión AC con sus parámetros de red (potencia aparente `ratedS`, tensión `ratedU`,
reactiva `maxQ`/`minQ`, control de tensión). En nuestro modelo eso es un asunto del
**punto de conexión / planta** (transversal, no por tecnología) y se aborda con PO 12.2
(*pendiente*), no como superclase de las unidades.

## Tablas

### `unidad_fv` — inversor FV y su isla de paneles (UGE)

Caja negra DC (paneles→inversor) con salida AC. 1 terminal. La potencia instalada legal
(RD 997/2025) es intra-fila → columna `GENERATED` (ver [ADR-005](../decisiones/ADR-005-calculo-potencia-instalada.md)).
Absorbe el antiguo `subcampo_fv`: 1 inversor = 1 isla; varias orientaciones/MPPT bajo un
mismo inversor se describen de forma agregada en `notas` (nivel administrativo, C3/P4).

| Columna | Tipo | Descripción |
|---|---|---|
| activo_id | UUID PK/FK | |
| tension_id | FK → tension | tensión de **salida AC** del inversor (define el nodo); típ. BT |
| potencia_nominal_ac_kw | NUMERIC | potencia nominal AC del inversor (UGE) |
| potencia_pico_modulos_kwp | NUMERIC | Σ potencia pico STC de los módulos (lado DC) |
| es_bifacial | BOOLEAN | |
| factor_bifacial | NUMERIC | default 1.15 (RD 997/2025 Art. 5.2) |
| **potencia_instalada_kw** | NUMERIC **GENERATED** | `LEAST(pico × factor_bifacial_efectivo, potencia_nominal_ac_kw)` |
| tipo_fotovoltaica | TEXT | suelo / cubierta / flotante… (catálogo registro; nullable) |
| seguimiento | TEXT | fijo / un_eje / dos_ejes (nullable) |
| orientacion_grados | NUMERIC | azimut (nullable) |
| inclinacion_grados | NUMERIC | (nullable) |
| num_modulos | INTEGER | (nullable, detalle) |
| num_strings | INTEGER | (nullable, detalle) |
| potencia_modulo_wp | NUMERIC | (nullable, detalle) |
| tecnologia_modulo | TEXT | monocristalino / HJT / CdTe… (nullable) |
| fabricante / modelo / num_serie | TEXT | (nullable) |

```sql
potencia_instalada_kw NUMERIC GENERATED ALWAYS AS (
  LEAST(
    potencia_pico_modulos_kwp * CASE WHEN es_bifacial THEN factor_bifacial ELSE 1.0 END,
    potencia_nominal_ac_kw
  )
) STORED
```

### `unidad_eolica` — aerogenerador (UGE)

Producto cerrado con trafo elevador integrado; entrega en MT. 1 terminal. Lo interno
(góndola, su BT, su trafo) es caja negra.

| Columna | Tipo | Descripción |
|---|---|---|
| activo_id | UUID PK/FK | |
| tension_id | FK → tension | tensión de **salida** (MT); define el nodo |
| potencia_nominal_kw | NUMERIC | potencia nominal del aerogenerador |
| **potencia_instalada_kw** | NUMERIC | = potencia nominal (RD 997/2025) |
| diametro_rotor_m | NUMERIC | (nullable) |
| altura_buje_m | NUMERIC | (nullable) |
| clase_iec_viento | TEXT | I / II / III / S (nullable) |
| fabricante / modelo / num_serie | TEXT | (nullable) |

### `unidad_almacenamiento` — batería + PCS (UGE)

Análogo a FV (caja negra DC = celdas+BMS; frontera = PCS bidireccional), más la
caracterización **energética**. **Bidireccional** (carga/descarga por el mismo punto) →
sigue siendo **1 terminal**. Sustituye al antiguo satélite `almacenamiento`.

| Columna | Tipo | Descripción |
|---|---|---|
| activo_id | UUID PK/FK | |
| tension_id | FK → tension | tensión de **salida AC** del PCS; BT o MT |
| potencia_nominal_kw | NUMERIC | potencia AC del PCS (bidireccional) |
| **potencia_instalada_kw** | NUMERIC | según RD 997/2025 |
| capacidad_kwh | NUMERIC | capacidad energética nominal |
| potencia_carga_kw | NUMERIC | máxima de carga |
| potencia_descarga_kw | NUMERIC | máxima de descarga |
| tecnologia | TEXT | litio_ion / flujo_vanadio / plomo_acido… |
| profundidad_descarga_pct | NUMERIC | DoD (nullable) |
| rendimiento_roundtrip_pct | NUMERIC | (nullable) |
| ciclos_vida | INTEGER | (nullable) |
| fabricante / modelo / num_serie | TEXT | (nullable) |

## Potencia instalada (ADR-005)

| Nivel | Estrategia |
|---|---|
| Unidad | columna (en FV `GENERATED`; en eólica/almacenamiento, el dato nominal) |
| Planta | **vista** (agregación), nunca columna almacenada |
| Operacional (por estados) | backend (Python), depende de estados → fuera del modelo |

```sql
-- Vista polimórfica de generación
CREATE VIEW v_generacion AS
  SELECT activo_id, 'fv'             AS tecnologia, potencia_instalada_kw FROM unidad_fv
  UNION ALL
  SELECT activo_id, 'eolica',          potencia_instalada_kw FROM unidad_eolica
  UNION ALL
  SELECT activo_id, 'almacenamiento',  potencia_instalada_kw FROM unidad_almacenamiento;

-- Potencia instalada por planta (envolvente que contiene las unidades)
CREATE VIEW v_potencia_planta AS
  SELECT ar.envolvente_id AS planta_id,
         SUM(g.potencia_instalada_kw) AS potencia_instalada_kw
  FROM v_generacion g
  JOIN activo_red ar ON ar.id = g.activo_id
  GROUP BY ar.envolvente_id;
```

> Si la pertenencia a la planta está **anidada** (la unidad cuelga de un CT/centro de
> inversión dentro de la planta), la agregación se resuelve recorriendo la jerarquía de
> `envolvente_id` (CTE recursivo). La vista anterior asume contención directa.

## Frontera con BDDAT (qué NO se modela aquí)

El RAIPEE (RD 413/2014, app PRETOR) trabaja a nivel **Instalación → Fase → División de
fase**, no de equipo. Su contenido es **registral**, no técnico, y vive en **BDDAT**:

| Dato del registro | Dónde vive |
|---|---|
| Titular, CIF | BDDAT (titularidad, ADR-000) |
| **Fase** (puesta en servicio) — CIL, nº y fechas de inscripción, RD, tipo inscripción | **BDDAT** (historial de la realidad; no pre-modelable) |
| **Potencia bruta / neta / mínima** (acreditadas por prueba, art. 37.3) | **BDDAT** (registral, por fase) |
| Tipo de hibridación (I/II/III) | BDDAT (clasificación retributiva) |
| Grupo normativo (art. 2 RD 413/2014) | catálogo / derivable del `tipo` y las unidades |
| Estado físico/administrativo | BDDAT (deducible, ADR-000) |
| Potencia instalada total | vista `v_potencia_planta` |
| Poligonal / ubicación | `geometria`+`activo_geometria` (ver abajo) |

**La fase es la unidad de puesta en servicio independiente** (lo que el promotor energiza
por separado). Es BDDAT: un **agrupador lógico** (sin recinto) que referencia un conjunto
de `activo_red.id`. Cuando BDDAT pregunta *"¿qué poligonal usa este cluster?"*, lo resuelve
**instalaciones**: recorre los `activo_red.id` de la fase → `activo_geometria` → la(s)
geometría(s) poligonal(es) del recinto que ocupan (compartidas cuando varios clusters están
en el mismo vallado). La poligonal **no es un campo de la fase**; es geometría real,
anclada a la envolvente-recinto y compartible.

## Topología por tecnología

![Esquema de tensiones de una planta FV, de los paneles al punto de entrega](../esquemas/tension-planta-fv.svg)

- **FV (terminal BT):** `unidad_fv` ──(línea BT)──▶ cuadro AC (`elemento_corte` BT) ──▶
  trafo BT/MT del CT elevador ──▶ red MT interna ──▶ embarrado SET ──▶ trafo elevador
  MT/AT ──▶ línea de evacuación ──▶ punto de conexión (gestor).
- **Eólica (terminal MT):** `unidad_eolica` ──(línea MT)──▶ red de evacuación interna del
  parque ──▶ SET ──▶ línea de evacuación.
- **Almacenamiento:** igual que FV (terminal BT con CT elevador) o como eólica (terminal MT
  si el AC-block lleva trafo integrado). En híbrida, comparte nodo/embarrado con la FV.

## Pendiente de definir

- **PO 12.2 / punto de conexión:** parámetros de red del conjunto (potencia aparente,
  capacidad de reactiva, control de tensión). Equivale a `PowerElectronicsConnection` de
  CIM; transversal y a nivel de **planta / punto de conexión**, no por unidad.
- **Catálogo `tipo_fotovoltaica`** (suelo/cubierta/flotante…) y demás subtipos del registro.
- Curvas de potencia de aerogeneradores; perfiles de generación FV.
- Reglas finas de potencia instalada de **híbridas** (RD 997/2025) en la vista de agregación.
