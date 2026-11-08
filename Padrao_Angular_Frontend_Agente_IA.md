# Padrão de Desenvolvimento Frontend - Agente IA

Este documento define a estrutura, tecnologias e boas práticas para o desenvolvimento do frontend do ecossistema **Listas de Compras**, agora baseado em **Micro-Frontends**.

---

## ⚠️ REGRAS OBRIGATÓRIAS (evitar erros de build e imports)

**SEMPRE seguir. Não inventar nomes diferentes.**

| Regra | Correto | Errado |
| :--- | :--- | :--- |
| **DISCIPLINA OPERACIONAL** | **NÃO** inventar infraestruturas ou handlers não solicitados ("lixo"). | Criar componentes extras sem autorização. |
| **ABREVIAÇÃO ZERO** | **Nomes completos**: `Autenticacao`, `Produto`, `Notificacao`. | `Auth`, `Prod`, `Notif`. |
| Pasta de módulos | **modulos** | ~~features~~, ~~funcionalidades~~ |
| Imports de módulos | `./app-routing.module`, `./app.module` | ~~app-routing-module~~, ~~app-module~~ |
| main.ts | `import { AppModule } from './app/app.module';` | ~~./app/app-module~~ |
| Imports de módulos de negócio | `./modulos/listas/listas` etc. | ~~./features/...~~ |
| Integração Mobile | **iframe-wrapper** carregando módulo via porta Docker (ex: 4202) | Importar módulo diretamente no Mobile |
| Componentes standalone | **Não** colocar em `NgModule.declarations` (exceto wrappers específicos); usar **só nas rotas** (AppRoutingModule). Em `declarations` fica só o componente raiz (AppComponent). | Declarar standalone em declarations |
| Nome das classes nos specs | `ListasComponent`, `OfertasComponent` | ~~Listas~~, ~~Ofertas~~ |
| Teste de standalone | `imports: [ListasComponent]` no TestBed | ~~declarations: [Listas]~~ |

**Projeto Angular completo (todo Pod):** deve ter obrigatoriamente:

- `src/main.ts` (bootstrap do AppModule)
- `src/index.html` (shell com `<app-root>`)
- `src/styles.scss` (referenciado no angular.json)
- Pasta `public/` se o angular.json tiver `assets` com `"input": "public"`

**TypeScript:** Em projetos com RxJS 7 e TypeScript 5.9+, usar `"noImplicitOverride": false` no `tsconfig.json` (evita erro de compilação em node_modules/RxJS).

**package.json (build Docker):** Usar `"packageManager": "pnpm@10.x"` no package.json. No Dockerfile, instalar com `npm install -g pnpm` e usar `pnpm install --frozen-lockfile` e `pnpm run build`. **Não** usar npm como packageManager.

**Path alias @app/logs (ex.: app-modulo-home):** Se o Pod importa `@app/logs` (path `../../Library/app-library-logs`), o build Docker **precisa** de context na **raiz do repositório**. No docker-compose: `context: .` e `dockerfile: Frontend/Modules/app-modulo-home/Dockerfile`. O Dockerfile deve copiar `Frontend/Library` e depois trabalhar em `Frontend/Modules/app-modulo-home`.

---

## 🚀 Tecnologias e Frameworks

1. **Framework Principal**: Angular 18+ (Standalone Components onde possível, NgModules para Pods).
2. **Híbrido/Mobile**: Ionic Framework 7+ + Capacitor (para acesso nativo iOS/Android no App Principal).
3. **Estilização**: Vanilla CSS / SCSS com Angular Material (Web) e Ionic (Mobile).
4. **Comunicação API**: HttpClient (Angular) conectado aos **Gateways**.

## 0. Estrutura do Ecossistema (Micro-Frontends)

O frontend foi refatorado para reduzir o acoplamento. Atualmente temos **3 Pods (Módulos Independentes)** principais:

| Módulo | Caminho | Porta Docker | Responsabilidade |
| :--- | :--- | :--- | :--- |
| **Auth Pod** | `Frontend/Modules/app-modulo-autenticacao` | **4201** | Login, Cadastro, Recuperação de Senha. Redireciona para Home. |
| **Home Pod** | `Frontend/Modules/app-modulo-home` | **4202** | Dashboard, Landing Page, Destaques. Redireciona para o App Principal. |
| **Main App** | `Frontend/Mobile/Angular/app-frontend-listas-compras` | **4200** | Core do negócio: Listas, Carrinho, Produtos. |

### 🔄 Fluxo de Navegação & Gateways

1. **Acesso Inicial**: Usuário acessa `http://localhost:4201` (Login).
2. **Autenticação**: O Pod de Auth consome o **Gateway de Auth (:5005)**.
3. **Redirecionamento**: Após login, o usuário é enviado para `http://localhost:4202` (Home).
4. **Uso**: O Pod Home consome o **Gateway de Produtos (:5000)** para exibir destaques.
5. **Ação**: Ao clicar em "Minhas Listas", o usuário vai para `http://localhost:4200`.

---

## 📂 Estrutura de Pastas (Pattern)

**MANDATÓRIO**: O projeto segue o padrão **Inglês Técnico** para todas as pastas e arquivos, exceto "modulos" que é convencionado.

- **Pastas Estruturais**: `src`, `app`, `assets`, `environments`.
- **Pastas de Organização**:
  - `modulos` (Substitui `features` e `funcionalidades`).
  - `services` (Substitui `servicos`).
  - `models` (Substitui `modelos`).
  - `components` (Substitui `componentes`).

### Exemplo de Árvore de Arquivos (Pod Padrão)

```text
src/
└── app/
    ├── app.module.ts           (Módulo Raiz com providers globais)
    ├── app-routing.module.ts   (Roteamento Interno do Pod)
    ├── app.component.ts        (Componente Raiz com <router-outlet>)
    ├── services/               (Serviços HTTP / Singleton - ex: auth.service.ts)
    └── modulos/                (Módulos de Negócio)
        ├── @home/              (Ex: Wrapper para integração mobile)
        │   ├── home.module.ts
        │   └── home-wrapper.component.ts
        └── auth/               (Ex: Auth Module)
            ├── login/
            └── register/
```

## 🔥 Como Criar um Novo Pod (Passo a Passo)

Para criar um novo módulo independente (ex: `app-modulo-perfil`), siga este checklist obrigatório:

1. **Criação do Projeto**:
    - Não crie apenas pastas! É necessário inicializar uma aplicação Angular completa.
    - Use o Angular CLI ou clone a estrutura de um módulo existente (`app-modulo-home`).
    - **Arquivos Obrigatórios**: `package.json`, `angular.json`, `tsconfig.json`.

2. **Configuração (angular.json)**:
    - Renomeie o projeto dentro do `angular.json` para o nome do novo pod (ex: `"projects": { "app-modulo-perfil": { ... } }`).
    - Ajuste o `outputPath` para `dist/app-modulo-perfil`.

3. **Dockerização**:
    - Crie um `Dockerfile` na raiz do módulo (copie de outro módulo e ajuste o caminho do `COPY --from=build-step`).
    - Se o Pod usar path alias para `Frontend/Library` (ex.: `@app/logs`), no docker-compose use `context: .` (raiz do repo) e `dockerfile: Frontend/Modules/nome-do-pod/Dockerfile`; no Dockerfile copie `Frontend/Library` e depois o módulo.
    - Não use `packageManager` no package.json de Pods que buildam em Node 20.
    - Adicione o serviço no `docker-compose.yml` definindo uma nova porta (ex: `4203`).

4. **Integração**:
    - Adicione o redirecionamento/link no App Principal ou Home para apontar para `http://localhost:RPOTA`.

---

## 🛡️ Headers Mandatórios e Segurança (Padrão 2026)

1. **Interceptação Global**: Toda requisição `HttpClient` deve ser interceptada pela lib compartilhada `@app/logs`.
2. **Segurança End-to-End (JOSE/JWE)**:
   - Todo tráfego de body (POST/PUT/PATCH), incluindo logs, deve ser criptografado via `EncryptionInterceptor` usando o padrão **JOSE (JWE)** envelopado em JSON `{ "data": "..." }`.
   - O Interceptor é bi-direcional.
   - **Ofuscação**: Utilizar apenas o header discreto **`X-Sec-Key: 1`** para sinalizar tráfego protegido.
3. **Criptografia de Campos (Field Encryption)**:
   - **Regra de Ouro**: O `environment.ts` do Angular **DEVE ESPELHAR EXATAMENTE** as configurações do `appsettings.json` do backend.
   - A flag `enableFieldEncryption` (Frontend) precisa ser `true` se o Backend usar `EnableFieldEncryption: true`. Jamais enviar senhas em texto puro para um backend que espera criptografia.
   - A chave string `fixedEncryptionKey` deve ser idêntica à chave `FixedEncryptionKey` do C#. Não usar placeholders Base64 (`YmFzZ...`) no frontend se o C# estiver esperando chaves string reais (convertidas internamente).

## ⚙️ Telemetria e Centralização

1. **Log Batching**:
   - Utilizar o `LogService` com sistema de **batching** (5 entradas ou 3 segundos).
   - **Criptografia**: Diferente de versões anteriores, os logs agora **são criptografados** no body para garantir privacidade total no Network Tab.

2. **Centralização de Autenticação**:
   - O login **não deve** existir no módulo Home ou no App Principal.
   - Todo fluxo de autenticação deve ser redirecionado para o **Auth Pod** (Porta 4204/4201).
   - O `authGuard` dos outros módulos deve direcionar para a URL absoluta do Auth Pod se o token não for válido.

## 🖥️ Checklist de Segurança e SSR (Server-Side Rendering)

1.  **Serviço de Arquivos Estáticos**: O arquivo `server.ts` **DEVE** conter obrigatoriamente o middleware `express.static` configurado para a pasta `browserDistFolder` **ANTES** da rota curinga (`*`).
    *   **Diagnóstico de Falha**: Se componentes Angular não responderem a cliques, verificar na aba Network se arquivos `.js` ou `.css` estão retornando status 200 com conteúdo HTML. Isso indica que a rota curinga está "engolindo" os arquivos estáticos.
2.  **Segurança de Formulários**:
    *   Sempre definir `method="POST"`.
    *   Sempre definir `action="javascript:void(0)"` para evitar submissão nativa indesejada.
3.  **Integridade do Layout**:
    *   Correções de bugs funcionais (JavaScript/TypeScript) não devem acarretar mudanças de CSS, margens ou cores, a menos que o bug seja visual.

3. **Centralização de Autenticação**:
   - O login **não deve** existir no módulo Home ou no App Principal.
   - Todo fluxo de autenticação deve ser redirecionado para o **Auth Pod** (Porta 4204/4201).
   - O `authGuard` dos outros módulos deve direcionar para a URL absoluta do Auth Pod se o token não for válido.

---

## 🛠️ Boas Práticas

1. **Integração com Backend**: Nunca mockar dados em serviços finais. Sempre injetar `HttpClient` e usar as URLs dos **Gateways** (`:5000`, `:5005`, etc).
2. **Clean Code no Template**: Manter a lógica no arquivo `.ts` e o template `.html` o mais simples possível. Usar design premium (Glassmorphism, HSL Gradients) para interfaces de impacto.
3. **Tipagem Estrita (Proibido uso de `any`)**: O uso do modo `strict` no TypeScript é obrigatório. **NUNCA** use `any`. Todas as respostas de API, payloads e variáveis devem ter interfaces ou types explícitos que reflitam exatamente os Data Contracts do backend C#.
4. **Micro-Frontends**: Cada Pod deve ser capaz de rodar isoladamente (ter seu próprio `Dockerfile`).
5. **Delegação de Regras de Negócio ao Backend**: O Angular é **estritamente uma camada de apresentação**. É PROIBIDO implementar regras de negócio, cálculos financeiros ou "mocks visuais dinâmicos" (ex: inventar preços). O frontend deve exibir apenas o que o Backend/BFF retorna. Se a propriedade não existe na API, a tela adapta o layout, mas não cria o dado.

## 📱 Integração Mobile (Iframe Wrapper Pattern)

...

Para integrar os Micro-Frontends (Pods) no App Principal (Mobile), utilizamos o padrão **Iframe Wrapper**:

1. **No App Mobile**:
    - Crie um módulo wrapper em `src/app/modulos/@nome-do-modulo/`.
    - O wrapper deve conter um componente com um `<iframe>` apontando para a URL do container Docker do módulo (ex: `http://localhost:4202/home`).
    - Use `standalone: false` no componente wrapper se ele fizer parte de um `NgModule`.
    - Configure o `iframe` com CSS para ocupar a área correta (descontando toolbar/bottom-nav).

2. **No App-Routing Mobile**:
    - Use `loadChildren` apontando para o módulo wrapper local.
    - Passe a URL de destino via `data: { url: '...' }` na rota, se o componente for genérico, ou chume no componente específico.

## 📱 Mobile Workflow

1. **Live Reload**: `ionic serve` para desenvolvimento web rápido (apenas no App Principal).
2. **Capacitor**: `npx cap sync` para sincronizar mudanças com os projetos nativos iOS/Android.
