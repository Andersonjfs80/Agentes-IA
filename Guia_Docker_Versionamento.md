# Guia Prático: Simulação de Versão com Docker e CI/CD

No mercado, a versão de um microsserviço raramente é alterada manualmente no `.csproj`. Ela é injetada pela esteira de CI/CD (Azure DevOps, GitLab, GitHub Actions) durante o deploy.

Aqui estão as 3 formas de simular e configurar isso usando Docker no nosso modelo de **Monorepo**:

## 0. Contexto Monorepo

No monorepo, cada `Dockerfile` deve estar na raiz do seu respectivo projeto, mas o `docker-compose.yml` mestre geralmente fica na raiz do Monorepo para orquestrar todos os serviços (Backend + Gateway + Frontend).

## 1. Simulação com `docker-compose.yml` (Local)

A forma mais prática de simular diferentes versões localmente é via bloco `environment`. O .NET lê automaticamente qualquer variável de ambiente que coincida com a chave da configuração.

```yaml
services:
  gateway-api:
    image: gateway-api:latest
    environment:
      - AppName=Gateway-Producao
      - AppVersion=2.5.0-beta      # <--- Nossa lógica capturará isso primeiro
      - BUILD_ID=10293             # <--- Fallback secundário
```

## 2. Injeção Dinâmica via `Dockerfile` (Estratégia de Build)

Se você quer que a versão fique "fixa" na imagem gerada pela esteira, use `ARG` no build e `ENV` no runtime.

```dockerfile
# Dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS base
WORKDIR /app

# Variável que será preenchida durante o 'docker build'
ARG VERSION_FROM_PIPELINE=1.0.0
ENV AppVersion=$VERSION_FROM_PIPELINE

COPY . .
ENTRYPOINT ["dotnet", "NomeProjeto.Gateway.dll"]
```

**Comando na Esteira:**

```bash
docker build --build-arg VERSION_FROM_PIPELINE="v3.1.2" -t gateway:v3.1.2 .
```

## 3. Prática de Mercado: Variáveis de Runtime

Em Kubernetes (K8s), é comum injetar a versão via `ConfigMap` ou diretamente no `deployment.yaml`.

O ponto chave é que o `configuration["AppVersion"]` no seu código C# prioriza o que vem do ambiente (Environment Variables) sobre o que está no comando.

**Por que sua lógica é robusta?**

```csharp
var resolvedVersion = configuration["AppVersion"] // 1. Variável de Ambiente / Docker
                   ?? configuration["BUILD_ID"]   // 2. Azure/Jenkins Automático
                   ?? configuration["GIT_TAG"];   // 3. GitLab Automático
```

Ao usar essa estrutura, você garante que:

- **Dev Local**: Pega do `appsettings.json` ou assembly.
- **Docker Local**: Pega do `docker-compose`.
- **Cloud/Esteira**: Pega automaticamente da variável injetada pelo agente de CI/CD.

---

## 🛠️ Estabilidade de Infraestrutura (Suporte)

Ao configurar serviços de infraestrutura (Kafka, Redis, SQL) no `docker-compose.yml`, evite o uso da tag `:latest`.

1. **Kafka & Zookeeper**:
   - O uso da tag `:latest` no Kafka (`confluentinc/cp-kafka`) pode introduzir imagens que exigem o protocolo KRaft por padrão (erro `KAFKA_PROCESS_ROLES not set`), o que quebra a integração simples com o Zookeeper.
   - **Recomendação**: Fixar na versão **7.5.0** para garantir compatibilidade com as configurações padrão de listeners e brokers.

2. **Redis & SQL Server**:
   - Manter versões estáveis para evitar mudanças bruscas em drivers de comunicação do Backend (.NET).

---
**Dica de Ouro:** Sempre que quiser testar, basta rodar seu projeto via terminal passando a variável:
`$env:AppVersion="Simulacao-1.0"; dotnet run`
