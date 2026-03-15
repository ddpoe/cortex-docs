# System Diagrams

All diagrams use Mermaid syntax and can be rendered in GitHub, MkDocs, or any Mermaid-compatible viewer.

---

## 1. System Component Diagram

```mermaid
graph TD
    subgraph Project["Your Project (e.g. pm_image_analysis)"]
        SRC[".py modules"]
        DOCS["docs/ markdown files"]
        NB["notebooks/"]
    end

    subgraph dFlow_opt["dFlow (optional enrichment)"]
        WDB[".dflow/workflow.db"]
    end

    subgraph Cortex["Cortex"]
        MS["module_scanner.py\nAST reader"]
        DS["doc_scanner.py\nmarkdown extractor"]
        BUILDER["builder.py"]
        IDX[".cortex/index.json\nCortexIndex"]
        CLI["cortex build / render / validate"]
    end

    CONSUMERS["Consumers\n(agents, tools, databases)"]

    SRC -->|pure AST| MS
    WDB -.->|SQL reads, if present| MS
    DOCS --> DS
    NB -->|document node| DS
    MS --> BUILDER
    DS --> BUILDER
    BUILDER --> IDX
    IDX --> CLI
    IDX --> CONSUMERS
    CLI --> CONSUMERS
```

---

## 2. Resolution Level Flow (Agent Query)

```mermaid
sequenceDiagram
    participant Agent
    participant Cortex as cortex render / index.json
    participant FS as File System

    Agent->>Cortex: level=0 (full project)
    Cortex-->>Agent: [{id, title, type}] × N nodes (~5 tokens each)
    Note over Agent: Identifies region of interest

    Agent->>Cortex: level=1 on composite_process node
    Cortex-->>Agent: {id, title, level_1, tags, linked_ids}
    Note over Agent: Understands module contracts

    Agent->>Cortex: level=2 on function node
    Cortex-->>Agent: {id, title, level_1, level_2}
    Note over Agent: Needs implementation? Check level_3

    Agent->>Cortex: level=3 on function node
    Cortex-->>Agent: {file: "methods/binary_label/run.py", line_start: 45, line_end: 120}

    Agent->>FS: Read file at line range 45–120
    FS-->>Agent: Source code
    Note over Agent: Only fetches code when truly needed
```

---

## 3. Ontology Edge Type Graph

Valid edge combinations between node types. Only shown edges are permitted by the ontology.

```mermaid
graph LR
    AP[atomic_process]
    CP[composite_process]
    EN[entity]
    CN[constraint]
    DOC[document]

    CP -->|composes| AP
    CP -->|composes| CP
    CP -->|delegates_to| AP
    CP -->|delegates_to| CP
    CP -->|depends_on| AP
    CP -->|depends_on| CP
    CP -->|depends_on| EN
    AP -->|depends_on| AP
    AP -->|depends_on| EN
    AP -->|consumes| EN
    AP -->|produces| EN
    CP -->|consumes| EN
    CP -->|produces| EN
    CN -->|constrains| AP
    CN -->|constrains| CP
    CN -->|constrains| EN
    CN -->|validates| AP
    CN -->|validates| CP
    CN -->|supersedes| CN
    DOC -->|documents| AP
    DOC -->|documents| CP
    DOC -->|documents| EN
    DOC -->|documents| CN
    DOC -->|documents| DOC
    AP -->|supersedes| AP
    CP -->|supersedes| CP
```

---

## 4. Project Knowledge Graph (Conceptual Example)

A small example of what the graph looks like for a binary classification pipeline.

```mermaid
graph TD
    PKG["pm_image_analysis\ncomposite_process"]
    MOD["methods.binary_label.run\ncomposite_process"]
    WF["run_binary_label_pipeline\ncomposite_process @workflow"]
    TC["train_classifier\natomic_process @task"]
    EC["evaluate_classifier\natomic_process @task"]
    D3["D3 — isotonic calibration\nconstraint:decision"]
    T1["test_train_classifier\nconstraint:test"]
    DOC["analysis_module_split.md\ndocument"]
    X["X_train\nentity:data_artifact"]
    MDL["fitted_model\nentity:model_artifact"]

    PKG -->|composes| MOD
    MOD -->|composes| WF
    MOD -->|composes| TC
    MOD -->|composes| EC
    WF -->|delegates_to| TC
    WF -->|delegates_to| EC
    TC -->|consumes| X
    TC -->|produces| MDL
    EC -->|consumes| MDL
    D3 -->|constrains| TC
    T1 -->|validates| TC
    DOC -->|documents| TC
    DOC -->|documents| D3
```
