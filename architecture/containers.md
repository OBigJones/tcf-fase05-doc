# Arquitetura de Containers — FIAP Tech Challenge Fase 5

Este documento descreve os microserviços do sistema, suas responsabilidades, tecnologias, dependências e posição na arquitetura.

---

## Visão Geral dos Containers

O ecossistema é composto por **seis microserviços de aplicação** e **cinco componentes de infraestrutura compartilhada**, todos provisionados em um cluster Kubernetes.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        Cluster Kubernetes                               │
│                                                                         │
│  ┌────────────┐                                                         │
│  │ BFF Service│  ← Único ponto de entrada HTTP externo                  │
│  │  Java 21   │                                                         │
│  │   :8081    │                                                         │
│  └─────┬──────┘                                                         │
│        │                                                                │
│   ┌────┼─────────────────┐                                              │
│   ▼    ▼                 ▼                                              │
│  ┌──────────┐  ┌──────────────┐  ┌──────────────────┐                  │
│  │   User   │  │    Auth      │  │  Video Manager   │                  │
│  │ Service  │  │  Service     │  │    Service       │                  │
│  │  .NET 8  │  │   .NET 8     │  │     .NET 8       │                  │
│  │  :8082   │  │   :8084      │  │     :8085        │                  │
│  └────┬─────┘  └──────┬───────┘  └────────┬─────────┘                  │
│       │               │                   │                             │
│   ┌───▼──┐  ┌────────▼──┐   ┌─────────────▼──────────────────────────┐ │
│   │ PG   │  │ PG        │   │ PG    RabbitMQ    Shared Storage        │ │
│   │user  │  │auth       │   │video  :5672      /uploads  /outputs     │ │
│   └──────┘  └──────┬───-┘   └───────────┬────────────────────────────┘ │
│                    │                    │                               │
│                    │  ┌─────────────────▼──────────┐                   │
│                    │  │   Movie Processor           │                   │
│                    │  │        Go  :8083            │                   │
│                    │  └─────────────────────────────┘                   │
│                    │                    │                               │
│                    │        ┌───────────▼──────────────────┐            │
│                    │        │  Communication Service       │            │
│                    │        │       .NET 8  :8086          │            │
│                    │        └───────────────┬──────────────┘            │
│                    │                        │                           │
│                    │                   ┌────▼────┐                      │
│                    │                   │ MailHog │                      │
│                    │                   │  :1025  │                      │
│                    │                   └─────────┘                      │
│                    │                                                    │
│                 ┌──▼──────────────────┐                                 │
│                 │        Redis        │                                 │
│                 │      :6379          │                                 │
│                 └─────────────────────┘                                 │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Microserviços

### BFF Service

| Atributo | Valor |
|---|---|
| Linguagem / Framework | Java 21 / Spring Boot 4.0.2 |
| Porta | `8081` |
| Arquitetura | Clean Architecture + Hexagonal (Ports & Adapters) |
| Banco de dados | Nenhum |
| Mensageria | Nenhuma (delegada a serviços downstream) |

**Responsabilidades:**
- Único ponto de entrada HTTP para clientes externos (web/mobile).
- Orquestração do fluxo de cadastro: cria o perfil no User Service, registra credenciais no Auth Service e retorna o JWT ao cliente.
- Validação stateless de tokens JWT em cada requisição autenticada, delegando ao Auth Service via `POST /auth/validate`.
- Repasse de upload de vídeos, listagem e download ao Video Manager Service.

**Integrações:**

| Serviço | Protocolo | Operações |
|---|---|---|
| Auth Service | HTTP REST | `POST /auth/credentials`, `POST /auth/login`, `POST /auth/validate` |
| User Service | HTTP REST | `POST /users` |
| Video Manager Service | HTTP REST | `POST /videos`, `GET /videos`, `GET /videos/{id}/result` |

---

### User Service

| Atributo | Valor |
|---|---|
| Linguagem / Framework | .NET 8 / ASP.NET Core |
| Porta | `8082` |
| Arquitetura | Clean Architecture |
| Banco de dados | PostgreSQL (`user_db`) |
| Cache | Redis (TTL 1 hora) |
| Mensageria | Nenhuma |

**Responsabilidades:**
- Cadastro de usuários com validação de CPF (algoritmo oficial) e e-mail.
- Verificação de duplicidade por CPF e por e-mail antes do cadastro.
- Identificação de usuários por CPF ou e-mail, com cache Redis para leituras repetidas.

**Consumido por:** BFF Service (cadastro) e Auth Service (verificação de usuário ativo).

---

### Auth Service

| Atributo | Valor |
|---|---|
| Linguagem / Framework | .NET 8 / ASP.NET Core |
| Porta | `8084` (HTTP) / `9094` (HTTPS) |
| Arquitetura | Clean Architecture |
| Banco de dados | PostgreSQL (`auth_db`) |
| Mensageria | Nenhuma |

**Responsabilidades:**
- Registro de credenciais (hash de senha com SHA-256).
- Autenticação de usuários com bloqueio de conta após 5 tentativas com falha (5 minutos).
- Emissão de tokens JWT assinados com HMAC-SHA256 (TTL padrão: 60 minutos em produção).
- Validação de tokens JWT, retornando claims embutidas (`userId`, `email`, `role`, `expiresAt`).
- Consulta ao User Service para verificar se o usuário existe e está ativo antes de autenticar.

**Consumido por:** BFF Service.

---

### Video Manager Service

| Atributo | Valor |
|---|---|
| Linguagem / Framework | .NET 8 / ASP.NET Core |
| Porta | `8085` |
| Arquitetura | Clean Architecture |
| Banco de dados | PostgreSQL (`video_manager_db`) |
| Mensageria | RabbitMQ (publisher: `video_queue` / consumer: `video_processed`) |
| Storage | Volume compartilhado (`/uploads`, `/outputs`) |

**Responsabilidades:**
- Receber uploads de vídeos via `multipart/form-data` (limite de 500 MB).
- Gravar o arquivo bruto no storage compartilhado, em `raw/<userId>/<videoId>/`.
- Persistir metadados do vídeo no PostgreSQL (nome, tamanho, content-type, FPS solicitado, status).
- Publicar mensagem na fila `video_queue` para acionar o processamento pelo Movie Processor.
- Consumir a fila `video_processed` para atualizar o status do vídeo.
- Notificar o usuário via chamada HTTP ao Communication Service após a conclusão.
- Listar vídeos do usuário com paginação e disponibilizar o resultado para download.

**Ciclo de vida do vídeo:**  
`PendingUpload (0)` → `Processing (1)` → `Finished (2)` | `ProcessingFailed (3)`

---

### Movie Processor Service

| Atributo | Valor |
|---|---|
| Linguagem / Framework | Go |
| Porta | `8083` |
| Arquitetura | Clean Architecture |
| Banco de dados | Nenhum |
| Mensageria | RabbitMQ (consumer: `video_queue` / publisher: `video_processed`) |
| Storage | Volume compartilhado (leitura de `/uploads`, escrita em `/outputs`) |
| Ferramenta externa | FFmpeg |

**Responsabilidades:**
- Worker assíncrono puro — não expõe API REST.
- Consumir mensagens da fila `video_queue` (prefetch = 1, ACK manual).
- Extrair frames do vídeo utilizando FFmpeg (`1 frame/segundo`, formato PNG).
- Empacotar todos os frames em um arquivo `.zip`.
- Publicar o resultado (caminho do ZIP e status) na fila `video_processed`.
- Realizar retry automático em caso de falha (até 3 tentativas com intervalo de 2 segundos).
- Limpar arquivos temporários após geração do ZIP.

**Réplicas:** mínimo 2, máximo 3 (HPA).

---

### Communication Service

| Atributo | Valor |
|---|---|
| Linguagem / Framework | .NET 8 / ASP.NET Core |
| Porta | `8086` (HTTP) / `9096` (HTTPS) |
| Arquitetura | Clean Architecture |
| Banco de dados | Nenhum |
| Mensageria | RabbitMQ (implementação presente no código, porém **não ativa**) |
| SMTP | MailHog (`localhost:1025` em desenvolvimento) |

**Responsabilidades:**
- Receber requisição HTTP do Video Manager Service (`POST /communications/test`).
- Selecionar o template de e-mail adequado com base no status do processamento (`Success` ou `Failure`).
- Construir o corpo do e-mail e enviá-lo via SMTP.
- Retornar confirmação ao chamador (`{ sent: true/false }`).

> O código contém uma implementação de consumidor RabbitMQ para a fila `communication_queue` (`RabbitMqConsumer`, `RabbitMqConsumerHostedService`), porém essa implementação está presente no código apenas como legado e **não está ativa no fluxo atual do sistema**.

---

## Componentes de Infraestrutura

### PostgreSQL 15

Instância única que hospeda três bancos de dados isolados por nome. O script de inicialização (`init.sql`, injetado via ConfigMap) cria automaticamente todos os bancos e tabelas na primeira execução.

| Banco | Serviço |
|---|---|
| `auth_db` | Auth Service |
| `user_db` | User Service |
| `video_manager_db` | Video Manager Service |

### Redis 7

Utilizado pelo User Service como cache distribuído para leituras de usuários (TTL de 1 hora).

### RabbitMQ 3

Broker de mensagens assíncronas implantado como StatefulSet para garantir identidade de rede estável. Desacopla o Video Manager do Movie Processor.

| Fila | Produtor | Consumidor |
|---|---|---|
| `video_queue` | Video Manager Service | Movie Processor Service |
| `video_processed` | Movie Processor Service | Video Manager Service |

### MailHog

Servidor SMTP simulado para o ambiente de desenvolvimento. Captura e-mails enviados pelo Communication Service sem entregá-los de fato. Interface web acessível em `localhost:8025`.

### Volume Compartilhado (PVC)

PersistentVolumeClaim `fase-05-movie-storage` (5Gi, `hostpath`) compartilhado entre Video Manager e Movie Processor:

- `uploads/` — arquivos de vídeo brutos gravados pelo Video Manager após o upload.
- `outputs/` — arquivos `.zip` com frames gerados pelo Movie Processor.

Ambos os Deployments utilizam `initContainer` com `busybox` para garantir permissões corretas no diretório montado antes da inicialização do container principal.
