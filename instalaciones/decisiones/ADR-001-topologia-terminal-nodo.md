# ADR-001 — Topología mediante Terminal–Nodo (patrón CIM)

**Estado:** Aceptado
**Fecha:** 2026-06-12
**Revisado:** 2026-06-13

## Contexto

Para representar la conectividad eléctrica de la red necesitamos un modelo
que permita saber qué activos están eléctricamente unidos entre sí, sin
conocer de antemano el tipo de cada uno.

## Decisión

Se adopta el patrón **Terminal–ConnectivityNode** del estándar IEC CIM 61970:

- Cada activo de red tiene cero o más **terminales** (puntos de conexión).
- Los terminales se conectan a **nodos** (puntos eléctricos virtuales, sin masa física).
- Dos activos están conectados si comparten el mismo nodo.

## Alternativas descartadas

| Alternativa | Problema |
|---|---|
| FK directa entre activos (`linea.extremo_a_id → elemento_corte`) | Acopla tipos; no escala |
| Tabla de adyacencia `conexion(activo_a, activo_b)` | Pierde el "punto de unión"; no soporta N activos en un nodo |

## Consecuencias

- La topología es independiente del tipo: el grafo se recorre siempre por `terminal`.
- Permite N activos en un mismo nodo (barras, derivaciones).
- Compatible con exportación RDF/XML (IEC 61970-501) si alguna vez se necesita.

---

## Terminales y multiconexión por tipo de activo

`terminales` = nº de puntos de conexión propios del activo.
`multiconexión` = si el activo **habilita** que en su nodo concurran más de 2 terminales.

| Activo | Terminales | Multiconexión |
|---|---|---|
| linea | 2 | no |
| elemento_corte | 2 | no |
| transformador | 2 | no |
| embarrado | 1 | **sí (N)** |
| empalme | 1 | **sí (N)** |
| apoyo | 0 | — |
| elemento_medida | 0 | — |
| pararrayos | 0 | — |
| red_tierra | 0 | — |
| envolvente | 0 | — |

Notas:
- `apoyo` = 0 siempre (es estructura; el terminal de extremo es de la línea, no del apoyo).
- `empalme` = 1 y multiconexión: es un "embarrado exclusivo de líneas".
- 3 devanados se modelan como 2 transformadores + embarrado (caso excepcional AT).

Estas reglas **no se implementan en la base de datos** (no son constraints ni
triggers): viven en el **backend** como validación. Se documentan aquí como
especificación de referencia.

---

## Reglas de conectividad (invariantes de backend)

- **R1 — Coherencia de tensión de servicio.** Todos los terminales de un nodo
  comparten tensión de servicio. La fija el **primer** activo con tensión propia
  que conecta; cualquier terminal posterior debe coincidir o se rechaza.

- **R2 — Sin excepciones (incluido el transformador).** Al conectar un terminal a
  un nodo, se comprueba/elige la tensión de ese terminal; debe coincidir con la del
  nodo (o definirla si está vacío). El trafo no es excepción: cada terminal lleva la
  tensión de su lado y se conecta a un nodo de esa tensión; si el activo tiene varias
  tensiones, se elige cuál se conecta. No coincide → error.

- **R3 — Origen y herencia de la tensión.**
  - Activo **con** tensión propia (línea, embarrado, trafo por lado) → **define** la
    del nodo si está vacío, o debe **coincidir**.
  - Activo **sin** tensión propia (elemento_corte, empalme) → **toma** la del nodo,
    sin comprobación.
  - Los no-transformadores (línea, corte, empalme) son **equipotenciales**: propagan
    la misma tensión a todos sus terminales. El transformador es el único que lleva
    tensión distinta por lado.

- **R4 — Activos de 0 terminales.** No entran en el grafo: se relacionan por
  **contención** (`envolvente_id`), nunca por terminal–nodo. Equivalencia:
  `terminales = 0` ⟺ no conductor ⟺ fuera del grafo ⟺ vinculado por contención.

- **R5 — Multiconexión.** El nodo es virtual y no "sabe" de buses. Que un nodo reúna
  **más de 2 terminales** lo habilita la presencia de un activo multiconexión:
  **embarrado** (general) o **empalme** (exclusivo de líneas). Sin habilitador, el
  nodo es punto a punto (≤ 2 terminales).

- **R6 — Cardinalidad de terminales propios.** Cada activo tiene su número fijo
  (línea/corte/trafo = 2; embarrado/empalme = 1; no conductores = 0).

- **R7 — Multiconexión como propiedad del activo.** Es una columna del activo
  (no/sí-N) que implementa R5: línea/corte/trafo = no; embarrado/empalme = sí.

- **R8 — Aislamiento suficiente.** Cada activo con `nivel_aislamiento` declarado
  (elemento_corte, línea) debe cumplir `nivel_aislamiento ≥ tensión de servicio del
  nodo`. Una celda de 24 kV no puede ir en un nodo de 66 kV. Sobreaislar está
  permitido (36 kV en nodo de 20 kV).

---

## Sobre los tipos de nodo

Los nodos **no tienen tipo**. El comportamiento "bus" (N terminales) es emergente
y lo habilita un embarrado/empalme conectado (R5), no un atributo del nodo:

```sql
-- Nodos con comportamiento bus (3+ terminales):
SELECT nodo_id, COUNT(*) AS n FROM terminal GROUP BY nodo_id HAVING COUNT(*) >= 3;
```

Añadir un `tipo_nodo` sería redundante y propenso a desincronización.
