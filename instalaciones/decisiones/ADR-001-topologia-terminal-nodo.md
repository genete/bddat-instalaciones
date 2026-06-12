# ADR-001 — Topología mediante Terminal–Nodo (patrón CIM)

**Estado:** Propuesto  
**Fecha:** 2026-06-12

## Contexto

Para representar la conectividad eléctrica de la red necesitamos un modelo
que permita saber qué elementos están eléctricamente unidos entre sí, sin
conocer de antemano el tipo de cada elemento.

## Decisión

Se adopta el patrón **Terminal–ConnectivityNode** del estándar IEC CIM 61970:

- Cada elemento tiene uno o más **terminales** (puntos de conexión).
- Los terminales se conectan a **nodos** (puntos eléctricos sin masa física).
- Dos elementos están conectados si comparten el mismo nodo.

## Alternativas descartadas

| Alternativa | Problema |
|---|---|
| FK directa entre elementos (linea.extremo_a_id → elemento_corte) | Acopla tipos; no escala a nuevos tipos de elementos |
| Tabla de adyacencia `conexion(elemento_a, elemento_b)` | Pierde el concepto de "punto de unión"; no soporta N elementos en un nodo |

## Consecuencias

- La topología es independiente del tipo de elemento: el grafo se recorre
  siempre por la tabla `terminal`.
- Permite N elementos conectados en un mismo nodo (barras, empalmes múltiples).
- Compatible con exportación RDF/XML según IEC 61970-501 si fuera necesario.
