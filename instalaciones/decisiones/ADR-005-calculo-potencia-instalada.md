# ADR-005 — Cálculo de potencia instalada: GENERATED en subcampo, VIEW en planta, backend para estados

**Estado:** Aceptado
**Fecha:** 2026-06-13

## Contexto

`potencia_instalada_kw` aparece en dos niveles jerárquicos con necesidades distintas:

1. **Subcampo FV**: fórmula cerrada sobre columnas de la misma fila
   (módulos, inversor, factor bifacial según RD 997/2025 Art. 5.2)
2. **Planta completa**: agregación de todos los subcampos
3. **Operacional / por estados**: la potencia "en servicio" real depende del
   estado de cada elemento (proyectado / autorizado / en_servicio / fuera_servicio)
   y varía dinámicamente

## Decisión

| Nivel              | Estrategia                  | Justificación                                           |
|--------------------|-----------------------------|---------------------------------------------------------|
| Subcampo FV        | Columna `GENERATED STORED`  | Fórmula intra-fila, siempre consistente, cero mantenimiento |
| Planta (legal)     | Vista `v_potencia_planta`   | Agregación entre tablas; vista siempre fresca, sin riesgo de desincronización |
| Planta (operacional)| Cálculo en backend (Python) | Depende de filtros de estado dinámicos; no es un dato fijo de la BD |

## Naturaleza del dato

`potencia_instalada_kw` es un dato **administrativo-legal**, no operacional:

- Determina la **competencia para tramitar** la autorización. En Andalucía,
  instalaciones > 50 MW son competencia del Ministerio (MITECO), no de la
  Junta de Andalucía (Consejería de Industria).
- Aparece en el proyecto de legalización y en el registro RETA/CNMC.
- **No refleja** la potencia en servicio en un instante dado.

La potencia operacional (qué está enchufado y funcionando ahora) se calcula
en el backend filtrando subcampos por el estado de sus elementos. Este cálculo
queda fuera del alcance del modelo de datos actual (los estados se estudiarán
en una iteración posterior).

## Alternativas descartadas

| Alternativa | Problema |
|---|---|
| Columna almacenada con trigger en `elemento_generacion` | Los triggers se desincronizán en operaciones bulk; complejidad innecesaria |
| Vista materializada con REFRESH | Overkill para el volumen esperado; añade operación de mantenimiento |
| Cálculo solo en Python para todos los niveles | La BD no tiene verdad completa; cualquier consulta directa da dato incompleto |

## Consecuencias

- Ninguna lógica de negocio sobre potencia instalada vive en triggers.
- La vista `v_potencia_planta` es de solo lectura; no se escribe en ella.
- El backend (Python/SQLAlchemy) es responsable de los cálculos que involucran estados.
