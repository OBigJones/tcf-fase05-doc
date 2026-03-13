# C4 — Nível 2: Containers

> **Modelo C4** — Nível 2 expande o sistema em seus containers (processos separados, serviços, bancos de dados, etc.) e mostra como eles se comunicam.

---

## Diagrama Geral dos Containers

```mermaid
C4Container
    title Containers — FIAP Tech Challenge Fase 5

    Person(usuario, "Usuário Final", "Acessa via navegador ou app mobile")

    System_Boundary(sistema, "Plataforma de Processamento de Vídeos") {

        Container(bff, "BFF Service", "Java 21 / Spring Boot 4.0.2", "Gateway HTTP único. Orquestra cadastro, autenticação, upload, listagem e download. Porta :8081")

        Container(user, "User Service", ".NET 8 / ASP.NET Core", "Cadastro e identificação de usuários. Valida CPF e e-mail. Porta :8082")

        Container(auth, "Auth Service", ".NET 8 / ASP.NET Core", "Autenticação, emissão e validação de JWT. Bloqueio de conta. Porta :8084")

        Container(vmgr, "Video Manager Service", ".NET 8 / ASP.NET Core", "Gerencia o ciclo de vida dos vídeos: upload, metadados, publicação em fila, status, download. Porta :8085")

        Container(movie, "Movie Processor Service", "Go", "Worker assíncrono. Extrai frames via FFmpeg, gera ZIP, publica resultado. Porta :8083")

        Container(comm, "Communication Service", ".NET 8 / ASP.NET Core", "Envia notificações por e-mail via SMTP. Porta :8086")

        ContainerDb(pg, "PostgreSQL 15", "PostgreSQL", "Três bancos isolados: auth_db, user_db, video_manager_db. Porta :5432")

        ContainerDb(redis, "Redis 7", "Redis", "Cache distribuído de usuários com TTL de 1 hora. Porta :6379")

        ContainerDb(rmq, "RabbitMQ 3", "RabbitMQ", "Broker de mensagens. Filas: video_queue e video_processed. Porta :5672")

        ContainerDb(storage, "Volume Compartilhado", "PVC Kubernetes (5Gi)", "Armazenamento de arquivos: /uploads (vídeos brutos) e /outputs (ZIPs gerados)")

        Container(mailhog, "MailHog", "SMTP / HTTP", "Servidor SMTP simulado para testes. Porta SMTP :1025, UI :8025")
    }

    Rel(usuario, bff, "Usa", "HTTPS / HTTP REST")

    Rel(bff, user, "Cria usuário", "HTTP POST /users")
    Rel(bff, auth, "Login, credencial, validação de token", "HTTP POST /auth/credentials, /login, /validate")
    Rel(bff, vmgr, "Upload, listagem, download", "HTTP POST|GET /videos")

    Rel(auth, user, "Verifica usuário ativo", "HTTP GET /users/{email}")

    Rel(user, pg, "Lê e escreve", "SQL (user_db)")
    Rel(user, redis, "Cache de usuários", "Redis (TTL 1h)")

    Rel(auth, pg, "Lê e escreve", "SQL (auth_db)")

    Rel(vmgr, pg, "Lê e escreve metadados", "SQL (video_manager_db)")
    Rel(vmgr, storage, "Grava arquivo bruto", "I/O arquivo (/uploads)")
    Rel(vmgr, rmq, "Publica tarefa", "AMQP — video_queue")
    Rel(rmq, movie, "Entrega tarefa", "AMQP — video_queue")
    Rel(movie, storage, "Lê vídeo bruto", "I/O arquivo (/uploads)")
    Rel(movie, storage, "Grava ZIP", "I/O arquivo (/outputs)")
    Rel(movie, rmq, "Publica resultado", "AMQP — video_processed")
    Rel(rmq, vmgr, "Entrega resultado", "AMQP — video_processed")
    Rel(vmgr, storage, "Lê ZIP (download)", "I/O arquivo (/outputs)")
    Rel(vmgr, comm, "Notifica resultado", "HTTP POST /communications/test")
    Rel(comm, mailhog, "Envia e-mail", "SMTP :1025")
```

---

## Diagrama — Fluxo de Mensageria RabbitMQ

```mermaid
sequenceDiagram
    participant C as Cliente
    participant BFF as BFF Service
    participant VM as Video Manager
    participant RMQ as RabbitMQ
    participant MP as Movie Processor
    participant ST as Shared Storage
    participant CS as Communication Service
    participant MH as MailHog

    C->>BFF: POST /api/bridge/process (multipart + Bearer token)
    BFF->>BFF: Valida token JWT (via Auth Service)
    BFF->>VM: POST /videos (multipart + X-UserId)
    VM->>ST: Grava vídeo bruto (/uploads)
    VM->>VM: Persiste metadados (PostgreSQL)
    VM->>RMQ: Publica video_queue { id, input_path, status: "queued" }
    VM-->>BFF: 200 OK (metadados do vídeo)
    BFF-->>C: 200 OK

    Note over MP: Worker em loop contínuo

    RMQ->>MP: Consome video_queue (prefetch=1)
    MP->>ST: Lê vídeo bruto (/uploads)
    MP->>MP: FFmpeg extrai frames (1fps → PNGs)
    MP->>ST: Grava ZIP (/outputs/frames_{id}.zip)
    MP->>MP: Remove arquivos temporários
    MP->>RMQ: Publica video_processed { id, output_path, status: "SUCCESS"|"ERROR" }
    MP->>RMQ: ACK manual da mensagem original

    RMQ->>VM: Consome video_processed
    VM->>VM: Atualiza status: Finished / ProcessingFailed (PostgreSQL)
    VM->>CS: POST /communications/test { email, fileName, status }
    CS->>MH: Envia e-mail via SMTP
    MH-->>CS: OK
    CS-->>VM: { sent: true }
```

---

## Diagrama — Fluxo de Autenticação e Cadastro

```mermaid
sequenceDiagram
    participant C as Cliente
    participant BFF as BFF Service
    participant US as User Service
    participant AS as Auth Service
    participant PG_U as PostgreSQL (user_db)
    participant PG_A as PostgreSQL (auth_db)
    participant RD as Redis

    Note over C,RD: Fluxo de Cadastro (Signup)

    C->>BFF: POST /api/auth/signup { name, email, cpf, password }
    BFF->>US: POST /users { name, email, cpf }
    US->>US: Valida CPF (algoritmo oficial)
    US->>US: Valida formato de e-mail
    US->>PG_U: Verifica duplicidade CPF e e-mail
    US->>PG_U: Persiste UserEntity
    US-->>BFF: 200 OK

    BFF->>AS: POST /auth/credentials { userId, email, password }
    AS->>US: GET /users/{email} (conta existe e está ativa?)
    US->>RD: Cache hit? (user:{email})
    RD-->>US: MISS
    US->>PG_U: SELECT usuario por e-mail
    US->>RD: Grava no cache (TTL 1h)
    US-->>AS: { name, email, cpf }
    AS->>AS: Hash SHA-256 da senha
    AS->>PG_A: Persiste UserCredential
    AS-->>BFF: 201 Created

    BFF->>AS: POST /auth/login { email, password }
    AS->>PG_A: Carrega UserCredential por e-mail
    AS->>AS: Verifica estado de bloqueio
    AS->>US: GET /users/{email}
    US->>RD: Cache hit? (user:{email})
    RD-->>US: HIT (TTL: 1h)
    US-->>AS: { name, email, cpf }
    AS->>AS: Compara hash SHA-256
    AS->>AS: Emite JWT (HMAC-SHA256)
    AS-->>BFF: { token, expiresAt }
    BFF-->>C: 200 OK { token }

    Note over C,RD: Fluxo de Login
    C->>BFF: POST /api/auth/login { email, password }
    BFF->>AS: POST /auth/login { email, password }
    AS->>PG_A: Carrega UserCredential
    AS->>AS: Verifica hash e bloqueio
    AS->>US: GET /users/{email}
    US-->>AS: Dados do usuário
    AS->>AS: Emite JWT
    AS-->>BFF: { token, expiresAt }
    BFF-->>C: 200 OK { token }
```

---

## Diagrama — Notificação por E-mail (SMTP)

```mermaid
flowchart LR
    VM["Video Manager Service\n(após consumir video_processed)"]
    CS["Communication Service\n:8086"]
    MH["MailHog\n:1025 (SMTP)\n:8025 (UI Web)"]
    USER(["Usuário\n(e-mail)"])

    VM -->|"POST /communications/test\n{ email, fileName, status }"| CS
    CS -->|"Seleciona template:\nSuccess ou Failure"| CS
    CS -->|"SMTP — envia e-mail"| MH
    MH -.->|"Visualização local\n(dev apenas)"| USER

    style VM fill:#1a6b9c,color:#fff
    style CS fill:#2d9e5f,color:#fff
    style MH fill:#c07800,color:#fff
```

**Templates de e-mail:**

| Status recebido | Assunto | Conteúdo |
|---|---|---|
| `Finished` | Processamento concluído: seu vídeo está disponível | Informa que o processamento foi bem-sucedido e o resultado está disponível para download. |
| Qualquer outro | Falha no processamento do seu vídeo | Informa que o processamento falhou e solicita nova tentativa. |

---

## Resumo das Responsabilidades por Container

| Container | Tipo | Linguagem | Banco | Fila | Porta |
|---|---|---|---|---|---|
| BFF Service | API Gateway | Java 21 | — | — | 8081 |
| User Service | Microserviço | .NET 8 | PostgreSQL (user_db) + Redis | — | 8082 |
| Movie Processor | Worker assíncrono | Go | — | Consumer: video_queue / Publisher: video_processed | 8083 |
| Auth Service | Microserviço | .NET 8 | PostgreSQL (auth_db) | — | 8084 |
| Video Manager | Microserviço | .NET 8 | PostgreSQL (video_manager_db) | Publisher: video_queue / Consumer: video_processed | 8085 |
| Communication Service | Microserviço | .NET 8 | — | — | 8086 |
| PostgreSQL 15 | Banco de dados | — | — | — | 5432 |
| Redis 7 | Cache | — | — | — | 6379 |
| RabbitMQ 3 | Broker | — | — | video_queue, video_processed | 5672 |
| Volume Compartilhado | Storage | — | — | — | — |
| MailHog | SMTP simulado | — | — | — | 1025 / 8025 |
