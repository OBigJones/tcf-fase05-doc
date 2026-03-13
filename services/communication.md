# Resumo — Communication Service

## Identificação

| Atributo | Valor |
|---|---|
| Nome | Communication Service |
| Linguagem / Framework | .NET 8 / ASP.NET Core |
| Arquitetura | Clean Architecture (Domain → Application ← Infrastructure → Api) |
| Porta HTTP | `8086` |
| Porta HTTPS | `9096` |
| Imagem Docker | `ddutra9/fase-05-communication:latest` |

## Responsabilidade

Responsável pelo envio de notificações por e-mail aos usuários após o processamento de vídeos. Opera como serviço de entrega de mensagens — sem estado persistente.

## Funcionalidades Principais

| Funcionalidade | Descrição |
|---|---|
| Recepção via HTTP | Recebe requisições `POST /communications/test` do Video Manager Service. |
| Seleção de template | Escolhe o template adequado com base no status do processamento (`Finished` → sucesso; qualquer outro → falha). |
| Geração do e-mail | Constrói o corpo do e-mail utilizando o template de domínio selecionado. |
| Envio por SMTP | Envia o e-mail via MailHog (SMTP `:1025` em desenvolvimento). |
| Confirmação | Retorna `{ sent: true/false, message: "Email sent" }` ao chamador. |

## Integrações

| Direção | Serviço | Tipo | Detalhes |
|---|---|---|---|
| Entrada | Video Manager Service | HTTP POST | `/communications/test` — fluxo ativo |
| Saída | MailHog | SMTP | `:1025` — envio de e-mail |

## Sem Banco de Dados

Este serviço não utiliza banco de dados. Todo estado é efêmero — a mensagem é recebida, o e-mail é enviado e a operação é concluída.

## Nota sobre Mensageria (Legado)

O código contém uma implementação completa de consumidor RabbitMQ para a fila `communication_queue` (`RabbitMqConsumer`, `RabbitMqConsumerHostedService`). Essa implementação está **presente no código mas não está ativa no fluxo atual do sistema**. O canal ativo é exclusivamente HTTP.

## Templates de E-mail

| Status recebido | Assunto do e-mail | Conteúdo |
|---|---|---|
| `Finished` | "Processamento concluído: seu vídeo está disponível" | Informa sucesso e disponibilidade para download |
| Qualquer outro | "Falha no processamento do seu vídeo" | Informa falha e solicita nova tentativa |

## Endpoint Ativo

| Método | Endpoint | Descrição |
|---|---|---|
| `POST` | `/communications/test` | Dispara envio de e-mail de notificação |
| `GET` | `/health` | Health check da aplicação |

**Payload esperado:**
```json
{
  "email": "usuario@exemplo.com",
  "fileName": "meu-video.mp4",
  "status": "Finished"
}
```
