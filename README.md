# bddat-instalaciones

Estudio de la **estructura de datos técnica** de las instalaciones eléctricas de
alta tensión (MT y AT, > 1 kV) para el proyecto [BDDAT](https://github.com/genete/BDDAT).

Repositorio independiente del principal para no interferir en el desarrollo diario.
Modela **solo la parte técnica** de los activos; el expediente, la tramitación y el
estado administrativo pertenecen a BDDAT (ver ADR-000).

## Estructura

```
instalaciones/
  arquitectura/
    modelo-datos.md              # tablas de red, topología, geometría, catálogos
    modelo-datos-generacion.md   # generación FV/eólica/almacenamiento (pendiente de revisión)
  decisiones/                    # ADR (Architecture Decision Records)
    ADR-000-principios-y-alcance.md      # principios, rol de CIM, frontera técnico/BDDAT
    ADR-001-topologia-terminal-nodo.md   # topología, terminales por tipo, reglas R1–R8
    ADR-002-herencia-tablas.md           # activo_red, FK simple, PostGIS aislado
    ADR-003-limite-modelo-generacion.md  # frontera en el inversor
    ADR-004-nomenclatura-subcampo-fv.md
    ADR-005-calculo-potencia-instalada.md
    ADR-006-criterios-modelado.md        # qué es activo, catálogos, funciones vs tipos
  esquemas/
    tension-planta-fv.svg
```

## Estado

Repasadas y consolidadas todas las **tablas de red** y las **reglas de
conectividad**. Pendiente: revisión de **generación** bajo los criterios
consolidados y las **agrupaciones lógicas** sin recinto físico.
