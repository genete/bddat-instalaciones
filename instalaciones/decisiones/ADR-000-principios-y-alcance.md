# ADR-000 — Principios fundacionales y alcance del modelo

**Estado:** Aceptado
**Fecha:** 2026-06-13

Este ADR es la base de la que cuelgan todos los demás. Fija el propósito del
modelo, sus principios y, sobre todo, su **frontera de alcance**: qué pertenece
a este modelo técnico y qué pertenece a BDDAT.

---

## Propósito

Definir la **estructura de datos para la descripción técnica** de los activos
de las instalaciones eléctricas de alta tensión (MT y AT, > 1 kV), incluyendo
distribución, transporte y generación.

El sistema debe servir a cuatro necesidades:

1. **Inventario de activos** montados manualmente, tanto los de un expediente
   como los que viven fuera de él (registro por titular para inspecciones,
   responsabilidades, mantenimiento). Esto último será otra aplicación que
   bebe de estos datos.
2. **Fuente de descriptores humanos**: vistas y consultas volcables a PDF
   (p. ej. el "resuelve" de una resolución de BDDAT).
3. **Geometría gestionable opcionalmente** y cálculos GIS calculables o no.
4. **Nivel de detalle administrativo**: descripción razonable para lo que se
   maneja en distribución, transporte y generación, acorde a lo que exige una
   autorización de modificación y a lo que necesita el registro de instalaciones
   de generación para estar completo. No se busca exhaustividad de ingeniería.

---

## Principios

**P1 — El activo es el núcleo, independiente del expediente y del titular.**
Un activo existe por sí mismo. Expediente, titular y tramitación lo *referencian*,
no al revés. Esto permite que otras aplicaciones consuman los mismos activos.

**P2 — El modelo es fuente de descriptores humanos.**
Las vistas deben poder volcarse a texto legible para resoluciones PDF. Implica
campos descriptivos y un orden jerárquico presentable.

**P3 — La geometría es opcional y aislada.**
Un activo puede existir sin geometría. Las operaciones GIS se calculan solo
cuando hay geometría. (Desarrollado en ADR-002.)

**P4 — Nivel de detalle administrativo, no de ingeniería.**
El criterio de "qué campos" lo marca la autorización y el registro, no un proyecto
de ingeniería. Un único esquema cubre distribución, transporte y generación.

**P5 — El activo tiene dos ejes de estado, ortogonales y acoplados por eventos.**
Ver sección "Frontera de alcance". El eje físico es del activo; el administrativo
es de BDDAT.

**P6 — Identidad persistente del activo.**
Un activo conserva su `id` a través de los expedientes y estados por los que pasa
en su vida (alta, modificación, baja). El `id` es el gancho de referencia para BDDAT
y para las aplicaciones que consumen estos datos.

**P7 — Portabilidad de motor como principio de proyecto.**
SQL estándar, sin características exclusivas de un motor (`INHERITS`, JSONB como
estructura, etc.). Lo inevitablemente exclusivo (PostGIS) queda aislado y con vía
de salida documentada. (Desarrollado en ADR-002.)

---

## Rol de IEC CIM 61970

IEC CIM 61970 es un **modelo de información abstracto en UML** orientado a la
**interoperabilidad e intercambio** entre sistemas de operación (EMS, SCADA,
CGMES entre TSOs). **No es un diseño de base de datos** y su propósito (operar y
analizar la red) difiere del nuestro (inventario + tramitación + GIS).

Por tanto, CIM es **referencia conceptual y de vocabulario, no un corsé de esquema**:

| Sí tomamos de CIM                                   | No hacemos                                  |
|-----------------------------------------------------|---------------------------------------------|
| El patrón **Terminal–ConnectivityNode** (ADR-001)   | Perseguir "CIM-compliance" del esquema      |
| La **taxonomía/vocabulario** de equipos             | Replicar su jerarquía de herencia completa  |
| Separar **conectividad** de **geografía**           | Exportar CGMES (no somos TSO)               |
| Referencia de atributos típicos por equipo          | Forzar las tablas a sus paquetes UML        |

Consecuencia: las decisiones de esquema se toman con el criterio de simplicidad y
mantenibilidad para BDDAT, sin temor a "desviarse del estándar".

---

## Frontera de alcance: qué es de este modelo y qué es de BDDAT

El activo tiene **dos ejes de estado** (P5):

| Eje              | Estados (ejemplos)                                                       | Dónde vive                       | Cardinalidad |
|------------------|--------------------------------------------------------------------------|----------------------------------|--------------|
| **Físico**       | no existe/proyectado · construido · en servicio · fuera de servicio · desmantelado | Atributo del **activo** (este modelo) | Único en cada instante |
| **Administrativo** | solicitado · en tramitación · autorizado (AAP) · autorizado construir (AAC) · autorizado explotar (APS) · inscrito · desestimado · cerrado · desmantelado | **BDDAT** (relación activo × expediente) | Múltiple en el tiempo |

**Reglas de la frontera:**

1. **Este modelo define solo la parte técnica**: identidad, características
   eléctricas y geometría del activo, más su **estado físico**.
2. **El estado físico se infiere de la tramitación**, no se observa ni se edita
   libremente. En un sistema administrativo, "en servicio" significa "existe acta
   de puesta en servicio firme", no una medida de campo. Si la realidad de campo
   no coincide con lo tramitado, eso es otro asunto (infracción, expediente
   sancionador) ajeno a este modelo.
3. **El eje administrativo, el expediente, las resoluciones y la máquina de
   estados de tramitación pertenecen a BDDAT**, no a este modelo. BDDAT los
   modela en tablas propias que referencian al activo por su `id`.
4. **El gancho de referencia son los `id` de los activos.** BDDAT añade los
   atributos administrativos desde fuera, vinculándose por esos `id`, y puede
   dividir/agrupar activos por sus `id`.
5. Este modelo es **agnóstico** respecto a la máquina de estados administrativa.

---

## Consecuencias

- La frontera **no influye** en el modelo de tablas técnicas, sus relaciones ni
  los atributos de cada activo. Es una delimitación conceptual.
- El estado físico es un atributo del activo (su forma exacta se concreta en la
  revisión de atributos); su valor lo alimenta la tramitación de BDDAT.
- Ninguna tabla de expediente, resolución o estado administrativo se define en
  este repositorio.
- Los demás ADR (001–005) se interpretan bajo estos principios.
