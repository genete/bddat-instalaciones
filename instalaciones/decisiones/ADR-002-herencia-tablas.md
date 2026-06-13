# ADR-002 — Estructura de tablas: activo_red como base, FK simple, GIS aislado

**Estado:** Aceptado
**Fecha:** 2026-06-12
**Revisado:** 2026-06-13

## Contexto

Los activos de la red comparten una **identidad común** (id, nombre, contenedor)
pero cada tipo tiene atributos propios muy distintos. Además tienen una dimensión
geográfica (los titulares presentan shapes en las solicitudes; la administración
los requiere para estudios de afecciones). BDDAT tiene como principio **no atarse a
características exclusivas de PostgreSQL**.

---

## Decisión 1 — Nombre de la tabla base: `activo_red`

La tabla base se llama **`activo_red`**, no `elemento`.

**Por qué:** los apoyos, arquetas y canalizaciones no son elementos eléctricos
(no conducen). Llamar `elemento` a la base implicaría que todos los activos son
eléctricos. `activo_red` agrupa todos los activos físicos de la red (conductores y
no conductores) bajo un término neutro.

La base **no lleva** `tipo` (es derivable de la subtabla donde existe el activo;
ver ADR-006). Las reglas de cuántos terminales tiene cada tipo viven en el backend
(ver ADR-001).

---

## Decisión 2 — Herencia por FK simple (Class Table Inheritance)

- Una tabla `activo_red` con la **identidad** (id, nombre, envolvente_id, notas).
- Una tabla por tipo (`linea`, `transformador`, etc.) con FK `activo_id → activo_red.id`
  y sus atributos específicos.

**Se prohíbe explícitamente `INHERITS`** (herencia nativa de PostgreSQL): aunque
PostgreSQL lo ofrece, impide migrar el esquema a otro motor. La FK simple es **SQL
estándar** y funciona en cualquier RDBMS.

La tabla base, aunque delgada, es necesaria: es el **ancla de identidad** al que
apuntan `terminal`, `activo_geometria`, la titularidad (BDDAT) y la contención
(`envolvente_id`). Sin ella, esas relaciones serían FK polimórficas (tipo + id),
que en SQL estándar pierden integridad referencial.

## Alternativas descartadas

| Alternativa | Problema |
|---|---|
| `INHERITS` de PostgreSQL | Lock-in total |
| Single Table Inheritance (todo en una tabla con columnas nullable) | Demasiados null; tipos muy distintos |
| Una tabla por tipo sin base | `terminal.activo_id` necesitaría FK polimórfica; sin ancla de identidad |

---

## Decisión 3 — PostGIS aislado: `geometria` + `activo_geometria`

`activo_red` **no contiene ninguna columna PostGIS**. La geometría es una **entidad
propia reutilizable**, y la asociación con los activos va en una **tabla de enlace
separada** (compartible):

```sql
-- Geometría como entidad propia (única tabla con PostGIS)
CREATE TABLE geometria (
  id           UUID PRIMARY KEY,
  geom         GEOMETRY(Geometry, 4326),   -- punto, línea o polígono
  tipo_geom    TEXT,                        -- punto / linea / poligono
  srid_origen  INTEGER,                     -- SRID del shape recibido
  fuente       TEXT                         -- titular / digitalizado / gps / importado
);

-- Enlace activo ↔ geometría (SQL puro, portable; permite compartir)
CREATE TABLE activo_geometria (
  activo_id    UUID REFERENCES activo_red(id),
  geometria_id UUID REFERENCES geometria(id)
);
```

**Por qué separar geometría y enlace:**
- La **geometría se define una vez** y puede ser **compartida** por varios activos
  (un apoyo y el empalme montado sobre él referencian la misma `geometria_id`, sin
  duplicar coordenadas). Mover esa geometría mueve a todos los que la comparten —
  coherencia deseada; si uno debe separarse, se le crea una geometría propia.
- El **enlace `activo_geometria` es SQL puro y portable**; solo `geometria` lleva el
  tipo PostGIS. La frontera de portabilidad queda nítida.

### Escenario de migración

1. `activo_red`, todas las subtablas y `activo_geometria` migran **sin tocar nada**.
2. `geometria` se exporta en un paso:

```bash
ogr2ogr -f GPKG instalaciones.gpkg PG:"dbname=bddat" geometria          # GeoPackage
ogr2ogr -f "ESRI Shapefile" instalaciones.shp PG:"dbname=bddat" geometria  # Shapefile
```

3. El sistema destino importa el GeoPackage/Shapefile.

Traducción bidireccional y sin pérdida con `ogr2ogr` (GDAL) o GeoPandas.

### Shapes de los titulares y sistemas de referencia

```
Shape titular (.shp/.gpkg) → ogr2ogr/GeoPandas → geometria (PostGIS)
geometria → ST_Transform(geom, 25830) → exportación a la Junta de Andalucía
```

- Almacenamiento: **WGS84 (SRID 4326)**.
- Exportación a administraciones: **ETRS89 / UTM30N (SRID 25830)** mediante `ST_Transform`.

---

## Consecuencias

- `activo_red`, subtablas y `activo_geometria` son SQL estándar y portables.
- `geometria` es la **única** tabla con dependencia de PostGIS.
- Un activo puede existir **sin geometría** (durante tramitación) y recibirla después.
- Varios activos pueden **compartir** una geometría sin duplicar coordenadas.
- El lock-in geográfico está acotado y documentado; la estrategia de salida es conocida.
