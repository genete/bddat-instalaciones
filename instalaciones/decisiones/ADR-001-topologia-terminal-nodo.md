# ADR-001 — Topología mediante Terminal–Nodo (patrón CIM)

**Estado:** Aceptado
**Fecha:** 2026-06-12  
**Revisado:** 2026-06-13

## Contexto

Para representar la conectividad eléctrica de la red necesitamos un modelo
que permita saber qué elementos están eléctricamente unidos entre sí, sin
conocer de antemano el tipo de cada elemento.

## Decisión

Se adopta el patrón **Terminal–ConnectivityNode** del estándar IEC CIM 61970:

- Cada activo de red tiene uno o más **terminales** (puntos de conexión).
- Los terminales se conectan a **nodos** (puntos eléctricos sin masa física).
- Dos elementos están conectados si comparten el mismo nodo.

## Alternativas descartadas

| Alternativa | Problema |
|---|---|
| FK directa entre elementos (`linea.extremo_a_id → elemento_corte`) | Acopla tipos; no escala a nuevos tipos de elementos |
| Tabla de adyacencia `conexion(elemento_a, elemento_b)` | Pierde el concepto de "punto de unión"; no soporta N elementos en un nodo |

## Consecuencias

- La topología es independiente del tipo de elemento: el grafo se recorre
  siempre por la tabla `terminal`.
- Permite N elementos conectados en un mismo nodo (barras, bifurcaciones en Y).
- Compatible con exportación RDF/XML según IEC 61970-501 si fuera necesario.

---

## Reglas de integridad: terminales por tipo de activo

Estas reglas **no se implementan en la base de datos** (no son constraints ni
triggers). Viven en el backend (Python) como validación antes de insertar o
modificar terminales. Se documentan aquí como especificación de referencia.

| tipo_elemento        | min | max | Notas                                                       |
|----------------------|-----|-----|-------------------------------------------------------------|
| linea                | 2   | 2   | Terminal 1 = inicio, terminal 2 = fin del tramo             |
| corte                | 2   | 2   | Terminal 1 y 2 = los dos lados del elemento de corte        |
| transformador        | 2   | 2   | Terminal 1 = AT, terminal 2 = BT. Caso 3 devanados excepcional solo en subestaciones REE |
| embarrado            | 1   | 1   | Siempre un único terminal en el nodo de barra               |
| empalme              | 2   | 2   | Siempre une exactamente dos tramos                          |
| apoyo alineación     | 0   | 0   | Sustentación mecánica; sin relevancia eléctrica             |
| apoyo derivación     | 0   | 0   | La bifurcación la expresa el nodo compartido, no el apoyo   |
| apoyo fin de línea   | 1   | 1   | Marca el extremo muerto de una línea                        |
| arqueta/canalización | 0   | 0   | Infraestructura civil; sin conectividad eléctrica propia    |
| elemento_generacion  | 1   | 1   | Punto de conexión MT (el transformador elevador es elemento separado) |
| medida / protección  | 0   | 0   | Contenidos en envolvente; sin terminales propios            |

## Sobre los tipos de nodo

Los nodos **no tienen tipo**. El comportamiento "bus" (N elementos conectados)
o "punto" (2 elementos) es una propiedad emergente que se lee contando terminales:

```sql
-- Nodos con 3 o más elementos conectados (comportamiento bus):
SELECT nodo_id, COUNT(*) AS num_elementos
FROM terminal
GROUP BY nodo_id
HAVING COUNT(*) >= 3;
```

Añadir un campo `tipo_nodo` sería redundante y susceptible de desincronizarse.
La única validación útil es que la `tension_nominal_kv` del nodo sea coherente
con la de los elementos que conectan a él — validación también del backend.
