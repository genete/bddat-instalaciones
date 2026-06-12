# ADR-002 — Herencia de tablas: tabla base + tablas específicas

**Estado:** Propuesto  
**Fecha:** 2026-06-12

## Contexto

Todos los elementos eléctricos comparten atributos comunes (id, nombre,
tensión nominal, estado, coordenadas). Pero cada tipo tiene atributos propios
muy distintos.

## Decisión

Se usa el patrón **Class Table Inheritance** (herencia por tabla de clase):

- Una tabla `elemento` con los atributos comunes.
- Una tabla por tipo (`linea`, `transformador`, etc.) con FK a `elemento.id`
  y sus atributos específicos.

En PostgreSQL se puede implementar con herencia nativa (`INHERITS`) o con FK.
Se recomienda **FK simple** para mejor compatibilidad con ORMs y herramientas GIS.

## Alternativas descartadas

| Alternativa | Problema |
|---|---|
| Tabla única con columnas nullable por tipo (Single Table Inheritance) | Demasiadas columnas null; difícil de mantener con tipos muy distintos |
| Una tabla por tipo sin tabla base | No hay punto único para la topología; `terminal.elemento_id` necesitaría una FK polimórfica |

## Consecuencias

- La tabla `terminal` puede tener FK a `elemento.id` de forma limpia.
- Consultas de todos los elementos: JOIN entre `elemento` y cada subtabla.
- En PostgreSQL se puede crear una vista por tipo para simplificar acceso.
