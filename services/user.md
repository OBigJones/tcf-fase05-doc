# Resumo — User Service

## Identificação

| Atributo | Valor |
|---|---|
| Nome | User Service |
| Linguagem / Framework | .NET 8 / ASP.NET Core |
| Arquitetura | Clean Architecture (Domain → Application ← Infrastructure → Api) |
| Porta | `8082` |
| Imagem Docker | `ddutra9/fase-05-user:latest` |

## Responsabilidade

Gerencia o cadastro e a identificação de usuários. É o ponto de verdade sobre a existência e os dados de uma conta no ecossistema.

## Funcionalidades Principais

| Funcionalidade | Descrição |
|---|---|
| Cadastro de usuários | Valida CPF (algoritmo oficial), valida formato de e-mail, verifica duplicidade por CPF e por e-mail, e persiste a conta no banco de dados. |
| Identificação de usuários | Busca de conta por CPF ou e-mail; consulta o cache Redis antes de acessar o banco. |
| Cache distribuído | Resultados de identificação são armazenados no Redis com TTL de 1 hora, reduzindo carga no banco em leituras repetidas. |
| Health Check | Endpoint `/health` para monitoramento pelo Kubernetes. |

## Integrações

| Direção | Serviço | Tipo | Detalhes |
|---|---|---|---|
| Entrada (cadastro) | BFF Service | HTTP POST `/users` | Criação de perfil durante o processo de signup |
| Entrada (consulta) | Auth Service | HTTP GET `/users/{email}` | Verificação de existência e status do usuário antes do login e registro de credenciais |

## Persistência

| Componente | Tecnologia | Banco de dados |
|---|---|---|
| Dados de usuário | PostgreSQL via Entity Framework Core | `user_db` |
| Cache de usuários | Redis (TTL 1 hora) | Chave: `user:{cpfOrEmail}` |

## Sem Mensageria

Este serviço não utiliza mensageria assíncrona. Toda comunicação é síncrona via HTTP REST.

## Endpoints

| Método | Endpoint | Descrição |
|---|---|---|
| `POST` | `/users` | Cria nova conta de usuário |
| `GET` | `/users/{cpfOrEmail}` | Busca usuário por CPF ou e-mail |
| `GET` | `/health` | Health check |

## Modelo de Domínio

| Conceito | Tipo | Descrição |
|---|---|---|
| `UserEntity` | Entidade | Dados do usuário: Id, Nome, CPF, E-mail |
| `CPFUtils` | Value Object | Validação completa do CPF pelo algoritmo oficial |
| `EmailUtils` | Value Object | Validação de formato de e-mail |
