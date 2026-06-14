# ADR-003 — Límite del modelo en instalaciones de generación: el inversor como frontera

**Estado:** Aceptado
**Fecha:** 2026-06-12
**Revisado:** 2026-06-14

## Contexto

Una planta fotovoltaica o un parque eólico tiene dos partes muy distintas:
- Lado DC/BT interno: paneles, strings, cajas combinadoras, cableado CC (FV);
  o BT interna de la góndola (eólica). Regulado por REBT (ITC-BT).
- Lado AC/MT: a partir del inversor o transformador elevador. Regulado por
  Reglamento AT (RD 337/2014) y Códigos de Red (RD 647/2020).

## Decisión

El modelo empieza en el **punto de conexión AC** del recurso de generación:
- En FV: el inversor (UGE según RD 647/2020). El lado DC queda fuera.
- En eólica: el transformador del aerogenerador o del parque.
- En baterías: el inversor bidireccional de la batería.

Este límite coincide exactamente con la definición de **UGE (Unidad de Generación
de Electricidad)** del RD 647/2020, que excluye expresamente el lado DC de FV.

**Matiz (revisión 2026-06-14): DC fuera, AC dentro aunque sea BT.** La frontera deja
fuera el lado **DC** (REBT/ITC-BT, *caja negra* de la unidad), pero la **AC de evacuación
sí se modela como activos de red, incluida la BT** entre el inversor y el centro elevador
(los cuadros AC y los puentes del transformador entran en el RD 337/2014, y la puesta en
servicio por clusters obliga a describirla). El modelo deja de ser estrictamente «> 1 kV»:
admite la BT AC de generación (ver [ADR-000](ADR-000-principios-y-alcance.md)).

## Alternativas descartadas

| Alternativa | Problema |
|---|---|
| Modelar también el lado DC (paneles, strings) | Duplica información del proyecto de legalización BT; normativa diferente (REBT vs AT); no aporta para la topología MT |
| Modelar solo a nivel de planta (sin UGE) | Pierde la trazabilidad de cada inversor/aerogenerador, necesaria para PO 12.2 |

## Consecuencias (revisadas 2026-06-14)

- Las unidades de generación **son activos de red**: subtipos directos de `activo_red`
  (`unidad_fv`, `unidad_eolica`, `unidad_almacenamiento`), con **1 terminal** (hoja del
  grafo, fuente) y **sin multiconexión**. Se relacionan por **contención**
  (`envolvente_id`), no por FK paralela. **Se eliminan `elemento_generacion` y la capa
  común `unidad_generacion`.**
- La **planta** es una `envolvente` (tipo `planta_*`), **sin atributos técnicos propios**:
  lo administrativo (CIL, potencias acreditadas, fase, hibridación, titular, estado) es de
  BDDAT; la potencia instalada es vista de agregación; la poligonal es geometría compartida.
- La tensión del terminal (BT en FV/batería, MT en eólica/AC-block) es un atributo de la
  unidad (`tension_id`), no un cambio de estructura.
- El lado DC (módulos, orientación, potencia pico) se modela **dentro de `unidad_fv`**
  (que absorbe el antiguo `subcampo_fv`), sin topología propia.
- Detalle completo en [modelo-datos-generacion.md](../arquitectura/modelo-datos-generacion.md).
