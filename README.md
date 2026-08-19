 # DocuMind Enterprise 🚀

> **Enterprise AI Agent Platform for Internal Knowledge Base & Automation**
> Built with .NET 9, PostgreSQL + pgvector, Clean Architecture, and AI Integration.

## 📌 Features
- **Smart Knowledge Base (RAG):** Upload PDF/Word documents and perform semantic search.
- **AI Action Agent:** Automate business tasks (e.g., Support Ticket creation) via .NET APIs.
- **Clean Architecture & CQRS:** Enterprise-grade design using .NET 9 & MediatR.

## 📐 Architecture & Design
### Use Case Diagram
```mermaid

flowchart LR
    %% Actors
    subgraph Actors["Actors"]
        User["User / Employee"]
        Admin["HR / Knowledge Admin"]
        Worker["Background Worker"]
    end

    %% Use Cases
    subgraph DocuMind["DocuMind Enterprise System"]
        UC1["UC-01: Search Knowledge Base (RAG Query)"]
        UC2["UC-02: Execute AI Action (Create Ticket)"]
        UC3["UC-03: Upload & Categorize Document"]
        UC4["UC-04: Manage Document Access Control"]
        UC5["UC-05: Ingest Document & Generate Embeddings"]
    end

    %% Relations
    User --> UC1
    User --> UC2
    Admin --> UC3
    Admin --> UC4
    Worker --> UC5

    UC3 -.->|Triggers| UC5

```

### ERD
```mermaid

erDiagram
    Users ||--o{ Documents : "uploads"
    Users ||--o{ SupportTickets : "creates"
    Documents ||--|{ DocumentChunks : "contains"

    Users {
        uuid Id PK
        string Email
        string FullName
        string Role "Admin / Employee"
        string Department "HR / IT / Sales"
        datetime CreatedAt
    }

    Documents {
        uuid Id PK
        uuid UploadedById FK
        string Title
        string FilePath
        string FileType "PDF / DOCX"
        string Department
        string Status "Pending / Processing / Processed / Failed"
        datetime UploadedAt
    }

    DocumentChunks {
        uuid Id PK
        uuid DocumentId FK
        int ChunkIndex
        string Content
        vector Embedding "1536-dim vector for pgvector"
        datetime CreatedAt
    }

    SupportTickets {
        uuid Id PK
        uuid UserId FK
        string Title
        string Description
        string Priority "Low / Medium / High"
        string Status "Open / InProgress / Closed"
        datetime CreatedAt
    }
```

### Sequence Diagram: Document Ingestion Flow
```mermaid

sequenceDiagram
    autonumber
    actor Admin
    participant API as .NET 9 Web API
    participant Storage as File Storage
    participant DB as PostgreSQL
    participant Worker as Background Service
    participant Ollama as Ollama / Azure OpenAI

    Admin->>API: POST /api/v1/documents/upload (PDF File)
    API->>Storage: Save raw PDF file
    API->>DB: Create Document record (Status = 'Pending')
    API-->>Admin: Return 202 Accepted (DocumentId)
    
    API->>Worker: Enqueue Task (DocumentId)
    
    rect rgb(240, 240, 240)
        Note over Worker,Ollama: Asynchronous Background Processing
        Worker->>DB: Update Status = 'Processing'
        Worker->>Storage: Read PDF content
        Worker->>Worker: Extract text & Chunking (~500 tokens)
        
        loop For each text Chunk
            Worker->>Ollama: POST /api/embeddings (Chunk Content)
            Ollama-->>Worker: Return Vector Float Array
            Worker->>DB: Insert into DocumentChunks table (pgvector)
        end
        
        Worker->>DB: Update Document Status = 'Processed'
    end
```

### Sequence Diagram: RAG Search Flow
```mermaid

sequenceDiagram
    autonumber
    actor User
    participant API as .NET 9 Web API
    participant Ollama as Ollama / Azure OpenAI
    participant DB as PostgreSQL (pgvector)
    participant SK as Semantic Kernel / AI Orchestrator

    User->>API: POST /api/v1/chat/query ("What is the sick leave policy in Ontario?")
    API->>Ollama: Generate Embedding for user query
    Ollama-->>API: Return Query Vector
    
    API->>DB: Query Top 3-5 nearest Chunks (Cosine Distance via EF Core)
    DB-->>API: Return Chunks Content + Document Metadata
    
    API->>SK: Send Prompt: [Context from Chunks] + [User Question]
    SK->>Ollama: Call LLM for answer synthesis
    Ollama-->>SK: Return synthesized response
    
    SK-->>API: Return Answer + Citation Metadata
    API-->>User: Return response (Answer + Document/Page citations)
```

### System Architecture
```mermaid

graph TD
    subgraph ClientLayer["Client / Presentation Layer"]
        UI["Web UI / Swagger UI"]
    end

    subgraph API Layer[".NET 9 Web API Layer"]
        Controllers["Controllers / Endpoints"]
        Middlewares["Auth & Exception Middlewares"]
    end

    subgraph AppLayer["Application Layer (Core)"]
        MediatR["CQRS Commands / Queries (MediatR)"]
        AIOrchestrator["AI Orchestrator (Semantic Kernel)"]
    end

    subgraph InfraLayer["Infrastructure Layer"]
        EFCore["EF Core DbContext (Npgsql)"]
        BgWorker["Background Processing Service"]
        FileStore["File Management System"]
    end

    subgraph DataLayer["External / Data Layer"]
        Postgres[(PostgreSQL + pgvector)]
        OllamaLocal["Ollama LLM & Embedding Service"]
    end

    UI --> Controllers
    Controllers --> Middlewares
    Controllers --> MediatR
    MediatR --> AIOrchestrator
    MediatR --> EFCore
    BgWorker --> FileStore
    BgWorker --> EFCore
    EFCore --> Postgres
    AIOrchestrator --> OllamaLocal
```
