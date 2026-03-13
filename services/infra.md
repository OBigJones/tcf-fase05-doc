# Resumo — Repositório de Infraestrutura

## Identificação

| Atributo | Valor |
|---|---|
| Nome | Infraestrutura — FIAP Tech Challenge Fase 5 |
| Tecnologia | Helm Chart para Kubernetes |
| Finalidade | Provisionamento e orquestração declarativa de todo o ecossistema |

## Responsabilidade

Define como código toda a infraestrutura necessária para executar o ecossistema de microsserviços em um cluster Kubernetes. Todos os recursos são declarados em um único Helm Chart, permitindo implantação reproduzível e versionada.

## O que é Provisionado

| Recurso | Tecnologia | Detalhes |
|---|---|---|
| Microsserviços | 6 Deployments | BFF, User, Auth, Video Manager, Movie Processor, Communication |
| Banco de dados | PostgreSQL 15 | Único pod com 3 bancos via script de inicialização (`init.sql`) |
| Cache | Redis 7 | Usado pelo User Service |
| Broker de mensagens | RabbitMQ 3 (StatefulSet) | Filas `video_queue` e `video_processed` |
| SMTP de desenvolvimento | MailHog (latest) | Captura e-mails sem entregá-los |
| Storage compartilhado | PVC `fase-05-movie-storage` (5Gi, hostpath) | Diretórios `/uploads` e `/outputs` |
| Secrets | 3 Secrets Kubernetes | PostgreSQL, RabbitMQ e chave JWT |
| Auto-escalonamento | HPA (CPU ≥ 30% / Memória ≥ 70%) | Configurado para todos os serviços |
| Métricas | Kubernetes Metrics Server | Obrigatório para HPA funcionar |
| Ajuste de kernel | DaemonSet sysctl-inotify | `fs.inotify.max_user_instances=1024` e `max_user_watches=524288` |

## Bancos de Dados Criados Automaticamente

| Banco | Serviço Responsável |
|---|---|
| `auth_db` | Auth Service |
| `user_db` | User Service |
| `video_manager_db` | Video Manager Service |

## Filas RabbitMQ

| Fila | Produtor | Consumidor |
|---|---|---|
| `video_queue` | Video Manager Service | Movie Processor Service |
| `video_processed` | Movie Processor Service | Video Manager Service |

## Auto-escalonamento (HPA)

| Serviço | Réplicas mínimas | Réplicas máximas |
|---|---|---|
| BFF | 1 | 1 |
| User Service | 1 | 1 |
| Auth Service | 1 | 1 |
| Video Manager | 1 | 2 |
| Movie Processor | 2 | 3 |
| Communication Service | 1 | 1 |

## Kubernetes Secrets

| Secret | Conteúdo |
|---|---|
| `<release>-postgres-secret` | Senha do PostgreSQL |
| `<release>-rabbitmq-secret` | Senha do RabbitMQ |
| `<release>-jwt-secret` | Chave de assinatura JWT (Auth Service) |

## Estrutura do Helm Chart

```
fase-5/
├── Chart.yaml          # Metadados e versão
├── values.yaml         # Valores padrão de todos os serviços
├── metrics/
│   └── components.yaml # Metrics Server (obrigatório para HPA)
└── templates/
    ├── auth/           # Deployment, HPA, JWT Secret, Service
    ├── bff/            # Deployment, HPA, Service
    ├── communication/  # Deployment, HPA, Service
    ├── db/             # PostgreSQL, Redis e ConfigMap do init.sql
    ├── movie/          # Deployment, HPA, Service, PVC, DaemonSet sysctl
    ├── movie_manager/  # Deployment, HPA, Service
    ├── queue/          # RabbitMQ StatefulSet, Service, Secret
    ├── smtp_mail/      # MailHog Deployment, Service
    └── user/           # Deployment, HPA, Service
```

## Volume Compartilhado

O PVC `fase-05-movie-storage` (5Gi) é o ponto de integração de arquivos entre Video Manager e Movie Processor:

1. **Video Manager** grava o vídeo bruto em `uploads/` e publica mensagem em `video_queue`.
2. **Movie Processor** lê de `uploads/`, processa e grava o ZIP em `outputs/`.
3. **Video Manager** consome o resultado de `video_processed` e disponibiliza o ZIP via `outputs/` para download.

Ambos os Deployments usam `initContainer` baseado em `busybox` para garantir permissões corretas (`chmod -R 777`) antes da inicialização do container principal.
