# ADR-006 — Criterios de modelado de activos

**Estado:** Aceptado
**Fecha:** 2026-06-13

Recoge los principios de diseño que emergieron al revisar tabla por tabla el
modelo de red. Complementan a [ADR-000](ADR-000-principios-y-alcance.md) y guían
qué se modela como activo, qué como atributo y qué como texto.

---

## C1 — Solo es activo lo que conduce o tiene datos propios relevantes

Un activo (fila en `activo_red` + especialización) se justifica si **conduce**
(participa en la topología) o si es **inventariable con datos administrativos
propios**. Lo accesorio se describe en texto, no se inventaría:

| Se modela como activo | Se describe en texto (no es activo) |
|---|---|
| línea, corte, trafo, embarrado, empalme, apoyo | arquetas, canalizaciones (→ `linea.obra_civil_descripcion`) |
| pararrayos, red_tierra, TI/TT de AIS | aisladores, distancia de fuga (→ `linea.nivel_aislamiento` resume la capacidad) |
| | relé de protección (→ implícito en `aparato=interruptor_automatico` / `notas`) |
| | cámaras de empalme (→ lo relevante es el empalme) |

Corolario: la **canalización** subterránea no es activo; coincide con la traza
(geometría LineString) de la propia línea.

---

## C2 — "Protección" y "medida" son funciones, no tipos de activo

No existe una tabla genérica `elemento_proteccion`. La protección se reparte según
el fenómeno:

- **sobreintensidad** → `elemento_corte` (función=protección + fusible/relé)
- **sobretensión** → `pararrayos`
- **defecto a tierra** → `red_tierra`

Igual la medida: integrada en la celda (`elemento_corte`, función=medida) en MT, o
como `elemento_medida` (TI/TT) en AIS/GIS.

---

## C3 — Un campo opcional que no se rellena no estorba

Datos ocasionales (altura/esfuerzo de apoyo, intensidad máxima de barra…) se
modelan como **nullable** o van a `notas`, en vez de forzarse. Si no se leen en la
resolución, no pasa nada. No se crea columna para un dato que la mayoría de las
veces sería desconocido o inventado.

---

## C4 — Catálogos para magnitudes normalizadas

Las magnitudes normalizadas son **FK a catálogo**, no valores libres:

- `tension` — tensiones de servicio (3, 6, 10, 15, 20, 30, 45, 66, 132, 220, 400 kV).
- `nivel_aislamiento` — niveles Um (RD 223/2008): 3.6, 7.2, 12, 17.5, 24, 36, 52,
  72.5, 123, 145, 245, 420 kV.

Lo que **no** está normalizado o es una "lista frankenstein" en la práctica
(conductores, material de apoyos) se deja en **texto plano** — su catalogación es
una sesión aparte.

---

## C5 — La unidad física independiente es una fila

Cuando varios elementos coexisten físicamente pero tienen **ciclo de vida
independiente** (alta/baja por separado), son **filas separadas**, no un campo
multivalor:

- **Doble circuito = dos `linea`** (una puede darse de baja sin la otra).
- **Transformador de 3 devanados = dos `transformador` + embarrado** (caso AT raro).
- **Doble barra = dos `embarrado`** + celda de acople.

Esto evita campos posicionales nullable (`tension_terciario`, `num_circuitos`,
`num_barras`) y respeta la independencia administrativa.

---

## C6 — Servicio vs aislamiento: dos tensiones distintas

- **Tensión de servicio**: la de operación. La llevan los activos que la definen
  (línea, embarrado, trafo por lado); el resto la hereda del nodo.
- **Nivel de aislamiento (Um)**: la capacidad dieléctrica, **independiente** del
  servicio. Una línea puede estar aislada a 24 kV y operarse a 20; cambiar la
  tensión de servicio no toca el aislamiento.

`nivel_aislamiento` es **opcional salvo donde no hay servicio del que derivarlo**:
obligatorio en `elemento_corte`, declarado en `linea`, ausente en embarrado y trafo.

---

## C7 — Multiconexión: embarrado y empalme

Un nodo solo reúne **más de 2 terminales** si hay un activo que lo habilita:
**embarrado** (general) o **empalme** (exclusivo de líneas). El resto de activos
son de paso (2) o extremo (1). Es propiedad del activo, no del nodo (ver ADR-001 R5/R7).

---

## C8 — Contención general, no solo envolventes

La contención (`envolvente_id`, self-FK de `activo_red`) la puede ejercer
**cualquier activo con ubicación**, no solo los de tipo `envolvente`. Un apoyo
contiene la aparamenta que monta (seccionadores, pararrayos, empalmes); una
envolvente (CT, subestación, posición) contiene su contenido; las envolventes se
**anidan** (subestación → posición → aparamenta). El descriptor humano (P2) recorre
esta jerarquía.
