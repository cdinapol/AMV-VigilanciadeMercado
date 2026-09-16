# AMV — Vigilancia de Mercado con Azure AI Foundry
### Arquitectura de referencia y diagrama lógico de la aplicación

> Basado en *"Caso 1 - AMV_Vigilancia_Mercado_AI_Foundry.docx"* (v1.0, 16-sep-2026).
> Principio de diseño: **arquitectura agnóstica al modelo**. Cada endpoint de Foundry
> puede intercambiarse sin rediseñar la plataforma. Para este prototipo, el modelo
> **MAI (MAI-1)** se usa **únicamente** en el componente **Síntesis / redacción de
> hallazgos**; el resto de componentes usa el servicio recomendado en la matriz de
> decisión del documento (Azure AI Speech, embeddings OpenAI, GPT-4o), dejando el
> razonamiento profundo pendiente de bake-off.

---

## 1. Arquitectura de referencia en Azure

```mermaid
flowchart TB
    subgraph ING["1. Ingesta"]
        BLOB[("Azure Blob Storage<br/>Grabaciones crudas")]
        EG["Event Grid<br/>Disparo por evento"]
        BLOB --> EG
    end

    subgraph TRX["2. Transcripción"]
        SPEECH["Azure AI Speech (batch)<br/>Timestamps por palabra + diarización"]
    end

    subgraph ENR["3. Enriquecimiento"]
        NER["Azure AI Language<br/>Custom NER · detección PII"]
        EXT["GPT-4o<br/>Extracción estructurada (JSON schema)"]
        CS["Azure Content Safety"]
    end

    subgraph IDX["4. Indexación y búsqueda"]
        EMB["text-embedding-3-large (OpenAI)"]
        SEARCH[("Azure AI Search<br/>BM25 + vectorial + semantic ranker")]
    end

    subgraph RAZ["5. Razonamiento y síntesis"]
        AGENT["Azure AI Foundry Agent Service<br/>Orquestador"]
        REASON["Modelo de razonamiento<br/>(pendiente bake-off: MAI-DS-R1 vs. o-series)"]
        CORR["Agent tool: correlación transaccional<br/>vs. base de operaciones"]
        SYN["MAI-1<br/>Síntesis / redacción de hallazgos"]
    end

    subgraph WB["6. Workbench del investigador"]
        UI["App / portal<br/>búsqueda · playback · reporte citado"]
    end

    subgraph GOV["Gobierno, seguridad y cumplimiento (transversal)"]
        PURVIEW["Microsoft Purview<br/>clasificación · eDiscovery"]
        RBAC["Entra ID · RBAC"]
        AUDIT["Auditoría / cadena de custodia"]
        EVAL["Evaluaciones Foundry<br/>groundedness"]
    end

    EG --> SPEECH --> NER --> EXT
    NER --> CS
    EXT --> EMB --> SEARCH
    SEARCH --> AGENT
    AGENT --> REASON
    AGENT --> CORR
    REASON --> SYN
    CORR --> SYN
    SYN --> UI
    SEARCH --> UI

    classDef mai fill:#cff3e9,stroke:#1f9d77,stroke-width:2px,color:#0b3d2e;
    classDef openai fill:#dceefb,stroke:#2e75b6,stroke-width:2px,color:#1b2a4a;
    classDef tbd fill:#fff4ce,stroke:#b8860b,stroke-width:2px,color:#4a3b00;
    classDef infra fill:#f1efea,stroke:#6b6b6b,stroke-width:1.5px,color:#333;
    classDef gov fill:#e5dff5,stroke:#6a4fb6,stroke-width:1.5px,color:#3b2a63;

    class SYN mai;
    class EXT,EMB openai;
    class REASON tbd;
    class BLOB,EG,SPEECH,NER,CS,SEARCH,AGENT,CORR,UI infra;
    class PURVIEW,RBAC,AUDIT,EVAL gov;
```

### Mapeo de modelo/servicio por componente

| Capa | Componente | Servicio / modelo | Estado |
|---|---|---|---|
| Ingesta | Almacenamiento y disparo | Azure Blob Storage + Event Grid | Infraestructura |
| Transcripción | Speech-to-text, diarización | **Azure AI Speech** (batch) | Recomendado (maduro) |
| Enriquecimiento | Entidades / PII | Azure AI Language (Custom NER) | Recomendado |
| Enriquecimiento | Extracción estructurada (instrumento, contraparte, cantidad, valor) | **GPT-4o** (JSON schema) | Recomendado |
| Búsqueda | Embeddings semánticos | **text-embedding-3-large** (OpenAI) | Recomendado |
| Búsqueda | Índice híbrido | Azure AI Search (BM25 + vector + semantic ranker) | Infraestructura |
| Razonamiento | Detección de colusión / insider trading | MAI-DS-R1 *vs.* o-series | **Pendiente bake-off** |
| Correlación | Cruce con base de operaciones | Agent tools/functions | Infraestructura |
| **Síntesis / redacción de hallazgos** | Borrador de informe con citas | **MAI-1** | **Fijado para este prototipo** |
| Gobierno | Seguridad, trazabilidad, cumplimiento | Content Safety, Purview, Entra ID, evaluaciones Foundry | Transversal |

---

## 2. Diagrama lógico de la aplicación (flujo de una investigación)

```mermaid
sequenceDiagram
    actor Inv as Investigador
    participant UI as Workbench (app)
    participant Search as Azure AI Search
    participant Agent as Foundry Agent Service
    participant Corr as Tool: correlación transaccional
    participant MAI as MAI-1 (síntesis)

    Inv->>UI: Busca símbolo / palabra clave / hablante
    UI->>Search: Consulta híbrida (BM25 + vectorial + semántica)
    Search-->>UI: Segmentos + timestamp exacto + confianza
    Inv->>UI: Reproduce audio, revisa evidencia, selecciona hallazgos
    UI->>Agent: Solicita síntesis de hallazgos seleccionados
    Agent->>Corr: Correlaciona entidades extraídas vs. operaciones reales
    Corr-->>Agent: Coincidencias (instrumento, hora, cantidad, contraparte)
    Agent->>MAI: Genera borrador de informe (citado, con groundedness)
    MAI-->>Agent: Texto redactado + referencias a evidencia
    Agent-->>UI: Informe + enlaces a grabación/timestamp/hablante
    UI-->>Inv: Revisión humana obligatoria, edición y aprobación
```

Puntos clave del flujo lógico:

- **Human-in-the-loop no negociable**: MAI-1 solo *redacta*; el investigador aprueba.
- **Citación obligatoria**: cada afirmación del borrador enlaza a grabación + timestamp + hablante.
- **Confidence scoring por campo**: valores bajo umbral se marcan para revisión manual.
- **Cadena de custodia**: cada acción (búsqueda, reproducción, generación, aprobación) queda auditada.

---

## 3. Por qué MAI-1 solo en Síntesis / redacción de hallazgos

Según la matriz de decisión del documento original, este es el único componente donde
la recomendación es **"indistinto"** entre MAI y OpenAI (elegir por costo/latencia). Es
el punto de menor riesgo probatorio para introducir un modelo de Microsoft:

- No participa en la extracción estructurada (ese dato ya quedó fijado y citado antes de llegar a MAI).
- No decide sobre colusión/insider trading (eso es razonamiento, pendiente de bake-off).
- Su salida siempre pasa por revisión humana antes de convertirse en evidencia.
- Reduce el "blast radius" de adoptar un modelo más nuevo mientras se recopila evidencia
  de desempeño para ampliarlo a otros componentes.

---

## 4. Prototipo de demostración

Ver [index.html](index.html): mockup interactivo (datos simulados, sin backend) que
representa el workbench del investigador — búsqueda híbrida, detalle de segmento con
extracción estructurada y correlación transaccional, y el panel de **Síntesis con MAI-1**
con citas y revisión humana. Incluye una leyenda visual de qué modelo/servicio impulsa
cada componente, para reforzar la conversación de arquitectura con el cliente.

## 5. Próximos pasos (del documento original)

1. Confirmar disponibilidad de modelos MAI y OpenAI en el tenant/región de AMV.
2. Construir un conjunto dorado anotado con grabaciones representativas.
3. Ejecutar bake-off (transcripción, extracción, razonamiento) con evaluaciones de Foundry.
4. Extender esta PoC a un end-to-end con datos reales de AMV.
5. Definir modelo de gobierno y cadena de custodia con áreas legal y de cumplimiento.
