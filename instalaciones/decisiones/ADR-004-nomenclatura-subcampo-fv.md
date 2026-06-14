# ADR-004 — Nomenclatura: `subcampo_fv` para la agrupación operativa de paneles

**Estado:** Aceptado · *revisado 2026-06-14*
**Fecha:** 2026-06-13

> **Nota (2026-06-14):** la **entidad** `subcampo_fv` se **absorbe en `unidad_fv`** (1
> inversor = 1 isla; varias orientaciones/MPPT → texto agregado). El término «subcampo» se
> conserva como concepto descriptivo, no como tabla. La elección de nomenclatura de este
> ADR sigue siendo válida para ese concepto. Ver
> [modelo-datos-generacion.md](../arquitectura/modelo-datos-generacion.md).

## Contexto

Necesitamos un nombre para la entidad que representa una "isla de paneles"
operativamente instalables: el conjunto de módulos FV que conectan a un inversor
concreto y constituyen la unidad mínima con potencia instalada propia a efectos
del RD 997/2025.

## Alternativas evaluadas

| Nombre            | Pros                                          | Contras                                      |
|-------------------|-----------------------------------------------|----------------------------------------------|
| `subcampo_fv`     | Término estándar en el sector FV español      | Específico de FV                             |
| `bloque_generacion` | Genérico, aplica a eólica/baterías          | "Bloque" no es término normativo             |
| `isla_fv`         | Intuitivo                                     | Coloquial, no aparece en normativa           |
| `zona_fv`         | Coincide con nomenclatura operativa de planta | Ambiguo, "zona" se usa para otras cosas      |
| `agregado_pe`     | Alineado con CIM PowerElectronicsConnection   | Demasiado técnico, ajeno al sector           |

## Decisión

Se usa **`subcampo_fv`** por ser el término que emplean proyectistas e ingenieros
en España en proyectos de legalización y en O&M de plantas fotovoltaicas.

Si en el futuro se necesita modelar agrupaciones equivalentes en eólica o baterías,
se crean tablas análogas (`subcampo_eolico`, `modulo_bateria_bloque`) con la misma
estructura lógica, en lugar de generalizar prematuramente.

## Consecuencias

- La tabla se llama `subcampo_fv` y es específica de tecnología fotovoltaica.
- La generalización a otras tecnologías queda pendiente de un estudio posterior.
