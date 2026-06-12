# ADR-003 — Límite del modelo en instalaciones de generación: el inversor como frontera

**Estado:** Propuesto  
**Fecha:** 2026-06-12

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

## Alternativas descartadas

| Alternativa | Problema |
|---|---|
| Modelar también el lado DC (paneles, strings) | Duplica información del proyecto de legalización BT; normativa diferente (REBT vs AT); no aporta para la topología MT |
| Modelar solo a nivel de planta (sin UGE) | Pierde la trazabilidad de cada inversor/aerogenerador, necesaria para PO 12.2 |

## Consecuencias

- Las `unidad_generacion` (inversores, aerogeneradores) no tienen terminales MT
  propios: están contenidas en la `envolvente` de la planta.
- El `elemento_generacion` sí conecta a la red MT mediante terminales.
- Los datos del lado DC (paneles, inclinación, potencia pico) son atributos
  de `elemento_generacion`, no entidades separadas con topología.
