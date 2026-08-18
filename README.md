 # DocuMind Enterprise 🚀

> **Enterprise AI Agent Platform for Internal Knowledge Base & Automation**
> Built with .NET 9, PostgreSQL + pgvector, Clean Architecture, and AI Integration.

## 📌 Features
- **Smart Knowledge Base (RAG):** Upload PDF/Word documents and perform semantic search.
- **AI Action Agent:** Automate business tasks (e.g., Support Ticket creation) via .NET APIs.
- **Clean Architecture & CQRS:** Enterprise-grade design using .NET 9 & MediatR.

## 📐 Architecture & Design
### Use Case Diagram
flowchart LR
    %% Actors
    subgraph Actors["Tác nhân"]
        User["User / Employee"]
        Admin["HR / Knowledge Admin"]
        Worker["Background Worker"]
    end

    %% Use Cases
    subgraph DocuMind["DocuMind Enterprise System"]
        UC1["UC-01: Tra cứu Quy trình (RAG Query)"]
        UC2["UC-02: Thực thi AI Action (Tạo Ticket)"]
        UC3["UC-03: Upload & Phân loại Document"]
        UC4["UC-04: Quản lý Phân quyền Tài liệu"]
        UC5["UC-05: Ingest Document & Generate Embeddings"]
    end

    %% Relations
    User --> UC1
    User --> UC2
    Admin --> UC3
    Admin --> UC4
    Worker --> UC5

    UC3 -.->|Triggers| UC5

### ERD
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

### Sequence Diagram: Document Ingestion Flow
sequenceDiagram
    autonumber
    actor Admin
    participant API as .NET 9 Web API
    participant Storage as File Storage
    participant DB as PostgreSQL
    participant Worker as Background Service
    participant Ollama as Ollama / Azure OpenAI

    Admin->>API: POST /api/v1/documents/upload (File PDF)
    API->>Storage: Lưu file PDF gốc
    API->>DB: Tạo Record Document (Status = 'Pending')
    API-->>Admin: Trả về 202 Accepted (DocumentId)
    
    API->>Worker: Enqueue Task (DocumentId)
    
    rect rgb(240, 240, 240)
        Note over Worker,Ollama: Tiến trình xử lý Background (Async)
        Worker->>DB: Cập nhật Status = 'Processing'
        Worker->>Storage: Đọc nội dung file PDF
        Worker->>Worker: Trích xuất Text & Chia nhỏ (Chunking ~500 tokens)
        
        loop Cho từng Chunk văn bản
            Worker->>Ollama: POST /api/embeddings (Chunk Content)
            Ollama-->>Worker: Trả về Vector Float Array
            Worker->>DB: Insert vào bảng DocumentChunks (pgvector)
        end
        
        Worker->>DB: Cập nhật Document Status = 'Processed'
    end

### Sequence Diagram: RAG Search Flow
sequenceDiagram
    autonumber
    actor User
    participant API as .NET 9 Web API
    participant Ollama as Ollama / Azure OpenAI
    participant DB as PostgreSQL (pgvector)
    participant SK as Semantic Kernel / AI Orchestrator

    User->>API: POST /api/v1/chat/query ("Nghỉ sick day ở Ontario được tính ra sao?")
    API->>Ollama: Tạo Embedding cho câu hỏi người dùng
    Ollama-->>API: Trả về Query Vector
    
    API->>DB: Query Top 3-5 Chunks gần nhất (Cosine Distance via EF Core)
    DB-->>API: Trả về Chunks Content + Title Document tương ứng
    
    API->>SK: Gửi Prompt: [Context từ Chunks] + [User Question]
    SK->>Ollama: Gọi LLM sinh câu trả lời
    Ollama-->>SK: Trả về văn bản đã tổng hợp
    
    SK-->>API: Trả về Answer + Citation Metadata
    API-->>User: Trả về kết quả (Câu trả lời + Dẫn chứng trang/file PDF)

### System Architecture
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
