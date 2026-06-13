# ADR-002 — Estructura de tablas: activo_red como base, FK simple, GIS aislado

**Estado:** Aceptado
**Fecha:** 2026-06-12
**Revisado:** 2026-06-13

## Contexto

Todos los activos de la red comparten atributos comunes (id, nombre, tensión
nominal, estado, gestor). Cada tipo tiene atributos propios muy distintos.
Además, los activos tienen una dimensión geográfica obligatoria (los titulares
presentan shapes en las solicitudes y la administración los requiere para estudios
de afecciones). El proyecto BDDAT tiene como principio no atarse a características
exclusivas de PostgreSQL.

---

## Decisión 1 — Nombre de la tabla base: `activo_red`

La tabla base se llama **`activo_red`**, no `elemento`.

**Por qué:** los soportes (apoyos, arquetas, canalizaciones) no son elementos
eléctricos — no conducen electricidad. Llamar `elemento` a la tabla base implica
que todos los activos son eléctricos, lo cual es incorrecto. `activo_red` agrupa
todos los activos físicos de la red (conductores y no conductores) bajo un
término neutro.

El campo `tipo_elemento` ya distingue qué tipos conducen y cuáles no. Las reglas
de integridad (terminales = 0 para apoyos de alineación, arquetas, etc.) viven
en el backend (ver ADR-001).

---

## Decisión 2 — Herencia por FK simple (Class Table Inheritance)

Se usa el patrón **Class Table Inheritance**:

- Una tabla `activo_red` con los atributos comunes.
- Una tabla por tipo (`linea`, `transformador`, etc.) con FK a `activo_red.id`
  y sus atributos específicos.

**Se prohíbe explícitamente el uso de `INHERITS`** (herencia nativa de
PostgreSQL). Aunque PostgreSQL lo ofrece, es una característica exclusiva que
impide migrar el esquema a cualquier otro motor relacional.

La FK simple es **SQL estándar** y funciona en cualquier RDBMS (MySQL,
SQLite, Oracle, SQL Server, etc.).

## Alternativas descartadas

| Alternativa | Problema |
|---|---|
| `INHERITS` de PostgreSQL | Lock-in total; imposible migrar el esquema a otro motor |
| Single Table Inheritance (una tabla con todas las columnas nullable) | Demasiadas columnas null; difícil de mantener con tipos muy distintos |
| Una tabla por tipo sin tabla base | No hay punto único para la topología; `terminal.elemento_id` necesitaría FK polimórfica |

---

## Decisión 3 — PostGIS aislado en tabla lateral `geometria_elemento`

La tabla `activo_red` **no contiene ninguna columna PostGIS**. La geometría
vive en una tabla lateral dedicada:

```sql
CREATE TABLE geometria_elemento (
  elemento_id  UUID PRIMARY KEY REFERENCES activo_red(id),
  geom         GEOMETRY(Geometry, 4326),   -- punto, línea o polígono
  tipo_geom    TEXT,                        -- punto / linea / poligono
  srid_origen  INTEGER,                     -- SRID del shape recibido
  fuente       TEXT                         -- titular / digitalizado / gps
);
```

**Por qué:** el proyecto BDDAT evita por principio atarse a características
exclusivas de PostgreSQL. PostGIS es exclusivo de PostgreSQL. Aislarlo en una
tabla propia garantiza que el núcleo del modelo (`activo_red` y todas sus
subtablas) es **completamente portable**.

### Escenario de migración

Si en el futuro se requiere migrar a otro motor:

1. `activo_red` y subtablas migran sin modificación alguna.
2. `geometria_elemento` se exporta en un paso con herramientas estándar:

```bash
# A GeoPackage (formato OGC, abierto)
ogr2ogr -f GPKG instalaciones.gpkg PG:"dbname=bddat" geometria_elemento

# A Shapefile (para administraciones)
ogr2ogr -f "ESRI Shapefile" instalaciones.shp PG:"dbname=bddat" geometria_elemento
```

3. El sistema destino importa desde GeoPackage o Shapefile.

La traducción entre PostGIS y cualquier formato GIS estándar (Shapefile, GeoPackage,
GeoJSON, KML) es bidireccional y sin pérdida mediante `ogr2ogr` (GDAL) o
GeoPandas. No hay riesgo de pérdida de datos geográficos.

### Shapes de los titulares

Los titulares presentan shapes en las solicitudes (obligatorio para estudios
de afecciones en Medio Ambiente y Ordenación del Territorio). El flujo es:

```
Shape del titular (.shp / .gpkg)
        ↓  ogr2ogr / GeoPandas
geometria_elemento (PostGIS)
        ↓  ST_Transform(geom, 25830)
Exportación ETRS89/UTM30N para la Junta de Andalucía
```

### Nota sobre sistemas de referencia

- Almacenamiento interno: **WGS84 (SRID 4326)** — estándar web y GPS.
- Exportación a administraciones: **ETRS89 / UTM zona 30N (SRID 25830)** — exigido por la Junta de Andalucía.
- La conversión se hace con `ST_Transform` al exportar; no es necesario almacenar ambos.

---

## Consecuencias

- `activo_red` y todas las subtablas son SQL estándar y portables a cualquier RDBMS.
- `geometria_elemento` es la única tabla con dependencia de PostGIS.
- Un activo puede existir sin geometría (durante tramitación) y recibirla después.
- Las consultas espaciales (solapamientos, cruces, extensiones por municipio) se
  ejecutan sobre `geometria_elemento` sin afectar al resto del modelo.
- El lock-in geográfico está acotado y documentado; la estrategia de salida es conocida.
