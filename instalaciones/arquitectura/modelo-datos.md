# Modelo de datos — Instalaciones eléctricas de alta tensión

## Referencia

La estructura propuesta es coherente con el estándar **IEC CIM 61970/61968**
(Common Information Model), que define cómo modelar redes eléctricas de forma
interoperable. Se ha simplificado para el propósito de BDDAT, manteniendo los
conceptos clave de topología Terminal–Nodo.

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
