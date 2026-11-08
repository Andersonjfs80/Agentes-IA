# Padrão de Arquitetura .NET - Agente IA

Este documento define os padrões obrigatórios para o ecossistema de microserviços.

## 1. Organização de Camadas

A solução segue os princípios de Clean Architecture, organizada em:

- **Api**: Controllers e Program.cs.
- **Domain**: Models, Commands, Handlers, Interfaces e Types.
- **Infrastructure**: DbContext, Repositories, Mappings e IoC.
- **Tests**: Testes Unitários, Funcionais e Integrados.

## 2. Nomenclatura Estrita

- **Abreviação Zero**: `Autenticacao` em vez de `Auth`.
- **Sufixos Proibidos**: Não usar abreviações em nomes de projetos ou pastas.

---

(Seções de 3 a 16 omitidas para brevidade na reconstrução parcial)
---

### 17.5. Boas Práticas (Cache)

1. **Chaves de Cache**: Use o padrão `{ControllerName}/{HandlerName}/{SessionId}` (ex: `Produto/ProdutoListagemQuery/sessionId`).
2. **Subtipos**: Caso houver algo em comum, use um subtipo ou nome da função específica: `{ControllerName}/{HandlerName}/{SessaoId}/{Subtipo}`.
3. **SessionId**: Sempre inclua o SessionId na chave para isolar caches entre usuários/sessões.
4. **TTL**: Defina TTL apropriado baseado na volatilidade dos dados (5-15 minutos).

## 18. Comunicação HTTP e Segurança SSL (Core_Http)

A biblioteca `Core_Http` centraliza a comunicação entre serviços, implementando resiliência e políticas de segurança.

### 18.1. Configuração

```json
{
  "Core_Http_Settings": {
    "RetryCount": 3,
    "BaseDelaySeconds": 2,
    "FailureThreshold": 5,
    "DurationOfBreakSeconds": 30,
    "DesativarSegurancaSsl": false
  }
}
```

### 18.2. Regras de Segurança SSL (Confiança Nativa)

1. **Variável de Risco**: A validação de certificados SSL só pode ser ignorada através da chave `DesativarSegurancaSsl`.
2. **Segurança 100% (Default)**: Por padrão, a segurança deve estar sempre **ATIVA** (`false`).
3. **Confiança no SO**: Em ambientes com certificados autoassinados (Docker), a solução obrigatória é instalar o certificado `.crt` no repositório de confiança do Sistema Operacional (Root Store) via Dockerfile.

## 19. Diagnóstico, Configurações e Lições Aprendidas (Docker e Segurança)

### 19.1. Sincronização Estrita de Configurações (Appsettings)
- **Problema**: Omissionar blocos transversais (ex: `Security`) em microserviços secundários ou esquecer de atualizá-los em arquivos de ambiente (ex: `appsettings.Homologation.json`).
- **Solução (Regra de Ouro)**: Sempre que alterar configurações estruturais como chaves de criptografia ou Kafka, valide e altere **todos** os arquivos `appsettings.json` de **todos** os backends e gateways. Se o Docker usar `ASPNETCORE_ENVIRONMENT=Homologation`, é obrigatório validar/atualizar os overrides presentes nos arquivos `appsettings.Homologation.json`, pois eles mesclam as configurações na inicialização.

### 19.2. Assimetria de Criptografia (Field Encryption)
- **Problema**: Erros fantasma de `401 Unauthorized` mesmo com credenciais corretas.
- **Solução**: Garantir que as variáveis do frontend (`environment.ts`) estejam rigorosamente espelhadas no backend. A chave string `fixedEncryptionKey` (Frontend) não pode divergir do `FixedEncryptionKey` (Backend), e se `enableFieldEncryption` for `true` na interface, `EnableFieldEncryption` DEVE ser `true` no backend para evitar que a API tente descriptografar texto puro e processe hashes corrompidos.

### 19.3. Bypass Crítico em Middlewares Globais
- **Logs (KafkaLoggingMiddleware)**: Middlewares de log automático devem implementar cláusulas explícitas para ignorar requisições feitas à própria rota de logs (ex: `/logs`), evitando recursividade e **loop infinito** que inflaciona o banco de dados/Kibana.
- **Documentação (BodyEncryptionMiddleware)**: Middlewares que interceptam payload e manipulam o formato JSON devem obrigatoriamente ignorar requisições em `/swagger` ou `/v1/api-docs`.

### 19.4. Conflito de Portas Kafka
- **Problema**: Erro `port is already allocated` ao mapear `29092:29092` em múltiplos microserviços.
- **Solução**: Apenas o container `kafka` deve realizar o mapeamento de porta externa. Os microserviços devem se conectar via rede interna usando o nome do serviço (`kafka:29092`).

### 19.5. Comunicação entre Containers (Localhost vs Service Name)
- **Problema**: Usar `localhost` em URLs de backend dentro do container causa erro de conexão.
- **Solução**: Sempre use o nome do container/serviço definido no `docker-compose.yml` (ex: `http://app-backend-autenticacao:7001`).

### 19.6. Ambiente Realista (Environment)
- **Diretriz**: Use `ASPNETCORE_ENVIRONMENT=Homologation` (ou similar) no Docker para simular o comportamento de produção. O ambiente `Development` deve ser reservado para rodar as aplicações localmente via IDE.

---
---

## 20. Arquitetura de Gateways (Listas de Compras)

O projeto usa um modelo **híbrido compacto** — microserviços com gateways centralizadores por domínio.

| Gateway | Porta | Responsabilidade |
|:--------|:------|:-----------------|
| `app-api-autenticacao` | 5006 | Login, token, sessão. Aponta para `app-backend-autenticacao`. |
| `app-api-cadastro` | 5022 | **Gateway central**: produtos, categorias, preços e todos os recursos de cadastro. Aponta para `app-backend-produto`. |

### Regra para novos recursos

- **Qualquer recurso de negócio novo** (ex: fornecedor, etiqueta, imagem) → entra em `app-api-cadastro`.
- **Autenticação/segurança** → entra em `app-api-autenticacao`.

### Exceção documentada

- `app-backend-lista-compras` ainda **não tem gateway dedicado**. O nginx do `app-mobile-compras` roteia `/app-api-lista-compras` diretamente para o backend enquanto o módulo amadurece. Futuramente poderá ter seu próprio gateway ou ser absorvido por `app-api-cadastro`.

## 21. Segurança End-to-End e Criptografia (JOSE/JWE)

Para garantir privacidade total em trânsito, o ecossistema utiliza o padrão **JOSE (JSON Web Encryption)**.

1.  **Middleware Crítico**: O `BodyEncryptionMiddleware` deve ser obrigatoriamente o **PRIMEIRO** middleware registrado após o `builder.Build()`.
    *   *Por que?* Para garantir que o body seja descriptografado ANTES que o ASP.NET valide o Content-Type ou tente fazer o Model Binding.
2.  **Formato de Tráfego**: 
    *   O corpo da requisição deve ser enviado como `text/plain` ou `application/json` (envelopado como string JSON). 
    *   O Middleware deve transformar o body em `application/json` após a descriptografia.
3.  **Headers Ofuscados**: Nunca usar nomes como `X-Body-Encrypted` ou `JOSE`. Utilizar apenas o header discreto **`X-Sec-Key: 1`** para evitar que terceiros identifiquem a tecnologia de segurança.

## 22. Logs e Observabilidade Inteligente
1.  **Nomenclatura PascalCase**: Usar estritamente PascalCase nos campos do `LogCustomModel` via atributos `[JsonPropertyName]`.
2.  **Detalhamento de URL**: Logs devem decompor a URL em campos específicos para indexação no Elasticsearch: `Scheme`, `Host`, `Port`, `Path`, `Query`, `Fragment`.
3.  **Fluxo em Duas Fases**: Toda requisição HTTP deve gerar dois logs distintos:
    *   **Tipo `request`**: Enviado *antes* do processamento do controller.
O projeto usa um modelo **híbrido compacto** — microserviços com gateways centralizadores por domínio.

| Gateway | Porta | Responsabilidade |
|:--------|:------|:-----------------|
| `app-api-autenticacao` | 5006 | Login, token, sessão. Aponta para `app-backend-autenticacao`. |
| `app-api-cadastro` | 5022 | **Gateway central**: produtos, categorias, preços e todos os recursos de cadastro. Aponta para `app-backend-produto`. |

### Regra para novos recursos

- **Qualquer recurso de negócio novo** (ex: fornecedor, etiqueta, imagem) → entra em `app-api-cadastro`.
- **Autenticação/segurança** → entra em `app-api-autenticacao`.

### Exceção documentada

- `app-backend-lista-compras` ainda **não tem gateway dedicado**. O nginx do `app-mobile-compras` roteia `/app-api-lista-compras` diretamente para o backend enquanto o módulo amadurece. Futuramente poderá ter seu próprio gateway ou ser absorvido por `app-api-cadastro`.

## 21. Segurança End-to-End e Criptografia (JOSE/JWE)

Para garantir privacidade total em trânsito, o ecossistema utiliza o padrão **JOSE (JSON Web Encryption)**.

1.  **Middleware Crítico**: O `BodyEncryptionMiddleware` deve ser obrigatoriamente o **PRIMEIRO** middleware registrado após o `builder.Build()`.
    *   *Por que?* Para garantir que o body seja descriptografado ANTES que o ASP.NET valide o Content-Type ou tente fazer o Model Binding.
2.  **Formato de Tráfego**: 
    *   O corpo da requisição deve ser enviado como `text/plain` ou `application/json` (envelopado como string JSON). 
    *   O Middleware deve transformar o body em `application/json` após a descriptografia.
3.  **Headers Ofuscados**: Nunca usar nomes como `X-Body-Encrypted` ou `JOSE`. Utilizar apenas o header discreto **`X-Sec-Key: 1`** para evitar que terceiros identifiquem a tecnologia de segurança.

## 22. Logs e Observabilidade Inteligente
1.  **Nomenclatura PascalCase**: Usar estritamente PascalCase nos campos do `LogCustomModel` via atributos `[JsonPropertyName]`.
2.  **Detalhamento de URL**: Logs devem decompor a URL em campos específicos para indexação no Elasticsearch: `Scheme`, `Host`, `Port`, `Path`, `Query`, `Fragment`.
3.  **Fluxo em Duas Fases**: Toda requisição HTTP deve gerar dois logs distintos:
    *   **Tipo `request`**: Enviado *antes* do processamento do controller.
    *   **Tipo `response`** ou **`error`**: Enviado *após* a conclusão, contendo o resultado e tempo de execução.
4.  **Unificação de Logs (Backend)**: O `KafkaLoggingMiddleware` deve utilizar a instância de `ILogCustom` do escopo para preencher os dados, garantindo que logs manuais inseridos pelo desenvolvedor sejam enviados junto com o log automático da requisição.
5.  **Rastreabilidade**: O `X-Trace-Id` deve ser gerado no primeiro contato e propagado por toda a cadeia de microserviços.
6.  **Sanitização Preventiva**: Campos sensíveis (ex: `Senha`, `Cpf`, `Token`) devem ser mascarados com `***` antes da serialização.
7.  **Log Batching (Frontend)**: Logs do frontend devem ser enviados em lotes para otimizar o tráfego.

---
## 23. Gestão de Documentação do Agente (Evitar Artefatos Efêmeros)

Sempre que o Agente IA iniciar um projeto ou sessão e precisar gerar arquivos de acompanhamento ou planejamento (como `task.md`, `implementation_plan.md` e `walkthrough.md`), ele **NÃO** deve deixá-los escondidos nas pastas temporárias da IDE (ex: `.gemini\antigravity-ide\brain\...`).

**Regra Absoluta:**
O Agente deve criar uma pasta física na raiz do projeto chamada `Docs_Agente/` e centralizar **todos** os arquivos de status e documentação lá dentro.
Isso garante que, mesmo abrindo uma nova sessão de chat, o contexto do projeto se mantenha salvo junto ao código-fonte.

---
O foco deste agente é: ajudar o usuário a manter a arquitetura consistente, limpa e segura.
