# Modelo de datos — Instalaciones eléctricas MT/AT (> 1 kV)

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
proyecto. El modelo debe soportar ambos:

| Rango          | Denominación oficial   | Gestor principal | Tensiones habituales              |
|----------------|------------------------|------------------|-----------------------------------|
| 1 kV – 36 kV   | **Media Tensión (MT)** | eDistribución    | 15 kV (legado), **20 kV** (nuevo) |
| > 36 kV        | **Alta Tensión (AT)**  | REE / Redeia     | 66 kV, 132 kV, **220 kV**, **400 kV** |

> La zona gris son los 45 kV y 66 kV de subtransporte: pueden ser de REE o de
> eDistribución según el caso. El campo `tension_nominal_kv` del modelo cubre
> todos los rangos sin distinción estructural.

---

### Red Eléctrica de España (REE / Redeia) — particularidades

REE es el único operador del sistema de transporte (TSO) en España. Gestiona la
red AT (principalmente 220 kV y 400 kV) y es responsable de la operación del
sistema eléctrico nacional.

**Conclusión relevante para el modelo:** REE **no se desvía del estándar IEC CIM**.
Al contrario, lo refuerza: intercambia modelos de red con ENTSO-E usando **CGMES**
(Common Grid Model Exchange Standard), que es un perfil europeo directamente
derivado de IEC CIM 61970. El modelo de tablas propuesto es por tanto compatible
con REE sin cambios estructurales.

#### Particularidades técnicas que sí afectan al modelo

REE publica requisitos mínimos para instalaciones conectadas a su red de transporte
(documento TI.E_02_040). Estos requisitos añaden campos que no son necesarios en MT:

| Requisito REE                                  | Impacto en el modelo                                      |
|------------------------------------------------|-----------------------------------------------------------|
| Doble sistema de protección con baterías independientes | Campo en `elemento_proteccion`: `num_sistemas_proteccion` |
| Interruptores en ambos lados de la interconexión | Regla topológica: elemento_corte siempre en frontera AT   |
| Requisitos de automatización/SCADA/telecontrol | Campo `telecontrolado` ya contemplado en `elemento_corte` |
| Coordinación de aislamiento específica por nivel de tensión | Campo `nivel_aislamiento_kv` en `elemento`                |
| Red de tierra de alta disipación               | Ya contemplado en `elemento_proteccion` subtipo red_tierra |

#### Normativa REE de referencia

| Documento                        | Contenido                                                  |
|----------------------------------|------------------------------------------------------------|
| **TI.E_02_040**                  | Condiciones técnicas conexión terceros a red de transporte |
| **Requisitos mínimos diseño Ed4**| Equipamiento mínimo instalaciones conectadas a REE         |
| **ITC-RAT 01–23 (RD 337/2014)** | Reglamento AT nacional, base legal común con eDistribución |
| **CGMES (ENTSO-E)**              | Perfil CIM para intercambio de modelos de red en Europa    |

Fuentes:
- [Condiciones técnicas conexión a red de transporte — REE (PDF)](https://www.ree.es/sites/default/files/01_ACTIVIDADES/Documentos/AccesoRed/TI.E_02_040_Ed5_Cond_Tecnic_Conex_Terceros_RdT_Peninsular.pdf)
- [Requisitos mínimos diseño y equipamiento Ed4 — REE (PDF)](https://www.ree.es/sites/default/files/12_CLIENTES/Documentos/Instalaciones_conectadas_a_la_red_de_transporte_Requisitos_minimos_dise%C3%B1o_equipamiento_Ed4.pdf)
- [RD 337/2014 ITC-RAT — BOE](https://www.boe.es/buscar/doc.php?id=BOE-A-2014-6084)
- [Acuerdo eDistribución–REE para instalaciones en frontera (PDF)](https://www.edistribucion.com/content/dam/edistribucion/conexion-a-la-red/normativa/Acuerdo_EDRD_REE.pdf)
- [CGMES conformance — DIgSILENT Ibérica](http://www.digsilentiberica.es/noticia/34/ENTSO-e-acredita-la-conformidad-de-DIgSILENT-PowerFactory-con-CGMES/)

---

### Terminología eDistribución (vs denominaciones genéricas)

| Este modelo         | Denominación eDistribución / sector       |
|---------------------|-------------------------------------------|
| `linea` aérea MT    | LAMT (Línea Aérea de Media Tensión)       |
| `linea` subterránea MT | LSMT (Línea Subterránea de Media Tensión) |
| `envolvente` CT     | CT (Centro de Transformación)             |
| `envolvente` CS     | CS (Centro de Seccionamiento)             |
| `elemento_corte`    | Celda de línea / celda de seccionamiento  |
| `transformador` en CT | Transformador MT/BT                     |

---

## Referencia técnica internacional

La estructura de tablas es coherente con el estándar **IEC CIM 61970/61968**
(Common Information Model), que define cómo modelar redes eléctricas de forma
interoperable. Se ha simplificado para el propósito de BDDAT, manteniendo los
conceptos clave de topología Terminal–Nodo.

- [Common Information Model (electricity) — Wikipedia](https://en.wikipedia.org/wiki/Common_Information_Model_(electricity))
- [smart-data-models/dataModel.EnergyCIM — GitHub](https://github.com/smart-data-models/dataModel.EnergyCIM)
- [ConnectivityNode — CIM Datamodel (Zepben)](https://zepben.github.io/evolve/docs/cim/ewb/IEC61970/Base/Core/ConnectivityNode/)
- [ACLineSegment — CIM Schema (LANL)](https://lanl-ansi.github.io/MG-RAVENS/_static/schema/ACLineSegment.html)
- [La información geográfica en redes de distribución eléctrica — Geoinnova](https://geoinnova.org/blog-territorio/la-informacion-geografica-en-las-redes-de-distribucion-de-energia-electrica/)

---

## Principio de topología (de CIM)

La conectividad eléctrica se modela con dos conceptos:

- **`nodo`** (ConnectivityNode): punto eléctrico donde se unen conductores.
  No es un elemento físico, es el "cable invisible" que une terminales.
- **`terminal`**: extremo de conexión de un elemento. Cada elemento tiene
  tantos terminales como puntos de conexión (una línea tiene 2, un
  transformador tiene 2 o 3, un embarrado tiene N).

Un elemento A y un elemento B están conectados cuando comparten el mismo `nodo`
a través de sus respectivos `terminal`.

---

## Diagrama de tablas

```
elemento (base)
  ├── linea
  ├── elemento_corte
  ├── transformador
  ├── elemento_medida
  ├── elemento_proteccion
  ├── envolvente          ←── otros elementos pueden tener FK a envolvente
  ├── soporte
  └── embarrado

elemento → terminal → nodo   (topología)
```

---

## Tablas

### `elemento` — tabla base para todos los activos

| Columna           | Tipo                | Descripción                                      |
|-------------------|---------------------|--------------------------------------------------|
| id                | UUID PK             |                                                  |
| nombre            | TEXT                | Denominación del elemento                        |
| tipo_elemento     | TEXT (enum)         | linea / corte / transformador / medida / proteccion / envolvente / soporte / embarrado |
| tension_nominal_kv| NUMERIC             | Tensión nominal de operación en kV               |
| estado            | TEXT                | en_servicio / fuera_servicio / en_construccion   |
| gestor_red        | TEXT                | edistribucion / ree / privado / otro             |
| nivel_aislamiento_kv | NUMERIC          | Nivel de aislamiento (relevante en AT/REE). Nullable |
| envolvente_id     | UUID FK (elemento)  | Envolvente que lo contiene (nullable)            |
| geom              | GEOMETRY(Point)     | Coordenada del elemento (PostGIS). Nullable para líneas |
| notas             | TEXT                |                                                  |

---

### `linea` — conductores aéreos, subterráneos, cables aislados

Hereda de `elemento` (FK `elemento_id`).

| Columna           | Tipo       | Descripción                                      |
|-------------------|------------|--------------------------------------------------|
| elemento_id       | UUID PK/FK |                                                  |
| subtipo           | TEXT       | aerea / subterranea / cable_aislado / conductor_desnudo |
| conductor_material| TEXT       | aluminio / cobre / ACSR / etc.                   |
| seccion_mm2       | NUMERIC    | Sección del conductor en mm²                     |
| longitud_m        | NUMERIC    | Longitud en metros                               |
| num_circuitos     | INTEGER    | Número de circuitos (terna, doble circuito...)   |
| geom_traza        | GEOMETRY(LineString) | Traza geográfica de la línea (PostGIS)  |

---

### `elemento_corte` — seccionadores, interruptores, reconectadores

Hereda de `elemento`.

| Columna           | Tipo       | Descripción                                      |
|-------------------|------------|--------------------------------------------------|
| elemento_id       | UUID PK/FK |                                                  |
| subtipo           | TEXT       | seccionador / interruptor / interruptor_automatico / reconectador / fusible_corte |
| accionamiento     | TEXT       | manual / motorizado / telecontrolado             |
| estado_normal     | TEXT       | normalmente_abierto / normalmente_cerrado        |
| intensidad_nominal_a | NUMERIC | Intensidad nominal en A                         |
| poder_corte_ka    | NUMERIC    | Poder de corte en kA (para interruptores)        |

---

### `transformador` — transformadores de potencia y distribución

Hereda de `elemento`.

| Columna              | Tipo       | Descripción                                   |
|----------------------|------------|-----------------------------------------------|
| elemento_id          | UUID PK/FK |                                               |
| potencia_kva         | NUMERIC    | Potencia nominal en kVA                       |
| tension_primario_kv  | NUMERIC    | Tensión en el lado de alta                    |
| tension_secundario_kv| NUMERIC    | Tensión en el lado de baja                    |
| tension_terciario_kv | NUMERIC    | Nullable, para transformadores de 3 devanados |
| grupo_vector         | TEXT       | Grupo de conexión (ej: Dyn11, YNd1)          |
| tipo_refrigeracion   | TEXT       | ONAN / ONAF / OFAF / seco                    |
| num_terminales       | INTEGER    | 2 o 3 (según devanados)                      |

---

### `elemento_medida` — transformadores de medida (TI, TT) y contadores

Hereda de `elemento`.

| Columna              | Tipo       | Descripción                                   |
|----------------------|------------|-----------------------------------------------|
| elemento_id          | UUID PK/FK |                                               |
| subtipo              | TEXT       | transformador_intensidad / transformador_tension / contador / analizador |
| relacion_primario    | NUMERIC    | Ej: 200 (para TI 200/5)                       |
| relacion_secundario  | NUMERIC    | Ej: 5                                         |
| clase_precision      | TEXT       | 0.2 / 0.5 / 1 / 3 / 5P / 10P                |

---

### `elemento_proteccion` — fusibles, pararrayos, redes de tierra

Hereda de `elemento`.

| Columna              | Tipo       | Descripción                                   |
|----------------------|------------|-----------------------------------------------|
| elemento_id          | UUID PK/FK |                                               |
| subtipo              | TEXT       | fusible / pararrayos / descargador_sobretension / red_tierra / rele |
| calibre_a            | NUMERIC    | Intensidad nominal / fusión (nullable)        |
| tension_descarga_kv  | NUMERIC    | Para pararrayos/descargadores (nullable)      |
| resistencia_tierra_ohm | NUMERIC  | Para redes de tierra (nullable)               |
| num_sistemas_proteccion | INTEGER | Sistemas de protección redundantes (REE exige 2 en AT) |

---

### `envolvente` — contenedores de elementos

Hereda de `elemento`.

| Columna              | Tipo       | Descripción                                   |
|----------------------|------------|-----------------------------------------------|
| elemento_id          | UUID PK/FK |                                               |
| subtipo              | TEXT       | centro_transformacion / subestacion / armario_seccionamiento / celda_prefabricada |
| propietario          | TEXT       | Propietario/gestor de la instalación          |
| acceso               | TEXT       | publico / privado / restringido               |

---

### `soporte` — apoyos, arquetas, canalizaciones, empalmes

Hereda de `elemento`.

| Columna              | Tipo       | Descripción                                   |
|----------------------|------------|-----------------------------------------------|
| elemento_id          | UUID PK/FK |                                               |
| subtipo              | TEXT       | apoyo / empalme / arqueta / canalizacion / herraje |
| material             | TEXT       | hormigon / metalico / madera / fibra          |
| altura_m             | NUMERIC    | Altura del apoyo en metros (nullable)         |

---

### `embarrado` — conjunto de barras de igual tensión

Hereda de `elemento`.

| Columna              | Tipo       | Descripción                                   |
|----------------------|------------|-----------------------------------------------|
| elemento_id          | UUID PK/FK |                                               |
| intensidad_maxima_a  | NUMERIC    | Corriente máxima admisible                    |
| num_barras           | INTEGER    | Número de barras del sistema (simple/doble)   |

---

### `nodo` — nodo de conectividad eléctrica (ConnectivityNode en CIM)

No es un elemento físico: es el punto lógico donde varios terminales confluyen.

| Columna              | Tipo             | Descripción                               |
|----------------------|------------------|-------------------------------------------|
| id                   | UUID PK          |                                           |
| nombre               | TEXT             | Referencia (ej: "Nodo_AT_CT_Ejemplo")     |
| tension_nominal_kv   | NUMERIC          |                                           |
| geom                 | GEOMETRY(Point)  | Coordenada aproximada (PostGIS, nullable) |

---

### `terminal` — punto de conexión de un elemento a la red

| Columna              | Tipo       | Descripción                                   |
|----------------------|------------|-----------------------------------------------|
| id                   | UUID PK    |                                               |
| elemento_id          | UUID FK    | Elemento al que pertenece este terminal       |
| nodo_id              | UUID FK    | Nodo al que está conectado                    |
| secuencia            | INTEGER    | 1=lado_alta, 2=lado_baja, 3=terciario (para transformadores) |

---

## Reglas de conectividad

| Tipo de elemento   | Num. terminales | Cómo conecta                                              |
|--------------------|-----------------|-----------------------------------------------------------|
| Línea              | 2               | Terminal 1 y 2 a nodos distintos (inicio y fin de tramo)  |
| Elemento de corte  | 2               | Terminal 1 y 2 al embarrado o a nodos de la red           |
| Transformador      | 2 (o 3)         | Terminal 1 = alta tensión, terminal 2 = baja tensión      |
| Embarrado          | N               | Cada terminal conecta un elemento de corte al embarrado   |
| Soporte / Empalme  | 1–2             | Punto de paso de la línea; puede ser nodo intermedio      |
| Medida / Protección| —               | Sin terminales propios: están contenidos en `envolvente`  |

---

## Notas de integración GIS

- Usar **PostGIS** con SRID 4326 (WGS84) para coordenadas en `geom` y `geom_traza`.
- La topología completa de la red puede exportarse como grafo para análisis
  con **pgRouting** o herramientas GIS externas (QGIS, ArcGIS).
- `elemento.envolvente_id` permite agrupar elementos físicamente
  y construir la jerarquía CT → celda → transformador.

---

## Pendiente de definir

- Características técnicas detalladas por tipo (catálogos de cables, aisladores, etc.)
- Histórico de estados y mantenimientos (tabla `evento_elemento`)
- Documentación asociada (tabla `documento_elemento`)
- Integración con el modelo de datos de BDDAT (referencias cruzadas)
