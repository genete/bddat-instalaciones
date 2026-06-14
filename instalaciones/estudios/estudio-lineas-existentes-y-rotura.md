# Estudio — Líneas existentes y rotura/inserción en anillo

> **Documento de estudio (no consolidado).** Prepara una sesión de diseño futura: recopila
> el problema, las opciones y las preguntas abiertas. **No decide.**
> Relacionado: [ADR-001](../decisiones/ADR-001-topologia-terminal-nodo.md) (topología),
> [ADR-007](../decisiones/ADR-007-agrupacion-logica-lineas.md) (agrupación lógica),
> [modelo-datos.md](../arquitectura/modelo-datos.md), [ADR-000](../decisiones/ADR-000-principios-y-alcance.md) (frontera con BDDAT).

## Por qué este estudio

En las resoluciones de BDDAT se habla constantemente de **líneas existentes** (de
distribuidoras, a menudo **no modeladas** en nuestro sistema) y de **conexiones a ellas**:
abrir un anillo, insertar un par de empalmes (o un elemento de corte / embarrado) y crear
sendas líneas nuevas a cada lado. Modelar esto toca la **identidad** de los activos, el
**reparto de geometría** y la **frontera** con BDDAT. Merece su sesión.

## Problema 1 — Rotura/inserción: split de un tramo

Insertar un activo (empalme, `elemento_corte`, embarrado) en medio de un tramo existente
convierte **1 `linea`** (nodo A — nodo B) en **2 tramos** (A — N, N — B) + el activo
insertado en el nodo nuevo **N**, y **parte la geometría** (LineString) por el punto.

A resolver:

- **Identidad (P6):** ¿la `linea` original **muere** y nacen dos tramos nuevos, o
  **conserva su `id`** en un lado y nace solo el otro? Tiene consecuencias para BDDAT: a qué
  activo apuntan los expedientes/resoluciones previas.
- **Reparto de geometría:** partir el LineString en el punto (`ST_LineSubstring` / `ST_Split`),
  repartir `longitud_m` (no derivable de la traza 2D) y decidir si las geometrías nuevas se
  crean o se comparten parcialmente.
- **Herencia de atributos:** los dos tramos heredan conductor, tensión, aislamiento… del
  original; ¿se duplican o se revisan?
- **Agrupación lógica (ADR-007):** los tramos resultantes deberían seguir colgando de la
  misma `agrupacion_logica` que el original.

## Problema 2 — La línea existente no modelada (de un tercero)

Para "abrir el anillo de la LA JANDA existente" hay que **representar esa línea** lo mínimo
para colgar la conexión, **sin** haber inventariado sus 30 km. Opciones a estudiar:

- **Tramo *placeholder*:** una `linea` con nombre y datos parciales, sin geometría completa,
  marcada como "existente / no inventariada".
- **Nodo frontera:** un `nodo` "frontera con red existente" al que se conecta lo nuevo, sin
  modelar la línea ajena.
- **Agrupación lógica vacía:** una `agrupacion_logica` "LA JANDA (existente)" como ancla de
  identidad, con o sin tramos.

Criterio: permitir describir la modificación en la resolución sin obligar a modelar red
ajena y **sin crear falsas fuentes de verdad** (coherente con ADR-000).

## Problema 3 — Frontera: el "verbo" (BDDAT) vs el "estado" (instalaciones)

La **operación** ("se abre el anillo y se crean dos empalmes") es lo que **describe la
resolución** → es un **verbo de BDDAT**. El **estado topológico resultante** (dos tramos +
el activo insertado) es de **instalaciones**. A decidir:

- ¿Instalaciones guarda solo el **estado final** (los activos resultantes) y BDDAT narra la
  transición?
- ¿O instalaciones registra la **operación de edición** (split/merge) como evento, para
  trazabilidad?

## Casos de uso típicos a contemplar

- Apertura de anillo MT con inserción de un **CT en entrada-salida** (2 celdas/empalmes + CT).
- Inserción de un **elemento de corte** (seccionamiento) en una línea existente.
- **Doble alimentación:** conexión a dos líneas existentes distintas.
- Conversión de tramo aéreo↔subterráneo (ver [pendientes-modelado.md](pendientes-modelado.md)).

## Preguntas abiertas para la sesión

1. Regla de **identidad en el split** (¿muere el original o sobrevive un lado?).
2. Modelo de la **línea existente no inventariada** (placeholder / nodo frontera / agrupación).
3. ¿Se registra la **operación** (split/merge) o solo el **estado**?
4. Mecánica de **reparto de geometría y `longitud_m`**.
5. Encaje con **BDDAT**: a qué `activo_red.id` apuntan las resoluciones tras el split.
