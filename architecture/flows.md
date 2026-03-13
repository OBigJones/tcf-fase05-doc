# Fluxos de Comunicação — FIAP Tech Challenge Fase 5

Este documento descreve os fluxos de comunicação do ecossistema, detalhando as interações síncronas (HTTP), assíncronas (RabbitMQ) e o uso do storage compartilhado.

---

## Mapa de Comunicação

```
Cliente (Web/Mobile)
    │ HTTP REST (Bearer JWT)
    ▼
BFF Service (:8081)
    │
    ├─── HTTP ──► Auth Service (:8084) ──► PostgreSQL (auth_db)
    │                   │
    │                   └─── HTTP ──► User Service (:8082)
    │
    ├─── HTTP ──► User Service (:8082) ──► PostgreSQL (user_db)
    │                                 ──► Redis (:6379)
    │
    └─── HTTP ──► Video Manager Service (:8085) ──► PostgreSQL (video_manager_db)
                        │                      ──► Shared Storage (/uploads)
                        │
                        │  [AMQP — video_queue]
                        ▼
                  RabbitMQ (:5672)
                        │
                        │  [AMQP — video_queue]
                        ▼
                  Movie Processor (:8083)
                        │  lê vídeo bruto ──► Shared Storage (/uploads)
                        │  grava ZIP      ──► Shared Storage (/outputs)
                        │
                        │  [AMQP — video_processed]
                        ▼
                  RabbitMQ (:5672)
                        │
                        │  [AMQP — video_processed]
                        ▼
                  Video Manager Service (:8085)
                        │  atualiza status ──► PostgreSQL (video_manager_db)
                        │
                        │  HTTP POST /communications/test
                        ▼
                  Communication Service (:8086)
                        │  SMTP
                        ▼
                   MailHog (:1025)
```

---

## Fluxos Detalhados

### Fluxo 1 — Cadastro de Usuário

**Protocolo:** HTTP síncrono  
**Iniciador:** Cliente (web/mobile)

```
Cliente
  └─► POST /api/auth/signup  (BFF Service)
        ├─► POST /users       (User Service)
        │     ├─ Valida CPF (algoritmo oficial no domínio)
        │     ├─ Valida e-mail
        │     ├─ Verifica duplicidade por CPF (PostgreSQL)
        │     ├─ Verifica duplicidade por e-mail (PostgreSQL)
        │     └─ Persiste UserEntity (PostgreSQL)
        │
        ├─► POST /auth/credentials  (Auth Service)
        │     ├─ GET /users/{email} → User Service (verifica se existe e está ativo)
        │     ├─ Verifica se e-mail já possui credencial (PostgreSQL)
        │     ├─ Gera hash SHA-256 da senha
        │     └─ Persiste UserCredential (PostgreSQL)
        │
        └─► POST /auth/login  (Auth Service)
              ├─ Carrega credencial por e-mail (PostgreSQL)
              ├─ Verifica estado de bloqueio
              ├─ GET /users/{email} → User Service (verifica conta ativa)
              ├─ Compara hash SHA-256
              └─ Emite JWT (HMAC-SHA256, TTL configurável)

BFF ◄── { token: "eyJ..." }
Cliente ◄── 200 OK { token }
```

---

### Fluxo 2 — Login

**Protocolo:** HTTP síncrono  
**Iniciador:** Cliente

```
Cliente
  └─► POST /api/auth/login  (BFF Service)
        └─► POST /auth/login  (Auth Service)
              ├─ Carrega UserCredential por e-mail (PostgreSQL)
              ├─ Verifica bloqueio (> 5 tentativas → bloqueio 5 min)
              ├─ GET /users/{email} → User Service (conta ativa?)
              ├─ Em falha: incrementa contador, bloqueia se atingiu limite
              └─ Em sucesso: reseta contador, atualiza lastLoginAt, emite JWT

Cliente ◄── 200 OK { token }
```

---

### Fluxo 3 — Validação de Token (por requisição autenticada)

**Protocolo:** HTTP síncrono — intercepta toda requisição protegida no BFF  
**Iniciador:** Filtro `TokenAuthenticationFilter` no BFF Service

```
BFF (TokenAuthenticationFilter)
  └─► POST /auth/validate  (Auth Service)
        ├─ Analisa e valida o JWT (assinatura, expiração)
        └─ Retorna claims: { userId, email, role, expiresAt }

[Se válido] → Spring SecurityContext recebe UserPrincipal com userId e email
[Se inválido] → rejeita a requisição com HTTP 401
```

---

### Fluxo 4 — Upload de Vídeo

**Protocolo:** HTTP (upload) + AMQP assíncrono (processamento)  
**Iniciador:** Cliente autenticado

```
Cliente (Bearer token)
  └─► POST /api/bridge/process  (BFF Service)
        ├─► POST /auth/validate  (Auth Service) — extrai e-mail do token
        └─► POST /videos  (Video Manager Service)
              ├─ Grava arquivo bruto em /video-storage/raw/<email>/<uuid>/
              ├─ Persiste metadados (PostgreSQL): status = PendingUpload
              ├─ Atualiza status → Processing
              └─ Publica em video_queue (RabbitMQ):
                   {
                     "id": "<uuid>",
                     "input_path": "/video-storage/raw/.../video.mp4",
                     "output_path": "",
                     "status": "queued"
                   }

BFF ◄── 202 Accepted (ou 200 OK com metadados do vídeo)
```

---

### Fluxo 5 — Processamento Assíncrono do Vídeo

**Protocolo:** AMQP (RabbitMQ) + I/O de arquivo (Storage Compartilhado)  
**Iniciador:** Publicação na fila `video_queue` pelo Video Manager Service

```
RabbitMQ (video_queue)
  └─► Movie Processor (VideoUseCase.ExecuteWorker — loop contínuo)
        └─► processWithRetry() — até 3 tentativas (intervalo: 2s)
              ├─► FFmpegProcessor.ExtractFrames()
              │     FFmpeg: ffmpeg -i <input> -vf fps=1 frame_%04d.png
              │     Frames PNG gravados em /app/temp/<ts>_<id>/
              │
              ├─► LocalStorage.CreateZip()
              │     Compacta frames em /app/outputs/frames_<id>.zip
              │
              └─► LocalStorage.Delete()
                    Remove diretório temporário

Movie Processor → Publica em video_processed (RabbitMQ):
  {
    "id": "<uuid>",
    "input_path": "/app/uploads/video.mp4",
    "output_path": "/app/outputs/frames_<id>.zip",
    "status": "SUCCESS" | "ERROR"
  }

Movie Processor → ACK manual da mensagem original
```

---

### Fluxo 6 — Recebimento do Resultado e Notificação

**Protocolo:** AMQP (resultado) + HTTP (notificação) + SMTP (e-mail)  
**Iniciador:** Publicação em `video_processed` pelo Movie Processor

```
RabbitMQ (video_processed)
  └─► Video Manager (VideoProcessedConsumerService — BackgroundService)
        └─► UpdateResultHandler.HandleAsync()
              ├─ Busca vídeo por ID (PostgreSQL)
              ├─ video.MarkFinished() ou video.MarkFailed() (regra de domínio)
              ├─ Persiste atualização de status (PostgreSQL)
              └─► POST /communications/test  (Communication Service)
                    {
                      "email": "usuario@email.com",
                      "fileName": "video.mp4",
                      "status": "Finished" | "Failed"
                    }
                    └─► SmtpEmailSender → MailHog (SMTP :1025)
                          └─► E-mail entregue ao destinatário
```

---

### Fluxo 7 — Listagem e Download de Resultados

**Protocolo:** HTTP síncrono  
**Iniciador:** Cliente autenticado

```
[Listagem]
Cliente (Bearer token)
  └─► GET /api/bridge/list?skip=0&take=10  (BFF Service)
        ├─► POST /auth/validate  (Auth Service)
        └─► GET /videos?skip=0&take=10  (Video Manager, header X-UserId: email)
              └─ Consulta paginada na tabela videos (PostgreSQL)
Cliente ◄── [ { VideoId, OriginalFileName, Status, ResultAvailable, ... } ]

[Download]
Cliente (Bearer token)
  └─► GET /api/bridge/{id}/download  (BFF Service)
        ├─► POST /auth/validate  (Auth Service)
        └─► GET /videos/{id}/result  (Video Manager, header X-UserId: email)
              ├─ Valida propriedade do vídeo (PostgreSQL)
              └─ Stream do arquivo ZIP (Shared Storage /outputs/)
Cliente ◄── Content-Disposition: attachment; filename="result-{id}.zip"
```

---

## Comunicação Síncrona — Resumo

| Origem | Destino | Método + Endpoint | Finalidade |
|---|---|---|---|
| BFF | User Service | `POST /users` | Criar perfil de usuário |
| BFF | Auth Service | `POST /auth/credentials` | Registrar credenciais |
| BFF | Auth Service | `POST /auth/login` | Autenticar e obter JWT |
| BFF | Auth Service | `POST /auth/validate` | Validar token em requisições protegidas |
| BFF | Video Manager | `POST /videos` | Upload de vídeo |
| BFF | Video Manager | `GET /videos` | Listar vídeos do usuário |
| BFF | Video Manager | `GET /videos/{id}/result` | Download do resultado |
| Auth Service | User Service | `GET /users/{email}` | Verificar existência e status do usuário |
| Video Manager | Communication Service | `POST /communications/test` | Notificar resultado do processamento |

---

## Comunicação Assíncrona — Filas RabbitMQ

### Fila `video_queue`

| Atributo | Valor |
|---|---|
| Tipo | Durável |
| Produtor | Video Manager Service |
| Consumidor | Movie Processor Service |
| Prefetch (QoS) | 1 (uma mensagem por vez) |
| ACK | Manual (após processamento) |

**Conteúdo da mensagem:**
```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "input_path": "/app/uploads/meu_video.mp4",
  "output_path": "",
  "status": "PENDING"
}
```

### Fila `video_processed`

| Atributo | Valor |
|---|---|
| Tipo | Durável |
| Produtor | Movie Processor Service |
| Consumidor | Video Manager Service |

**Conteúdo da mensagem:**
```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "input_path": "/app/uploads/meu_video.mp4",
  "output_path": "/app/outputs/frames_550e8400-....zip",
  "status": "SUCCESS"
}
```

**Mapeamento de status recebido pelo Video Manager:**

| Valor recebido | Estado final |
|---|---|
| `finished`, `success`, `succeeded` | `VideoStatus.Finished` |
| `failed`, `error`, `processing_failed` | `VideoStatus.ProcessingFailed` |
| Qualquer outro | `VideoStatus.Processing` |

---

## Storage Compartilhado

O volume `fase-05-movie-storage` (PVC Kubernetes, 5Gi) é o ponto de integração de arquivos entre Video Manager e Movie Processor.

| Diretório | Quem escreve | Quem lê |
|---|---|---|
| `/uploads/` (ou `raw/<userId>/<videoId>/`) | Video Manager Service | Movie Processor Service |
| `/outputs/` | Movie Processor Service | Video Manager Service (para download) |

O caminho exato do arquivo é comunicado via mensagem RabbitMQ (`input_path` e `output_path`), sem que os serviços precisem conhecer um ao outro diretamente.

---

## Nota sobre o Fluxo de Notificação

A arquitetura atual usa **HTTP direto** do Video Manager para o Communication Service como canal ativo de notificação. O Communication Service contém no código uma implementação de consumidor RabbitMQ (`communication_queue`) que está presente como legado, mas **não é ativada no fluxo atual do sistema**. O canal ativo é exclusivamente a chamada `POST /communications/test`.
