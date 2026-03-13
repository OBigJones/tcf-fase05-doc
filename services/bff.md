# Resumo — BFF Service

## Identificação

| Atributo | Valor |
|---|---|
| Nome | BFF Service (Backend for Frontend) |
| Linguagem / Framework | Java 21 / Spring Boot 4.0.2 |
| Arquitetura | Clean Architecture + Hexagonal (Ports & Adapters) |
| Porta | `8081` |
| Imagem base | `eclipse-temurin:21-jre-alpine` |
| Imagem Docker | `ddutra9/fase-05-bff:latest` |

## Responsabilidade

Único ponto de entrada HTTP para clientes externos (web/mobile). Atua como gateway que orquestra chamadas para os demais serviços da plataforma, centralizando autenticação, roteamento e controle de acesso.

## Funcionalidades Principais

| Funcionalidade | Descrição |
|---|---|
| Gateway de entrada | Único ponto de acesso HTTP para clientes externos. |
| Cadastro de usuário | Orquestra a criação do perfil (User Service), registro de credenciais e login (Auth Service), retornando o JWT ao cliente. |
| Login | Delega autenticação ao Auth Service e retorna o JWT. |
| Validação de token | Intercepta todas as requisições autenticadas via `TokenAuthenticationFilter` e valida o Bearer token no Auth Service. |
| Upload de vídeos | Recebe arquivos multipart e os encaminha ao Video Manager Service com o identificador do usuário. |
| Listagem de vídeos | Recupera a lista paginada de vídeos do usuário autenticado do Video Manager Service. |
| Download de resultados | Faz streaming do arquivo ZIP de resultado ao cliente passando pelo Video Manager Service. |

## Integrações

| Serviço Destino | Variável de Ambiente | Padrão Local | Operações |
|---|---|---|---|
| Auth Service | `AUTH_URL` | `http://localhost:8084/auth` | `POST /credentials`, `POST /login`, `POST /validate` |
| User Service | `USER_URL` | `http://localhost:8082/users` | `POST /users` |
| Video Manager Service | `PROCESSOR_VIDEO_URL` | `http://localhost:8085` | `POST /videos`, `GET /videos`, `GET /videos/{id}/result` |

## Sem Banco de Dados / Sem Mensageria

O BFF **não possui banco de dados próprio** e **não publica nem consome filas de mensageria diretamente**. Toda persistência e processamento assíncrono são delegados aos serviços especializados.

## Segurança

- Autenticação **stateless** via Bearer Token JWT.
- `TokenAuthenticationFilter` intercepta requisições e valida o token no Auth Service.
- Rotas públicas: `/api/auth/signup`, `/api/auth/login`, `/health`, `/error`.
- CSRF desabilitado; `SessionCreationPolicy.STATELESS`.

## Endpoints

### Autenticação (público)

| Método | Endpoint | Descrição |
|---|---|---|
| `POST` | `/api/auth/signup` | Cadastra novo usuário e retorna JWT |
| `POST` | `/api/auth/login` | Autentica usuário e retorna JWT |
| `GET` | `/health` | Health check |

### Vídeos (requer `Authorization: Bearer <token>`)

| Método | Endpoint | Descrição |
|---|---|---|
| `POST` | `/api/bridge/process` | Upload de vídeo(s) para processamento |
| `GET` | `/api/bridge/list` | Lista vídeos do usuário com paginação |
| `GET` | `/api/bridge/{id}/download` | Download do ZIP com frames gerados |
