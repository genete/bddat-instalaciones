# Modelo de datos — Elementos de generación y almacenamiento

## Cobertura en IEC CIM 61970

La edición **IEC 61970-301:2020 (Ed. 7)** cubre generación renovable y almacenamiento
de forma explícita. Versiones anteriores (2013/2016) eran deficientes en este área,
por lo que es importante usar la edición 2020 como referencia.

### Jerarquía de clases CIM para generación

```
GeneratingUnit (base)
  ├── SolarGeneratingUnit         → planta fotovoltaica (nivel agregado)
  ├── WindGeneratingUnit          → parque eólico (nivel agregado)
  ├── HydroGeneratingUnit
  └── ThermalGeneratingUnit

PowerElectronicsUnit (base — para recursos conectados vía electrónica de potencia)
  ├── BatteryUnit                 → batería / almacenamiento
  ├── PhotovoltaicUnit            → módulo FV individual / inversor
  └── PowerElectronicsWindUnit    → aerogenerador con convertidor AC/DC/AC

PowerElectronicsConnection      → punto de conexión del recurso PE a la red AC
```

`GeneratingUnit` representa el recurso de generación a nivel de planta o parque.
`PowerElectronicsUnit` representa cada unidad individual conectada por electrónica
de potencia (inversor, aerogenerador tipo 4, batería).

---

## Marco normativo español

### Terminología RD 647/2020 (Códigos de Red de conexión)

El RD 647/2020 transpone los Reglamentos EU 2016/631 y 2016/1388 al derecho español.
Define la jerarquía de una instalación de generación:

| Término   | Definición                                                                 | Equivalente CIM         |
|-----------|----------------------------------------------------------------------------|-------------------------|
| **MGE**   | Módulo de Generación de Electricidad — la instalación completa             | `GeneratingUnit`        |
| **UGE**   | Unidad de Generación de Electricidad — cada generador individual           | `PowerElectronicsUnit`  |
| **CAMGE** | Componentes Adicionales del MGE — elementos activos que no son UGE         | Equipos auxiliares      |

Precisiones importantes del RD 647/2020:
- En FV: **solo el inversor** es la UGE (el lado DC — paneles, strings — no forma
  parte de la planta de generación principal a efectos de conexión a red)
- En eólica: **el aerogenerador completo** (torre + palas + góndola) es la UGE
- Almacenamiento híbrido: puede compartir el mismo punto de conexión con la
  generación si se cumplen requisitos técnicos

### Normativa de referencia

| Norma                  | Contenido                                                          |
|------------------------|--------------------------------------------------------------------|
| **RD 647/2020**        | Implementación códigos de red UE para generación y almacenamiento  |
| **RD 1183/2020**       | Acceso y conexión a redes de transporte y distribución             |
| **PO 12.2 (REE)**      | Requisitos mínimos instalaciones de generación conectadas a REE    |
| **ITC-BT (RD 842/2002)** | REBT — regula el lado BT/DC interno de las instalaciones FV     |
| **RD 244/2019**        | Condiciones técnicas y económicas del autoconsumo                  |
| **IEC 61970-302:2024** | Modelos dinámicos: aerogeneradores tipo 1–4, inversores, baterías  |

Fuentes:
- [RD 647/2020 — BOE](https://boe.es/eli/es/rd/2020/07/07/647)
- [PO 12.2 REE instalaciones generación — ESIOS (PDF)](https://api.esios.ree.es/documents/449/download?locale=es)
- [PowerElectronicsUnit — CIM Datamodel (Zepben)](https://zepben.github.io/evolve/docs/cim/ewb/IEC61970/Base/Generation/Production/PowerElectronicsUnit/)
- [IEC 61970-302:2024 Dynamics — IEC Webstore](https://webstore.iec.ch/en/publication/68152)
- [Extensión IEC 61970 para almacenamiento — ResearchGate](https://www.researchgate.net/publication/292358936_Extension_of_IEC_61970_for_electrical_energy_storage_modelling)

---

## Tablas propuestas

### `elemento_generacion` — planta o parque (nivel MGE)

Hereda de `elemento` (misma tabla base que el resto de la red).

| Columna               | Tipo       | Descripción                                                  |
|-----------------------|------------|--------------------------------------------------------------|
| elemento_id           | UUID PK/FK |                                                              |
| subtipo               | TEXT       | fotovoltaica / eolica / hidraulica / almacenamiento / hibrida / cogeneracion |
| potencia_instalada_kw | NUMERIC    | Potencia pico instalada total en kW                          |
| potencia_nominal_kw   | NUMERIC    | Potencia nominal de conexión a red en kW                     |
| tension_conexion_kv   | NUMERIC    | Tensión en el punto de conexión a la red MT/AT               |
| num_unidades          | INTEGER    | Número de UGE (inversores, aerogeneradores, módulos batería)  |
| factor_capacidad      | NUMERIC    | Factor de capacidad típico (0–1), orientativo                |

---

### `unidad_generacion` — UGE individual (inversor, aerogenerador, módulo batería)

Equivale a `PowerElectronicsUnit` en CIM. No hereda de `elemento` directamente
porque son activos eléctricos internos de la planta, sin conectividad MT propia.

| Columna               | Tipo       | Descripción                                                  |
|-----------------------|------------|--------------------------------------------------------------|
| id                    | UUID PK    |                                                              |
| elemento_generacion_id| UUID FK    | Planta a la que pertenece                                    |
| subtipo               | TEXT       | inversor_fv / aerogenerador / modulo_bateria / microturbina  |
| potencia_nominal_kw   | NUMERIC    | Potencia nominal de la unidad en kW                          |
| tension_salida_v      | NUMERIC    | Tensión de salida AC en voltios (típico: 400V BT)            |
| fabricante            | TEXT       |                                                              |
| modelo                | TEXT       |                                                              |
| num_serie             | TEXT       | (nullable)                                                   |

---

### `almacenamiento` — características específicas de baterías/sistemas de almacenamiento

Complementa `unidad_generacion` cuando `subtipo = modulo_bateria` o
`elemento_generacion.subtipo = almacenamiento`.

| Columna               | Tipo       | Descripción                                                  |
|-----------------------|------------|--------------------------------------------------------------|
| id                    | UUID PK    |                                                              |
| unidad_id             | UUID FK    | Unidad de generación (módulo batería)                        |
| tecnologia            | TEXT       | litio_ion / plomo_acido / flujo_vanadio / hidrogeno / otro   |
| capacidad_kwh         | NUMERIC    | Capacidad energética nominal en kWh                          |
| potencia_carga_kw     | NUMERIC    | Potencia máxima de carga                                     |
| potencia_descarga_kw  | NUMERIC    | Potencia máxima de descarga                                  |
| ciclos_vida           | INTEGER    | Ciclos de carga/descarga nominales                           |
| dod_pct               | NUMERIC    | Profundidad de descarga máxima en % (Depth of Discharge)     |
| soc_inicial_pct       | NUMERIC    | Estado de carga inicial nominal en %                         |

---

## Límites del modelo: qué queda fuera (lado DC/BT interno)

El lado DC de una planta fotovoltaica (paneles, strings, cajas de protección DC,
cableado CC) **no forma parte del modelo de red MT/AT**:

- Está regulado por el REBT (ITC-BT), no por el Reglamento AT
- El CIM no modela el interior DC de una planta FV
- El punto de entrada al modelo es el **inversor** (UGE), que es donde el recurso
  conecta a la red AC

```
Paneles FV → strings → caja combinadora → inversor ← PUNTO DE ENTRADA AL MODELO
                                             ↓
                                    transformador BT/MT
                                             ↓
                                      red MT (20 kV)
```

Lo mismo aplica al lado BT de un parque eólico: el modelo MT empieza en el
transformador del aerogenerador (o del parque), no dentro de la góndola.

---

## Conectividad con el resto del modelo

`elemento_generacion` **sí hereda de `elemento`** y por tanto tiene terminales
y conecta a nodos como cualquier otro elemento de red:

- Terminal 1 del elemento_generacion → nodo MT de conexión
- Si tiene transformador elevador propio: terminal 2 → nodo BT interno

La `unidad_generacion` no tiene terminales de red propios: está contenida dentro
de la envolvente de la planta (si existe).

---

## Pendiente de definir

- Curvas de potencia de aerogeneradores (tabla `curva_potencia_eolica`)
- Perfiles de generación FV (irradiación, inclinación, orientación)
- Requisitos de PO 12.2 para control de reactiva y tensión en punto de conexión
- Medidas y contadores SCADA asociados a la generación
