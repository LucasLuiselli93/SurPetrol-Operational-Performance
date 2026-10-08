# SurPetrol - Oil & Gas Integrated Operational Performance Dashboard

Plataforma de Business Intelligence diseñada para la dirección ejecutiva y operativa de **SurPetrol**, orientada a la supervisión integral de activos de upstream, midstream, downstream y comercialización en Argentina. El dashboard centraliza indicadores de volumen de producción física, márgenes financieros (EBITDA), utilización de plantas de refino, seguridad operacional (TRIR) y compliance regulatorio ante la SEC y CNV.

---

## 📊 Visualización Ejecutiva

![SurPetrol Operational Performance Overview](assets/dashboard_overview.png)

> **Vista General (Marzo 2026):** Monitoreo macro de producción en cuencas clave (Vaca Muerta, Golfo San Jorge, Austral), balance de ingresos vs. precio realizado, mapa geográfico de campos y scorecard de cumplimiento normativo.

---

## 📌 Escenario de Negocio y Requerimientos

La operación integral de hidrocarburos presentaba silos de información críticos:
- **Desconexión entre campo y finanzas:** Los datos de telemetría de boca de pozo (SCADA) no estaban alineados con la facturación y los precios netos realizados (Netback) por barril equivalente.
- **Heterogeneidad de métricas por Unidad de Negocio:** Upstream operaba en barriles equivalentes día (`kboe/d`), Downstream medía tasas de corte y disponibilidad de refinería (`kbbl/d`), mientras que Finanzas consolidaba en `USD M` bajo normativas contables internacionales.
- **Control de Riesgo y Compliance:** Necesidad de auditar en tiempo real la exposición regulatoria (plazos de entrega de reportes a CNV/SEC) y la tasa de accidentabilidad (TRIR) vinculada a cada cluster de perforación.

---

## 🛠️ Stack Tecnológico

- **Herramienta de BI:** Power BI Desktop / Service (Capacidad Fabric/Premium).
- **Arquitectura de Datos:** Esquema de Constelación (*Fact Constellation*) integrando hechos físicos y financieros a diferentes granularidades.
- **Fuentes de Datos:** ERP corporativo (SAP), Data Warehouse central, telemetría SCADA y feeds de mercado de commodities (Brent/WTI/Gas).
- **Cálculo Analítico:** DAX avanzado (Time Intelligence multidivisa, agregaciones semicomitidas y normalización de unidades físicas).
- **Seguridad y Distribución:** Row-Level Security (RLS) dinámico según unidad de negocio y región geográfica.

---

## 📐 Arquitectura de Datos y Modelo Conceptual

Para evitar duplicidad y permitir el cruce de producción diaria con resultados contables mensuales, se estructuró un modelo multidimensional con dimensiones conformadas:

- **Tablas de Hechos (Fact Tables):**
  - `Fact_Produccion_Diaria`: Volumen bruto y neto por pozo/batería (`kboe/d`, gas, petróleo, agua).
  - `Fact_Finanzas_Mensual`: Ingresos netos, OPEX, CAPEX y EBITDA por unidad de negocio y cuenca.
  - `Fact_Refineria_Metricas`: Capacidad nominal, throughput real y paradas de planta.
  - `Fact_Incidentes_HSE`: Horas hombre trabajadas e incidentes registrables para el cálculo del TRIR.
  - `Fact_Regulatory_Milestones`: Estatus y fechas límite de presentaciones regulatorias (SEC/CNV).

- **Dimensiones Conformadas:**
  - `Dim_Cuencas_Activos`: Jerarquía Cuenca → Bloque / Concesión → Campo / Yacimiento.
  - `Dim_UnidadNegocio`: Upstream, Midstream, Downstream, Comercialización.
  - `Dim_Calendario`: Dimensión de tiempo canónica con soporte para cierres contables y calendarios operativos.
  - `Dim_KPI_Targets`: Metas operativas y umbrales de criticidad.

---

## 🧠 Desafíos Técnicos Resueltos

### 1. Homogeneización de Unidades Físicas a Barril Equivalente de Petróleo (BOE)
El gas natural y el crudo provienen de sistemas SCADA en unidades incompatibles ($m^3/d$ y $Mm^3/d$). Se implementó una lógica de conversión estandarizada dentro de la capa semántica basada en factores de poder calorífico según la cuenca de origen:

```dax
Produccion_kboe_d = 
SUMX(
    Dim_Cuencas_Activos,
    DIVIDE(
        [Produccion_Petroleo_m3] * 6.2898 + ([Produccion_Gas_Mm3] * 1000 * RELATED(Dim_Factores_Conversion[Factor_Gas_BOE])),
        [Dias_Activos_Mes] * 1000,
        0
    )
)
