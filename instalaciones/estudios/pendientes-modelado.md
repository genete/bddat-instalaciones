# Pendientes de modelado (deudas menores)

> **Documento de backlog (no consolidado).** Recoge temas detectados que no son urgentes y
> se abordarán en sesiones futuras. Relacionado:
> [modelo-datos.md](../arquitectura/modelo-datos.md),
> [ADR-006](../decisiones/ADR-006-criterios-modelado.md),
> [ADR-001](../decisiones/ADR-001-topologia-terminal-nodo.md).

## 1. Tipología de empalmes (con la sesión de conductores)

El `empalme.tipo` actual (recto / derivacion / terminal_botella / transicion_aero_subt) se
queda corto. Según las líneas a unir hay tipologías y características propias:

- **Aéreo–subterráneo:** conversión aéreo-subterránea (botellas terminales, autoválvulas).
- **Subterráneo–subterráneo:** empalme seco u otros tipos; REE detalla características.
- **Aéreo–aéreo:** puentes flojos, nudos, herrajes.

Acción: ampliar el catálogo `empalme.tipo` y valorar un campo de **características** (C3/C4).
Va naturalmente **con la sesión de cables/conductores** (ya pendiente).

## 2. Activos shunt de 1 terminal (compensación de reactiva)

Las subestaciones llevan **baterías de condensadores** (y a veces **reactancias shunt**) para
compensar reactiva. **Sí conducen** (aportan/absorben reactiva), conectadas a barra →
**1 terminal, hoja shunt** (CIM `ShuntCompensator`). Encajan en el patrón "hoja de 1 terminal"
que ya introdujo la generación ([ADR-001](../decisiones/ADR-001-topologia-terminal-nodo.md)).

Acción: definir un activo `compensador` (tipo: condensador / reactancia), 1 terminal.
**Ojo a la colisión de nombre** con la batería de *almacenamiento* (generación): son cosas
distintas.

## 3. Equipos de neutro / puesta a tierra con transformador

No toda la puesta a tierra son picas. Hay **equipos de neutro** que **conducen**:
**transformador de puesta a tierra (zig-zag)**, **reactancia/resistencia de neutro**
(Petersen). Conectan a barra y crean/limitan el neutro → **1 terminal a barra** (el otro lado
es tierra). Distintos de `red_tierra` (malla de picas, 0 terminales, que se queda igual).

Acción: añadir el **equipo de neutro** como activo conectado (1 terminal), separado de
`red_tierra`. Posiblemente un tipo de transformador especial o tabla propia.

## 4. Auxiliares no conductores

Sistemas de **telemando, control, SICOP, antiintrusismo**: por C1 **no son activos eléctricos**
(no conducen). El telecontrol ya se refleja en `elemento_corte.accionamiento = telecontrolado`.

Acción (recomendación): describir en **`notas`** de la envolvente, salvo que BDDAT exija
inventariarlos como activos auxiliares de 0 terminales.

## Patrón común (2 y 3)

Compensadores y equipos de neutro **no abren un patrón nuevo**: son aplicaciones del **"hoja
de 1 terminal"** (fuente/shunt) ya consolidado con la generación. Modelado claro; se abordan
tras los temas de enjundia.
