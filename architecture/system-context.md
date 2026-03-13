# Contexto do Sistema — FIAP Tech Challenge Fase 5

## Visão de Alto Nível

O sistema é uma **plataforma de processamento de vídeos** que permite a usuários autenticados realizarem upload de arquivos de vídeo, aguardarem o processamento assíncrono e receberem os resultados por e-mail e via download direto.

Do ponto de vista externo, o sistema se apresenta como um único endpoint HTTP acessível a clientes web e móveis. Internamente, é composto por um ecossistema de microsserviços independentes que colaboram de forma síncrona (HTTP REST) e assíncrona (RabbitMQ).

---

## Atores Externos

| Ator | Tipo | Descrição |
|---|---|---|
| **Usuário Final** | Pessoa | Acessa o sistema via navegador ou aplicativo móvel para se cadastrar, fazer login, enviar vídeos e baixar os resultados processados. |
| **MailHog / Servidor SMTP** | Sistema externo | Servidor de e-mail que recebe as notificações enviadas pelo Communication Service. Em ambiente de desenvolvimento, o MailHog captura os e-mails sem enviá-los de fato. |

---

## Fronteiras do Sistema

### O que o sistema faz

- Autenticar usuários e gerenciar credenciais com emissão de tokens JWT.
- Receber uploads de arquivos de vídeo (até 500 MB).
- Processar vídeos de forma assíncrona, extraindo frames (1 frame/segundo) e empacotando-os em um arquivo `.zip`.
- Notificar o usuário por e-mail sobre o resultado do processamento (sucesso ou falha).
- Disponibilizar o arquivo `.zip` com os frames para download.

### O que o sistema não faz

- Não realiza streaming de vídeo em tempo real.
- Não exporta vídeos em outros formatos além das imagens PNG extraídas.
- Não gerencia múltiplos tenants ou organizações.
- Não envia e-mails reais em ambiente de desenvolvimento (MailHog os captura localmente).

---

## Posição no Ecossistema

```
                    ┌─────────────────────────────────────────────┐
                    │          FIAP Tech Challenge — Fase 5        │
                    │                                             │
  Usuário Final ───►│  BFF Service  (ponto de entrada único)      │
  (Web / Mobile)    │      │                                      │
                    │      ├──► Auth Service                      │
                    │      ├──► User Service                      │
                    │      └──► Video Manager Service             │
                    │               │                             │
                    │        ┌──────┘                             │
                    │        ▼                                    │
                    │     RabbitMQ                                │
                    │        │                                    │
                    │        ▼                                    │
                    │     Movie Processor                         │
                    │        │                                    │
                    │        ▼                                    │
                    │  Communication Service ──► MailHog (SMTP)   │◄── Sistema externo
                    │                                             │   de e-mail
                    └─────────────────────────────────────────────┘
```

---

## Princípios Arquiteturais

Todos os serviços do ecossistema seguem os mesmos princípios de projeto:

### Clean Architecture

Cada serviço adota a **Clean Architecture** (também denominada Onion Architecture), com quatro camadas bem definidas:

| Camada | Responsabilidade |
|---|---|
| **Domain** | Entidades de negócio, value objects e regras de domínio. Sem dependências externas. |
| **Application** | Casos de uso, interfaces (portas) e orquestração. Depende apenas do Domain. |
| **Infrastructure** | Implementações concretas: banco de dados, filas, clientes HTTP, storage. |
| **Presentation / API** | Controladores REST, DTOs, mapeadores, documentação Swagger. |

A regra central é que as dependências sempre apontam de fora para dentro — a camada de Domain não conhece nenhuma outra.

### Separação de Responsabilidades

Cada microsserviço possui uma única responsabilidade claramente delimitada. Não há lógica de negócio duplicada entre serviços.

### Stateless

Os serviços não mantêm estado de sessão. A autenticação é realizada via JWT em cada requisição.

### Desacoplamento Assíncrono

O processamento de vídeos (operação lenta e intensiva em recursos) é desacoplado do fluxo de upload via RabbitMQ. O usuário não aguarda o processamento — ele é notificado por e-mail quando concluído.

---

## Ambientes

| Ambiente | Características |
|---|---|
| **Desenvolvimento** | Serviços rodando localmente ou via Docker Compose; MailHog usado como SMTP; RabbitMQ com painel de gerenciamento disponível em `localhost:15672`. |
| **Produção / Staging** | Todos os componentes implantados no cluster Kubernetes via Helm Chart; auto-escalonamento via HPA habilitado; senhas e segredos gerenciados via Kubernetes Secrets. |
