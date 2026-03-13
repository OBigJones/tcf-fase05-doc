# Resumo — Movie Processor Service

## Identificação

| Atributo | Valor |
|---|---|
| Nome | Movie Processor Service |
| Linguagem / Framework | Go |
| Arquitetura | Clean Architecture (domain → application ← infrastructure ← cmd/api) |
| Porta | `8083` |
| Imagem Docker | `ddutra9/fase-05-movie:latest` |
| Réplicas (K8s) | Mínimo: 2 / Máximo: 3 (HPA) |
| Ferramenta de processamento | FFmpeg |

## Responsabilidade

Worker assíncrono puro responsável pelo processamento de vídeos. Não expõe nenhuma API REST — opera exclusivamente consumindo mensagens do RabbitMQ, processando os arquivos e publicando os resultados de volta na fila de saída.

## Funcionalidades Principais

| Funcionalidade | Descrição |
|---|---|
| Consumo de tarefas | Lê mensagens da fila `video_queue` (prefetch=1, ACK manual). |
| Extração de frames | Executa FFmpeg: `ffmpeg -i <input> -vf fps=1 frame_%04d.png` — 1 frame por segundo. |
| Geração de ZIP | Compacta todos os frames PNG em `/app/outputs/frames_{id}.zip`. |
| Publicação de resultado | Envia mensagem na fila `video_processed` com o caminho do ZIP e o status (`SUCCESS` ou `ERROR`). |
| Retry automático | Reprocessa até 3 vezes em caso de falha (intervalo de 2 segundos entre tentativas, configurável via `MAX_RETRIES`). |
| Limpeza de temporários | Remove o diretório temporário de frames após criação do ZIP. |
| ACK manual | Confirma a mensagem original na fila somente após o processamento concluído. |

## Integrações

| Direção | Componente | Tipo | Detalhes |
|---|---|---|---|
| Entrada | RabbitMQ | AMQP (consumer) | Consome `video_queue`; prefetch=1 |
| Saída | RabbitMQ | AMQP (publisher) | Publica em `video_processed` |
| Entrada | Shared Storage | I/O arquivo (leitura) | Acessa o vídeo pelo `input_path` da mensagem |
| Saída | Shared Storage | I/O arquivo (escrita) | Grava o arquivo `.zip` em `/app/outputs/` |

**Não realiza chamadas HTTP para nenhum outro serviço.**

## Sem API REST / Sem Banco de Dados

Este serviço **não expõe endpoints HTTP** e **não utiliza banco de dados**. O único ponto de entrada externo é a fila `video_queue` do RabbitMQ.

## Mensagens

**Consumida (`video_queue`):**
```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "input_path": "/app/uploads/meu_video.mp4",
  "output_path": "",
  "status": "PENDING"
}
```

**Publicada (`video_processed`):**
```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "input_path": "/app/uploads/meu_video.mp4",
  "output_path": "/app/outputs/frames_550e8400-....zip",
  "status": "SUCCESS"
}
```

## Configuração do Consumidor RabbitMQ

| Parâmetro | Valor |
|---|---|
| Prefetch (QoS) | 1 (uma mensagem por vez por réplica) |
| Auto-ACK | Desabilitado (ACK manual) |
| Retries | Até 3 vezes com intervalo de 2 segundos |
