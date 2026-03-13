# Resumo — Video Manager Service

## Identificação

| Atributo | Valor |
|---|---|
| Nome | Video Manager Service |
| Linguagem / Framework | .NET 8 / ASP.NET Core |
| Arquitetura | Clean Architecture (Domain → Application ← Infrastructure → Api) |
| Porta | `8085` |
| Imagem base | `mcr.microsoft.com/dotnet/aspnet:8.0-alpine` |
| Imagem Docker | `ddutra9/fase-05-video-manager:latest` |

## Responsabilidade

Gerencia o ciclo de vida completo dos vídeos no ecossistema: recebe uploads, persiste metadados, aciona o processamento assíncrono, disponibiliza os resultados para download e notifica o usuário.

> **Importante:** Este serviço **não processa vídeos**. O processamento é responsabilidade exclusiva do Movie Processor Service.

## Funcionalidades Principais

| Funcionalidade | Descrição |
|---|---|
| Receber uploads | Aceita arquivos de vídeo via `multipart/form-data` (limite de 500 MB). |
| Persistência de metadados | Armazena no PostgreSQL: nome, tamanho, content-type, FPS solicitado e status do ciclo de vida. |
| Armazenamento de arquivo | Grava o arquivo bruto no storage compartilhado em `raw/<userId>/<videoId>/`. |
| Enfileiramento | Publica mensagem na fila `video_queue` do RabbitMQ após o upload. |
| Recebimento de resultado | Consome fila `video_processed` e atualiza o status do vídeo. |
| Notificação | Chama o Communication Service via HTTP após mudança de status. |
| Listagem | Retorna vídeos do usuário com paginação (`?skip=0&take=20`). |
| Download | Disponibiliza o arquivo `.zip` com os frames gerados pelo Movie Processor. |

## Ciclo de Vida do Vídeo

```
PendingUpload (0)  →  Processing (1)  →  Finished (2)
                                        ↘  ProcessingFailed (3)
```

## Integrações

| Direção | Serviço / Componente | Tipo | Detalhes |
|---|---|---|---|
| Entrada | BFF Service | HTTP | `POST /videos`, `GET /videos`, `GET /videos/{id}/result` |
| Saída | RabbitMQ | AMQP (publisher) | Publica em `video_queue` após upload |
| Entrada | RabbitMQ | AMQP (consumer) | Consome `video_processed` (BackgroundService) |
| Saída | Communication Service | HTTP POST | `/communications/test` — notifica resultado |
| Saída/Entrada | Shared Storage | I/O arquivo | Grava em `/uploads`; lê de `/outputs` para download |

## Persistência

| Componente | Tecnologia | Banco de dados |
|---|---|---|
| Metadados de vídeos | PostgreSQL via Entity Framework Core | `video_manager_db` |

## Endpoints

| Método | Rota | Descrição | Header |
|---|---|---|---|
| `POST` | `/videos` | Upload de vídeo (multipart) | `X-UserId: email` |
| `GET` | `/videos` | Lista vídeos com paginação | `X-UserId: email` |
| `GET` | `/videos/{id}/result` | Download do ZIP com frames | `X-UserId: email` |
| `GET` | `/health` | Health check | — |

## Mensagem Publicada (`video_queue`)

```json
{
  "id": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "input_path": "/video-storage/raw/user@email.com/3fa85f64/video.mp4",
  "output_path": "",
  "status": "queued"
}
```
