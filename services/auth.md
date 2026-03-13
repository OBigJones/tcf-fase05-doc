# Resumo — Auth Service

## Identificação

| Atributo | Valor |
|---|---|
| Nome | Auth Service |
| Linguagem / Framework | .NET 8 / ASP.NET Core |
| Arquitetura | Clean Architecture (Domain → Application ← Infrastructure → Api) |
| Porta HTTP | `8084` |
| Porta HTTPS | `9094` |
| Imagem Docker | `ddutra9/fase-05-auth:latest` |

## Responsabilidade

Componente central de segurança do ecossistema. Gerencia credenciais de usuários, autentica logins, emite tokens JWT e valida tokens apresentados pelos demais serviços.

## Funcionalidades Principais

| Funcionalidade | Descrição |
|---|---|
| Registro de credenciais | Recebe `UserId`, `email` e `password`; valida existência do usuário no User Service; faz hash SHA-256 da senha e persiste a credencial. |
| Autenticação | Valida e-mail e senha contra o hash armazenado; impõe bloqueio após 5 tentativas consecutivas com falha (bloqueio de 5 minutos). |
| Emissão de JWT | Gera token JWT assinado com HMAC-SHA256, contendo claims `sub`, `role`, `email`, `jti` e `iat`. TTL configurável (padrão: 60 min em produção). |
| Validação de JWT | Analisa e valida qualquer JWT emitido pelo serviço; retorna claims embutidas ou erro estruturado. |
| Bloqueio de conta | Rastreia tentativas com falha por credencial; bloqueia temporariamente após exceder o limite. |

## Integrações

| Direção | Serviço | Tipo | Detalhes |
|---|---|---|---|
| Saída | User Service | HTTP GET | `/users/{email}` — verifica se usuário existe e está ativo |
| Entrada | BFF Service | HTTP | Único consumidor dos quatro endpoints de autenticação |

## Persistência

| Componente | Tecnologia | Banco de dados |
|---|---|---|
| Credenciais de usuário | PostgreSQL via Entity Framework Core (Npgsql) | `auth_db` |

## Sem Mensageria

Este serviço não publica nem consome mensagens do RabbitMQ. Toda comunicação é HTTP síncrono.

## Modelo de Domínio

| Conceito | Tipo | Descrição |
|---|---|---|
| `UserCredential` | Entidade (Aggregate Root) | Credenciais armazenadas com lógica de bloqueio de conta |
| `JwtToken` | Value Object | Token emitido com data de expiração |
| `TokenClaims` | Value Object | Claims a serem incorporadas no JWT |
| `Email` | Value Object | E-mail validado com invariantes |
| `PasswordHash` | Value Object | Hash de senha validado |
| `TokenValidationResult` | Value Object | Resultado estruturado da validação de token |
