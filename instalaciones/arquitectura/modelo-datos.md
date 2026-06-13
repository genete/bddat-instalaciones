# Modelo de datos — Instalaciones eléctricas MT/AT (> 1 kV)

> Este documento describe la **parte técnica** de los activos de red. El estado
> administrativo, el expediente y la tramitación pertenecen a BDDAT y no se
> modelan aquí (ver [ADR-000](../decisiones/ADR-000-principios-y-alcance.md)).
> Los principios de diseño están en [ADR-000](../decisiones/ADR-000-principios-y-alcance.md),
> la topología y reglas en [ADR-001](../decisiones/ADR-001-topologia-terminal-nodo.md),
> la estructura y portabilidad en [ADR-002](../decisiones/ADR-002-herencia-tablas.md)
> y los criterios de modelado en [ADR-006](../decisiones/ADR-006-criterios-modelado.md).

## Contexto normativo — Andalucía

### Distribuidoras

En Andalucía, **eDistribución Redes Digitales S.L.U.** (antes Endesa Distribución)
gestiona la práctica totalidad de la red de distribución. Existen algunas pequeñas
distribuidoras locales (cooperativas, empresas municipales) registradas en la CNMC,
pero en la práctica adoptan las normas de eDistribución o la normativa estatal
directamente, sin normas propias diferenciadas.

La Junta de Andalucía aprobó las normas particulares de eDistribución mediante
Resolución de 5 de mayo de 2005 (BOJA), parcialmente derogada en 2019 y 2020
al ser sustituida por las nuevas especificaciones estatales aprobadas en BOE.

### Normas técnicas de referencia (eDistribución)

| Código   | Nombre                                              | Ámbito                    |
|----------|-----------------------------------------------------|---------------------------|
| **NRZ001** | Especificaciones Particulares instalaciones distribución | MT ≤ 36 kV          |
| **NRZ101** | Instalaciones privadas conectadas a la red. Generalidades | MT/BT privadas     |
| **LRZ001** | Especificaciones líneas aéreas de AT                | AT > 36 kV                |
| **NRZ002** | Instalaciones distribución en baja tensión          | BT ≤ 1.000 V              |

Fuentes normativas:
- [NRZ001 — eDistribución (PDF)](https://www.edistribucion.com/content/dam/edistribucion/ingenieriadered/NRZ001.pdf)
- [LRZ001 Líneas Aéreas AT — eDistribución (PDF)](https://www.edistribucion.com/content/dam/edistribucion/conexion-a-la-red/normativa/LRZ001_EP%20L%C3%ADneas%20A%C3%A9reas%20de%20Alta%20Tensi%C3%B3n_v2.pdf)
- [BOE-A-2021-2294 — Aprobación especificaciones eDistribución](https://www.boe.es/diario_boe/txt.php?id=BOE-A-2021-2294)
- [Energía eléctrica — Junta de Andalucía](https://www.juntadeandalucia.es/organismos/industriaenergiayminas/areas/energia/electricidad.html)
- [Listado distribuidoras — CNMC](https://sede.cnmc.gob.es/listado/censo/1)

### Niveles de tensión en Andalucía

La normativa española distingue dos rangos dentro del ">1000 V" que usa el
proyecto. El modelo soporta ambos mediante el catálogo de tensiones:

| Rango          | Denominación oficial   | Gestor principal | Tensiones habituales              |
|----------------|------------------------|------------------|-----------------------------------|
| 1 kV – 36 kV   | **Media Tensión (MT)** | eDistribución    | 15 kV (legado), **20 kV** (nuevo) |
| > 36 kV        | **Alta Tensión (AT)**  | REE / Redeia     | 66 kV, 132 kV, **220 kV**, **400 kV** |

> La zona gris son los 45 kV y 66 kV de subtransporte: pueden ser de REE o de
> eDistribución según el caso. El catálogo de tensiones cubre todos los rangos
> sin distinción estructural.

---

### Red Eléctrica de España (REE / Redeia) — particularidades

REE es el único operador del sistema de transporte (TSO) en España. Gestiona la
red AT (principalmente 220 kV y 400 kV) y la operación del sistema nacional.

**Conclusión para el modelo:** REE **no se desvía del estándar IEC CIM**. Lo
refuerza: intercambia modelos con ENTSO-E vía **CGMES** (perfil europeo derivado
de IEC CIM 61970). El modelo es compatible con REE sin cambios estructurales.

REE publica requisitos mínimos para instalaciones conectadas a su red de transporte
(TI.E_02_040). Su impacto en este modelo es menor y se resuelve con los campos ya
previstos:

| Requisito REE | Cómo se refleja en el modelo |
|---|---|
| Doble sistema de protección con baterías independientes | Se describe en la celda de protección (`elemento_corte`, función=protección) / `notas` |
| Interruptores en ambos lados de la interconexión | Topología: sendos `elemento_corte` en la frontera |
| Automatización / SCADA / telecontrol | `elemento_corte.accionamiento = telecontrolado` |
| Coordinación de aislamiento por nivel de tensión | `nivel_aislamiento` (catálogo) en `elemento_corte` y `linea` |
| Red de tierra de alta disipación | `red_tierra.resistencia_ohm` |

Fuentes:
- [TI.E_02_040 — REE (PDF)](https://www.ree.es/sites/default/files/01_ACTIVIDADES/Documentos/AccesoRed/TI.E_02_040_Ed5_Cond_Tecnic_Conex_Terceros_RdT_Peninsular.pdf)
- [Requisitos mínimos diseño Ed4 — REE (PDF)](https://www.ree.es/sites/default/files/12_CLIENTES/Documentos/Instalaciones_conectadas_a_la_red_de_transporte_Requisitos_minimos_dise%C3%B1o_equipamiento_Ed4.pdf)
- [RD 337/2014 ITC-RAT — BOE](https://www.boe.es/buscar/doc.php?id=BOE-A-2014-6084)
- [CGMES conformance — DIgSILENT Ibérica](http://www.digsilentiberica.es/noticia/34/ENTSO-e-acredita-la-conformidad-de-DIgSILENT-PowerFactory-con-CGMES/)

---

### Terminología eDistribución (vs denominaciones del modelo)

| Este modelo                | Denominación eDistribución / sector       |
|----------------------------|-------------------------------------------|
| `linea` aérea MT           | LAMT (Línea Aérea de Media Tensión)       |
| `linea` subterránea MT     | LSMT (Línea Subterránea de Media Tensión) |
| `envolvente` CT            | CT (Centro de Transformación)             |
| `envolvente` CS            | CS (Centro de Seccionamiento)             |
| `elemento_corte`           | Celda de línea / de protección / seccionador |
| `transformador` en CT      | Transformador MT/BT                       |

---

## Rol de IEC CIM 61970

CIM es **referencia conceptual y de vocabulario, no un corsé de esquema** (ver
[ADR-000](../decisiones/ADR-000-principios-y-alcance.md)). Se toma el patrón
**Terminal–ConnectivityNode** y la taxonomía; no se persigue CIM-compliance ni CGMES.

---

## Principio de topología

La conectividad eléctrica se modela con dos conceptos (ver [ADR-001](../decisiones/ADR-001-topologia-terminal-nodo.md)):

- **`nodo`** (ConnectivityNode): punto eléctrico donde se unen conductores. No es
  un activo físico; es virtual.
- **`terminal`**: extremo de conexión de un activo. Cada activo tiene tantos
  terminales como puntos de conexión.

Un activo A y un activo B están conectados cuando comparten el mismo `nodo` a
través de sus respectivos `terminal`.

---

## Diagrama de tablas

```
catálogos:  tension      nivel_aislamiento

activo_red (base — identidad, SQL estándar, sin PostGIS)
  ├── envolvente            ← contenedor (recinto); anidable
  ├── linea
  ├── elemento_corte
  ├── transformador
  ├── embarrado
  ├── elemento_medida       (solo AIS/GIS)
  ├── pararrayos
  ├── red_tierra
  ├── apoyo
  └── empalme

activo_red → terminal → nodo            (topología eléctrica)
activo_red → activo_geometria → geometria   (geografía; PostGIS aislado y compartible)
activo_red.envolvente_id → activo_red       (contención, self-FK)
```

> "Protección" y "medida" **no son tipos de activo, son funciones**: se reparten
> entre `elemento_corte` (función), `pararrayos`, `red_tierra`, `elemento_medida`
> (ver [ADR-006](../decisiones/ADR-006-criterios-modelado.md)).

---

## Catálogos

### `tension` — tensiones de servicio normalizadas

| Columna | Tipo | Descripción |
|---|---|---|
| id | PK | |
| valor_kv | NUMERIC | 3, 6, 10, 15, 20, 30, 45, 66, 132, 220, 400 |
| rango | TEXT | MT / AT |

### `nivel_aislamiento` — niveles normalizados (Um, RD 223/2008)

| Columna | Tipo | Descripción |
|---|---|---|
| id | PK | |
| valor_kv | NUMERIC | 3.6, 7.2, 12, 17.5, 24, 36, 52, 72.5, 123, 145, 245, 420 |

---

## Tablas

### `activo_red` — tabla base (identidad)

Tabla de **identidad**, deliberadamente delgada. **Sin columnas PostGIS.** No es
instanciable por sí sola: todo activo tiene una especialización. Su valor es ser
el ancla común de terminal, geometría, titularidad (BDDAT) y contención.

| Columna       | Tipo               | Descripción                          |
|---------------|--------------------|--------------------------------------|
| id            | UUID PK            | Gancho de referencia para BDDAT      |
| nombre        | TEXT               | Denominación del activo              |
| envolvente_id | UUID FK(activo_red)| Contenedor (nullable). Self-FK       |
| notas         | TEXT               | Texto libre para datos ocasionales   |

> No lleva `tipo` (derivable de la subtabla), ni tensión (cardinalidad variable:
> el trafo tiene varias), ni estado (lo infiere BDDAT), ni titularidad (tabla
> propia con histórico en BDDAT). Ver [ADR-006](../decisiones/ADR-006-criterios-modelado.md).

### `envolvente` — contenedor (recinto), anidable

| Columna   | Tipo       | Descripción                                                       |
|-----------|------------|-------------------------------------------------------------------|
| activo_id | UUID PK/FK | |
| tipo      | TEXT       | centro_transformacion / subestacion / posicion / armario_seccionamiento / celda_prefabricada |

> El nombre describe; el `tipo` categoriza (lo usa el descriptor para la palabra:
> "Subestación", "Posición de línea"). La contención (`envolvente_id`) la puede
> ejercer cualquier activo con ubicación, no solo las envolventes (un apoyo
> contiene la aparamenta que monta).

### `linea` — conductores aéreos y subterráneos

| Columna | Tipo | Descripción |
|---|---|---|
| activo_id | UUID PK/FK | |
| tipo | TEXT | aerea / subterranea |
| tension_id | FK → tension | tensión de **servicio** |
| nivel_aislamiento | FK → nivel_aislamiento | **capacidad** (Um); puede diferir del servicio |
| conductor | TEXT | texto plano (tipología de cables a sesión propia) |
| conductores_por_fase | INTEGER | haces dúplex/tríplex/cuádruplex |
| longitud_m | NUMERIC | no derivable de la traza 2D (flecha/zanja) |
| obra_civil_descripcion | TEXT | arquetas, canalizaciones, cámaras ("12 arquetas A1…") |

> El aislamiento (desnudo/aislado) es propiedad del conductor → sesión de cables.
> Doble circuito = **dos líneas** (independencia de ciclo de vida). 2 terminales.

### `elemento_corte` — aparamenta de corte y maniobra

| Columna | Tipo | Descripción |
|---|---|---|
| activo_id | UUID PK/FK | |
| funcion | TEXT | linea / proteccion / acople / medida / remonte / seccionamiento / servicios_auxiliares |
| aparato | TEXT | seccionador / interruptor / interruptor_automatico / reconectador / ruptofusible / cortacircuitos_fusible |
| tecnologia | TEXT | AIS / GIS (medio aislante) |
| nivel_aislamiento | FK → nivel_aislamiento | **obligatorio** (no tiene servicio del que derivarlo) |
| accionamiento | TEXT | manual / motorizado / telecontrolado |
| intensidad_nominal_a | NUMERIC | |
| poder_corte_ka | NUMERIC | nullable (≈0 en seccionadores) |
| calibre_fusible_a | NUMERIC | nullable (solo con fusible) |
| tipo_fusible | TEXT | expulsion / limitador (nullable) |

> `función` = rol de la celda; `aparato` = tecnología de corte (ejes ortogonales).
> `interruptor_automatico` ⊃ relé (implícito). La tensión de **servicio** se hereda
> del embarrado/nodo. 2 terminales.

### `transformador`

| Columna | Tipo | Descripción |
|---|---|---|
| activo_id | UUID PK/FK | |
| tension_primario_id | FK → tension | lado de mayor tensión |
| tension_secundario_id | FK → tension | lado de menor tensión |
| potencia_kva | NUMERIC | valor ONAN (el menor); ampliación por migración aditiva |
| grupo_vector | TEXT | Dyn11, YNyn0… |
| refrigeracion | TEXT | seco / ONAN / ONAF / OFAF |

> 3 devanados = **2 trafos + embarrado** (caso excepcional, AT). 2 terminales.
> No lleva nivel de aislamiento (no está sobreaislado en general; se asume el de
> cada tensión).

### `embarrado`

| Columna | Tipo | Descripción |
|---|---|---|
| activo_id | UUID PK/FK | |
| tension_id | FK → tension | tensión de servicio (define la del nodo-barra) |
| configuracion | TEXT | simple / partida / doble / triple |

> Doble barra = **2 embarrados** + celda de acople. La microtopología de barras no
> se modela (es operación, no autorización). 1 terminal, multiconexión. No lleva
> aislamiento (solo ve los electrones que pasan; lo definen las celdas/aisladores).
> Intensidad máxima → `notas` cuando excepcionalmente sea limitación de diseño.

### `elemento_medida` — TI/TT individualizados (solo AIS/GIS)

| Columna | Tipo | Descripción |
|---|---|---|
| activo_id | UUID PK/FK | |
| tipo | TEXT | transformador_intensidad / transformador_tension / contador / analizador |
| relacion_primario | NUMERIC | 600 (A) ó 220000 (V) según tipo |
| relacion_secundario | NUMERIC | 5 (A) ó 110 (V) |
| clase_precision | TEXT | 0.2 / 0.5 / 5P… |

> En MT la medida va **integrada en la celda** (`elemento_corte`, función=medida);
> esta tabla es para los TI/TT sueltos de subestaciones AIS/GIS. 0 terminales,
> contenido en la posición.

### `pararrayos` — protección contra sobretensiones

| Columna | Tipo | Descripción |
|---|---|---|
| activo_id | UUID PK/FK | |
| tension_nominal_kv | NUMERIC | Ur |
| corriente_descarga_ka | NUMERIC | 5 / 10 / 20 |
| clase | TEXT | distribucion / estacion |

> Tecnología ZnO casi universal (no se modela). 0 terminales, contenido en envolvente.

### `red_tierra` — protección contra defectos a tierra

| Columna | Tipo | Descripción |
|---|---|---|
| activo_id | UUID PK/FK | |
| rol | TEXT | proteccion / servicio |
| resistencia_ohm | NUMERIC | para tensiones de contacto/paso en defecto |
| distancia_separacion_m | NUMERIC | separación entre redes; solo en la fila de servicio (nullable) |
| descripcion | TEXT | picas, distancias, sección, material |

> Una o dos redes separativas según haya neutro a tierra (transformación). Cada
> envolvente (subestación, CT de SSAA) lleva las suyas. 0 terminales.

### `apoyo` — sustentación de líneas aéreas

| Columna | Tipo | Descripción |
|---|---|---|
| activo_id | UUID PK/FK | |
| funcion | TEXT | alineacion / angulo / anclaje / fin_de_linea / derivacion |
| material | TEXT | texto libre (metálico celosía galvanizado…) |
| altura_m | NUMERIC | nullable |
| esfuerzo_dan | NUMERIC | esfuerzo nominal en punta; nullable |

> 0 terminales (estructura mecánica). Puede **contener** la aparamenta que monta
> (seccionadores, pararrayos, empalmes) vía `envolvente_id`.

### `empalme` — unión de conductores

| Columna | Tipo | Descripción |
|---|---|---|
| activo_id | UUID PK/FK | |
| tipo | TEXT | recto / derivacion / terminal_botella / transicion_aero_subt |

> 1 terminal, **multiconexión** ("embarrado exclusivo de líneas"): habilita que en
> su nodo concurran N líneas. Geometría propia; sin tensión propia (la hereda).

### `nodo` — punto de conectividad (virtual)

| Columna | Tipo | Descripción |
|---|---|---|
| id | UUID PK | |
| nombre | TEXT | referencia |
| tension_id | FK → tension | tensión de servicio; la fija el primer activo con tensión propia que conecta (nullable hasta entonces) |

> Virtual: no tiene geometría propia. Su comportamiento "bus" es emergente (lo
> habilita un embarrado o empalme conectado), no un atributo.

### `terminal` — conexión de un activo a un nodo

| Columna | Tipo | Descripción |
|---|---|---|
| id | UUID PK | |
| activo_id | UUID FK → activo_red | activo al que pertenece |
| nodo_id | UUID FK → nodo | nodo al que conecta |
| secuencia | INTEGER | distingue terminales del activo (p.ej. primario/secundario del trafo) |

---

## Geometría (PostGIS aislado y compartible)

Ver [ADR-002](../decisiones/ADR-002-herencia-tablas.md). La geometría es entidad
propia reutilizable; el enlace a activos es indirecto y compartible (un apoyo y un
empalme colocalizados comparten una misma `geometria_id`).

### `geometria` — única tabla con PostGIS

| Columna | Tipo | Descripción |
|---|---|---|
| id | UUID PK | |
| geom | GEOMETRY(Geometry, 4326) | **única columna PostGIS del modelo** |
| tipo_geom | TEXT | punto / linea / poligono |
| srid_origen | INTEGER | SRID del shape recibido (ej. 25830) |
| fuente | TEXT | titular / digitalizado / gps / importado |

### `activo_geometria` — enlace (SQL puro, portable)

| Columna | Tipo | Descripción |
|---|---|---|
| activo_id | UUID FK → activo_red | |
| geometria_id | UUID FK → geometria | compartible entre activos |

**Coordenadas:** almacenamiento en **WGS84 (4326)**; exportación a la Junta de
Andalucía en **ETRS89 UTM30N (25830)** mediante `ST_Transform`. Entrada de shapes
del titular vía `ogr2ogr`/GeoPandas.

---

## Reglas de conectividad

Resumen; el detalle y las invariantes R1–R8 están en
[ADR-001](../decisiones/ADR-001-topologia-terminal-nodo.md).

| Activo | Terminales | Multiconexión |
|---|---|---|
| linea, elemento_corte, transformador | 2 | no |
| embarrado, empalme | 1 | **sí (N)** |
| apoyo, elemento_medida, pararrayos, red_tierra, envolvente | 0 | — |

- Terminales = 0 ⟺ no conductor, fuera del grafo, vinculado por **contención**.
- Misma tensión de servicio en todos los terminales de un nodo (R1).
- `nivel_aislamiento` del activo ≥ tensión de servicio del nodo (R8).
- Multiconexión (>2 terminales en un nodo) la habilita un embarrado o un empalme (R5).

---

## Notas de integración GIS

- La topología eléctrica (qué afecta a qué) se recorre por `terminal` + `nodo`, no por GIS.
- Los cálculos espaciales (solapamientos, cruces, extensión por municipio) operan
  sobre `geometria` sin tocar el núcleo del modelo.
- `activo_red.envolvente_id` construye la jerarquía CT/subestación → posición → aparamenta.

---

## Pendiente de definir

- **Generación** (`elemento_generacion`, `subcampo_fv`…): pendiente de revisión
  bajo los criterios consolidados (ver `modelo-datos-generacion.md`).
- **Agrupaciones lógicas** sin recinto físico (p. ej. "línea MT-15 y todos sus
  apoyos a lo largo de 30 km"): sesión transversal pendiente.
- Tipología de **conductores** y de **aisladores**: sesión propia.
- Integración con BDDAT: titularidad (con histórico), expediente, estado
  administrativo (referencian al `activo_red.id`; ver ADR-000).
