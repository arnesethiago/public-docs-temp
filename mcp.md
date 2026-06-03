# MCP + Plati — arquitetura, integração e operações

Documento de referência para servidores **MCP** (Model Context Protocol) integrados à **Plati** (WhatsApp / conversas). Descreve o padrão arquitetural comum a todos os MCPs da Plati.

---

## Sumário

1. [Visão geral e stack](#1-visão-geral-e-stack)
2. [Fluxo de uma requisição (ponto a ponto)](#2-fluxo-de-uma-requisição-ponto-a-ponto)
3. [Por que `__mcp-meta` e não só headers](#3-por-que-__mcp-meta-e-não-só-headers)
4. [Durable Object, sessão e isolamento](#4-durable-object-sessão-e-isolamento)
5. [Autenticação em camadas](#5-autenticação-em-camadas)
6. [Metadados (`CustomMeta`) — referência](#6-metadados-custommeta--referência)
7. [Integração Plati (APIs e padrões)](#7-integração-plati-apis-e-padrões)
8. [Mensagens WhatsApp — tipos, limites e fallbacks](#8-mensagens-whatsapp--tipos-limites-e-fallbacks)
9. [Padrões de UX: loading, fire-and-forget e `waitUntil`](#9-padrões-de-ux-loading-fire-and-forget-e-waituntil)
10. [Multi-workspace e credenciais](#10-multi-workspace-e-credenciais)
11. [Bindings Cloudflare (KV, D1, etc.)](#11-bindings-cloudflare-kv-d1-etc)
12. [Logs — papel, conteúdo e o que nunca expor](#12-logs--papel-conteúdo-e-o-que-nunca-expor)
13. [Erros em duas camadas (`error-handler`)](#13-erros-em-duas-camadas-error-handler)
14. [Rate limiting](#14-rate-limiting)
15. [Registro de tools e contratos](#15-registro-de-tools-e-contratos)
16. [Exemplo estendido de tool](#16-exemplo-estendido-de-tool)
17. [Testes e MCP Inspector](#17-testes-e-mcp-inspector)
18. [Deploy e secrets](#18-deploy-e-secrets)
19. [Anti-padrões](#19-anti-padrões)
20. [Glossário](#20-glossário)
21. [Este documento é suficiente para uma IA “implementar tudo”?](#21-este-documento-é-suficiente-para-uma-ia-implementar-tudo)
22. [Escopo MVP e checklist para implementação (repo novo)](#22-escopo-mvp-e-checklist-para-implementação-repo-novo)

---

## 1. Visão geral e stack

### O que é, em uma frase

Um **Worker** Cloudflare expõe o transporte MCP (HTTP/SSE). Cada sessão MCP roda num **Durable Object** (`McpAgent` + `McpServer`), que registra **tools**. As tools chamam a **API Plati** (conversas, contatos) e/ou **APIs externas de negócio** (REST). O contexto do usuário (contato, conversa, workspace) chega por **headers HTTP** e é repassado de forma controlada ao código da tool.

### Stack típica

| Peça | Papel |
|------|--------|
| **Cloudflare Workers** | Entry `export default { fetch }`, roteamento, validação de API key, injeção de metadados. |
| **Durable Objects** | Classe `MyMCP extends McpAgent` — uma instância lógica por sessão MCP. |
| **`agents/mcp`** | Integração MCP ↔ DO (SSE, JSON-RPC, ciclo de vida). |
| **`@modelcontextprotocol/sdk`** | `McpServer`, `server.tool(...)`. |
| **TypeScript** | Tipos `Env`, `CustomMeta`, estendidos em `types.ts`. |
| **Wrangler** | `wrangler.jsonc` — nome do worker, bindings, `migrations` para D1. |
| **Zod** | Validação de argumentos das tools (quando usado). |

### Diagrama de alto nível

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         Cliente (Plati / orquestrador)                     │
│  Envia: Authorization / x-api-key, x-contact-id, x-conversation-id, ...   │
└─────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                         Worker.fetch(request, env, ctx)                    │
│  • Valida MCP_API_KEY (ou política Plati + headers)                        │
│  • extractCustomMeta(request) → objeto { contactId, workspaceId, ... }   │
│  • Serializa meta em header interno __mcp-meta (JSON)                    │
│  • Obtém stub DO por sessão → stub.fetch(requestWithMeta)                │
└─────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    Durable Object (MyMCP)                                  │
│  • fetch() lê __mcp-meta → env._customMeta = parsed                      │
│  • super.fetch() → protocolo MCP, despacho para tools                      │
│  • init() registra tools uma vez por instância                            │
└─────────────────────────────────────────────────────────────────────────┘
                                      │
                    ┌─────────────────┼─────────────────┐
                    ▼                 ▼                 ▼
              ┌──────────┐    ┌──────────────┐   ┌─────────────┐
              │  Plati   │    │ API negócio  │   │ KV / D1     │
              │  API     │    │ (REST)       │   │ (cache)     │
              └──────────┘    └──────────────┘   └─────────────┘
```

---

## 2. Fluxo de uma requisição (ponto a ponto)

1. **HTTP chega** ao Worker (path típico do MCP/SSE conforme `McpAgent` + rotas).
2. **Autenticação** — comparação de `x-api-key` com `env.MCP_API_KEY` (ou regra especial: em alguns projetos, presença de `x-contact-id` / `x-conversation-id` como sinal de chamada Plati confiável, combinada com outras validações).
3. **`extractCustomMeta(request)`** — lê headers `x-*` e opcionalmente `user-agent`, monta objeto com `timestamp`.
4. **Clonagem do request** com header adicional **`__mcp-meta`**: JSON stringificado do `CustomMeta`. O Worker é o único que escreve esse header antes do DO.
5. **Resolução do Durable Object** — por ID de sessão (depende do binding `McpAgent` / `MyMCP` no `wrangler`). O cliente envia um identificador de sessão que determina qual instância DO atende.
6. **`MyMCP.fetch`** — se existir `__mcp-meta`, faz `JSON.parse` e atribui `this.env._customMeta = customMeta`.
7. **`super.fetch`** — processamento MCP (tools/list, tools/call, etc.).
8. **Tool handler** — usa `getRequestMetadata(env)` ou `env._customMeta` para `contactId`, `conversationUid`, etc., chama APIs com `fetch`, retorna `content` (texto/JSON) para o modelo.

Ordem importa: metadados devem estar **definidos antes** do handler da tool porque o modelo não “passa” `contactId` — vem do ambiente da sessão.

---

## 3. Por que `__mcp-meta` e não só headers

- **Headers externos** (`x-contact-id`, …) são enviados pelo cliente na **borda** do Worker.
- O Worker **confia** na política de quem chama (Plati, rede interna). Dentro do DO, o código precisa de um objeto **já parseado** e **único por request**.
- O header **`__mcp-meta`** é **interno**: só o Worker adiciona ao request encaminhado ao DO. O cliente típico **não** envia `__mcp-meta` diretamente; assim reduz-se risco de spoofing de metadados se o cliente não for o Worker.
- Em **testes** (`curl`, Inspector), você pode simular o fluxo completo ou só o Worker; o importante é que **produção** mantenha a cadeia Worker → DO.

*(Se a rota aceitar o MCP direto do cliente sem Worker no meio, reavalie o modelo de confiança — o desenho “oficial” pressupõe Worker como gatekeeper.)*

---

## 4. Durable Object, sessão e isolamento

### Isolamento

- Cada **instância DO** é isolada pelo runtime **Cloudflare**. Memória de `env` não vaza entre instâncias.
- **`env._customMeta`** é por instância; corresponde ao último request que **aquela** instância processou naquele ciclo.

### Sessão MCP e `x-conversation-id`

- O **mcp-session-id** (ou equivalente) determina **qual** DO. Se **conversas diferentes** reutilizarem o mesmo ID de sessão MCP, há risco de **misturar contexto** (produtos indo para o usuário errado).
- **Recomendação operacional:** alinhar com o time Plati para que **cada conversa** (ou cada usuário) tenha **sessão MCP estável e única** ou que o roteamento inclua `x-conversation-id` de forma consistente com o ID do DO.
- Alguns repositórios logam **auditoria** de colisão quando o mesmo DO recebe `conversationUid` diferentes — útil para detectar bugs de integração.

### Concorrência

- Dentro de um DO, execução é **single-threaded** por isolado; não há race em `env._customMeta` entre duas mensagens simultâneas **do mesmo** DO, mas **duas requisições** podem intercalar se o modelo de runtime permitir. Por isso **código crítico** (ex.: capturar `conversationUid`/`contactId` **no início** do handler da tool, antes de `await` longos) evita sobrescrita de `env` entre awaits.

---

## 5. Autenticação em camadas

| Camada | O quê | Onde |
|--------|--------|------|
| **MCP / Worker** | `x-api-key` ↔ `MCP_API_KEY` (segredo no Worker) | `fetch` do Worker |
| **Plati API** | `x-api-key` ↔ `PLATI_API_KEY` (segredo no Worker) | `fetch` para `PLATI_API_URL` |
| **Identidade do usuário** | Não é “senha do usuário” — é `contactId` + resolução de telefone via Plati | Tools que precisam de telefone |

**Separar** chave do MCP (quem pode chamar **seu** servidor) da chave da Plati (quem pode chamar **a** Plati). Em cenários multi-tenant, a chave Plati pode vir de **lookup** ou de **um serviço intermediário**; ver [§10](#10-multi-workspace-e-credenciais).

---

## 6. Metadados (`CustomMeta`) — referência

Campos comuns (nomes exatos podem variar entre repos):

| Campo no objeto | Origem típica | Uso |
|-----------------|-----------------|-----|
| `contactId` | `x-contact-id` | Autorização, lookup de telefone, `GET /contacts/{id}` |
| `conversationUid` | `x-conversation-id` | `POST .../conversations/{conversationUid}/messages` |
| `channelId` | `x-channel-id` | Contexto de canal, rotas que incluem channel |
| `workspaceId` | `x-workspace-id` | Multi-tenant, logs, futuras políticas |
| `requestId` | `x-request-id` | Correlação e rastreio |
| `userAgent` | `user-agent` | Analytics, debug |
| `timestamp` | gerado no Worker | Ordem aproximada / TTL |

**Função helper:** `getRequestMetadata(env): CustomMeta | undefined` — retorna `env._customMeta`.

**Tools genéricas** (`get_current_time`, utilitários sem side effect Plati) podem **ignorar** metadados ou usar só `requestId` para logs.

---

## 7. Integração Plati (APIs e padrões)

### Base URL

- Variável **`PLATI_API_URL`** (ex.: ambiente beta vs produção). Normalizar sempre **sem** barra final duplicada ao concatenar paths.

### Contato e telefone

- **`GET {PLATI_API_URL}/channels/{PLATI_CHANNEL_ID}/contacts/{contactId}`**  
  Retorna dados do contato; o **telefone** é usado para APIs externas que **não** devem aceitar telefone digitado pelo modelo.
- Helpers típicos: `getPlatiContact`, `getPhoneFromContactId` (com cache em **KV** opcional para reduzir chamadas).

### Mensagens (conversas)

- **`POST .../conversations/{conversationUid}/messages`** — mensagem na conversa atual.
- Alternativa por canal + telefone:  
  **`POST .../conversations/channels/{channelUid}/contacts/{phone}/messages`**  
  quando `conversationUid` não existe ou retorna 404 (ver [§8](#8-mensagens-whatsapp--tipos-limites-e-fallbacks)).

### Bloqueio / moderação

- Fluxos como “remover cadastro” podem chamar **PATCH** ou endpoint de **block** no canal — depende da API Plati exposta ao integrador.

### Métricas e dashboard

- Alguns MCPs implementam **rotas extras** no Worker (`/dashboard`, `/api/...`) com **proxy** autenticado para APIs Plati de métricas (contatos, conversas, mensagens), alimentando HTML ou JSON.

---

## 8. Mensagens WhatsApp — tipos, limites e fallbacks

### `messageType: "text"`

- Estrutura com `contents[]` e blocos `type: "text"`, `data.text`.
- Limites de tamanho do WhatsApp (ex.: **1024** caracteres no corpo em muitos casos) — **truncar** no sender.

### `messageType: "interactive"`

- Combina **texto**, **header** (imagem, título), **botões** (URL, reply), **footer**.
- Texto de botão URL: limite curto (ex.: **20 caracteres**) — abreviar rótulos (“Ver produto”).
- Imagens: URLs externas podem ser bloqueadas por hotlink; alguns MCPs usam **`/img-proxy?url=...`** no **mesmo Worker** para servir imagens permitidas.

### Normalização de telefone (Brasil)

- Dígitos apenas; prefixo **55** quando necessário; consistente com o que a API Plati espera no path.

### Fallback 404 na conversa

1. Tentar `POST .../conversations/{conversationUid}/messages`.
2. Se **404** (“conversation not found”), tentar **telefone** via `getPhoneFromContactId` + `PLATI_CHANNEL_ID` no path de **channel + contact**.

Implementação típica em um helper como `send-loading-message.ts`.

---

## 9. Padrões de UX: loading, fire-and-forget e `waitUntil`

### Mensagem “estou processando”

- **Objetivo:** reduzir percepção de latência em APIs lentas (busca externa, etc.).
- **Fire-and-forget:** função dispara `fetch` Plati e **não** espera — `void sendPromise` ou `.catch(() => {})` no final para não quebrar o runtime; a tool continua e retorna quando a API externa responder.
- **Await + delay:** em fluxos de busca pesada, envia mensagem de loading, **espera** o envio e um **delay curto** (~800 ms) para o usuário ler antes de iniciar a operação lenta.

### Trabalho em background (não é mensagem)

- **`ctx.waitUntil(promise)`** (quando `ExecutionContext` está disponível no agent): **mantém** o Worker vivo para tarefas que **não** bloqueiam a resposta MCP — cache, lembretes, crawl assíncrono.
- Distinto de **fire-and-forget** para WhatsApp: `waitUntil` é para **continuar** trabalho após responder ao cliente MCP.

---

## 10. Multi-workspace e credenciais

### Objetivo

Um **único código** e idealmente um **único deploy** atendendo **vários workspaces** Plati, sem criar um repositório novo por cliente.

### O que é simples

- Passar **`x-workspace-id`** em todo request e usar para **logs**, **feature flags** ou **roteamento lógico** dentro das tools.

### O que é difícil

- Se **cada workspace** exige **API key Plati diferente** e a API **não** aceita uma **única** chave de integração com escopo multi-workspace, você precisa de **um** destes:
  - **Lookup** `workspaceId → credencial` guardada em **KV/D1/Secrets** (provisionado pelo produto Plati ou painel).
  - **Proxy interno** na Plati: o MCP chama só a Plati com **um** token de serviço; a Plati aplica a key certa por workspace.
  - **N deploys** do mesmo código (Wrangler environments / Terraform) com envs diferentes por workspace — mesmo código, **N** configurações.

Não há mágica: se a secret não pode estar no Worker único nem em lookup nem atrás de proxy, **N deploys** ou **mudança de API** são os caminhos.

---

## 11. Bindings Cloudflare (KV, D1, etc.)

| Binding | Uso típico nos MCPs |
|---------|-------------------|
| **KV** | Cache de telefone por `contactId`, rate limit por chave, flags. |
| **D1** | Produtos cacheados, analytics, Q&A de eventos, mensagens. |
| **Vectorize** | Opcional (FAQ, busca semântica) — nem todo MCP usa. |
| **Secrets** | `wrangler secret put` — `MCP_API_KEY`, `PLATI_API_KEY`, chaves de APIs terceiras. |

---

## 12. Logs — papel, conteúdo e o que nunca expor

### Papel

- **Depuração** (reproduzir falhas com `requestId`).
- **Auditoria** (quem chamou qual tool e quando).
- **Operação** (taxa de erro, latência percebida).
- **Não** substituir o **error-handler** para o usuário final — logs são para **equipe** e **ferramentas**.

### Boas práticas

- Prefixar linhas com **tag** estável: `[SEARCH]`, `[CREATE_CONTACT]`, `[MCP]`.
- Incluir **`requestId`** quando existir.
- **Truncar** IDs: `contactId.slice(0, 8)`.
- **Nunca** logar: telefone completo, e-mail, tokens, corpo de chave API, PII desnecessária.

### Camadas extras

- **request-logger** / **tool-logger**: persistir ou expor endpoint de consulta (útil em suporte). Mesmo assim, **cuidado com GDPR** e retenção.

### Correlação

- Do log do Worker ao log do DO ao log da tool: **mesmo** `x-request-id` liga o fluxo.

---

## 13. Erros em duas camadas (`error-handler`)

- **Camada usuário / modelo:** mensagem **genérica**, sem nomes de API, sem stacktrace, sem IDs internos sensíveis.
- **Camada técnica:** `console.error` com detalhes, `error` object, status HTTP quando relevante.

**Tipo de erro** (enum comum): `AUTHENTICATION`, `VALIDATION`, `EXTERNAL_API`, `RATE_LIMIT`, etc. — para mapear resposta segura vs retry.

---

## 14. Rate limiting

- Implementação típica: **KV** com chave por `contactId` ou IP + janela deslizante.
- **Objetivo:** proteger APIs externas e custo; retornar mensagem amigável quando exceder.

---

## 15. Registro de tools e contratos

- **`init()`** do `McpAgent` chama `registerXTools(this.server, this.env)` — import dinâmico (`await import(...)`) opcional para cold start.
- Cada tool: **nome** (`snake_case` estável), **descrição** (para o modelo escolher quando usar), **schema** (Zod ou objeto compatível com SDK), **handler** async.
- **Descrições** devem ser claras sobre **quando** usar e **quais** args são obrigatórios.

---

## 16. Exemplo estendido de tool

```typescript
import type { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { z } from "zod";
import { getRequestMetadata } from "../index"; // ou módulo dedicado
import { checkRateLimit } from "../lib/rate-limit";
import { createErrorResponse, ErrorType } from "../lib/error-handler";

const InputSchema = z.object({
  query: z.string().min(1).describe("Termo de busca do usuário"),
  limit: z.number().int().min(1).max(10).optional().describe("Máximo de resultados"),
});

export function registerExampleSearchTool(server: McpServer, env: Env) {
  server.tool(
    "example_search",
    "Busca exemplo: requer sessão Plati com contact id. Usa apenas metadados da requisição.",
    {
      query: z.string().min(1).describe("Termo de busca do usuário"),
      limit: z.number().int().min(1).max(10).optional().describe("Máximo de resultados"),
    },
    async (args) => {
      const parsed = InputSchema.safeParse(args);
      if (!parsed.success) {
        return {
          content: [{ type: "text", text: "Argumentos inválidos." }],
          isError: true,
        };
      }

      const meta = getRequestMetadata(env);
      const requestId = meta?.requestId ?? "no-request-id";
      console.log(
        `[example_search] requestId=${requestId} contact=${meta?.contactId?.slice(0, 8) ?? "none"} query=${parsed.data.query.slice(0, 40)}`,
      );

      if (!meta?.contactId) {
        return createErrorResponse(
          "Sessão sem contato identificado.",
          ErrorType.AUTHENTICATION,
          { technicalMessage: "Missing x-contact-id" },
        );
      }

      const rl = await checkRateLimit(env, meta.contactId, "example_search");
      if (!rl.ok) {
        return createErrorResponse(
          "Muitas solicitações. Tente novamente em instantes.",
          ErrorType.RATE_LIMIT,
          {},
        );
      }

      try {
        // const results = await fetchExternalApi(...);
        return {
          content: [
            {
              type: "text",
              text: JSON.stringify({ ok: true, results: [] }, null, 2),
            },
          ],
        };
      } catch (e) {
        console.error("[example_search]", requestId, e);
        return createErrorResponse(
          "Não foi possível concluir a busca agora.",
          ErrorType.EXTERNAL_API,
          { technicalMessage: e instanceof Error ? e.message : String(e) },
        );
      }
    },
  );
}
```

### Checklist ao adicionar uma tool

1. Validar entrada com Zod (ou equivalente).
2. Exigir `contactId` / `conversationUid` quando a ação for ligada à conversa Plati.
3. Obter telefone via **`getPhoneFromContactId`** quando a API exigir número verificado.
4. Retornar erro claro para o modelo; detalhes sensíveis só em log.
5. Respeitar **rate limit** e políticas do produto.
6. Capturar metadados **no início** do handler se houver `await` longos (evitar race em `env._customMeta`).

---

## 17. Testes e MCP Inspector

- Arquivos como `mcp-inspector-local.json`, `mcp-inspector-prod.json` — comandos para subir o **MCP Inspector** apontando para `localhost` ou URL de produção.
- Scripts em `tests/manual/*.sh`: testes de **SSE**, headers, **sessão**, **múltiplas sessões**.
- Para reproduzir bugs: mesmo `x-request-id` + mesmos headers Plati.

---

## 18. Deploy e secrets

- **`wrangler deploy`** — ambiente de produção.
- **Secrets:** nunca commitar `.dev.vars`; usar `.dev.vars.example` como template sem valores reais.
- **Variáveis** por ambiente: `wrangler.jsonc` com `vars` não secretas; secrets via CLI ou dashboard.

---

## 19. Anti-padrões

- Aceitar **telefone** ou **email** como argumento da tool quando a política exige **telefone só via Plati**.
- Logar **PII** completo para “debug rápido” em produção.
- Confiar em `__mcp-meta` vindo do cliente sem passar pelo Worker.
- **Assumir** `env._customMeta` imutável após `await` longo sem capturar valores no início.
- **Prometer** ao usuário sucesso antes do retorno da tool (regra de produto em vários `prompt.md`).

---

## 20. Glossário

| Termo | Significado |
|-------|-------------|
| **MCP** | Model Context Protocol — JSON-RPC sobre HTTP/SSE para listar e invocar tools. |
| **DO** | Durable Object — estado isolado por ID de instância. |
| **Plati** | Plataforma de conversas (WhatsApp, etc.) e APIs de contato/mensagem. |
| **CustomMeta** | Objeto com contexto da requisição (contact, conversation, workspace, …). |
| **Fire-and-forget** | Disparar envio Plati sem `await` na tool principal. |
| **waitUntil** | Registrar promise no Worker para trabalho pós-resposta. |

---

## 21. Este documento é suficiente para uma IA “implementar tudo”?

**Em parte.** Ele deixa **claro o desenho** (Worker → `__mcp-meta` → DO → tools, Plati, logs, erros, rate limit). Isso é o bastante para gerar um **esqueleto correto** e evitar erros conceituais comuns.

O que **não** está fechado o suficiente para uma implementação **única e sem decisões** — e qualquer modelo vai precisar **escolher** ou **perguntar**:

| Lacuna | Por quê |
|--------|---------|
| **Versões exatas** de `agents/mcp`, `@modelcontextprotocol/sdk`, `wrangler`, Node | O doc não fixa `package.json`; a IA deve alinhar com o que o Cloudflare documenta hoje ou com o template oficial de MCP. |
| **`wrangler.jsonc` concreto** | Nomes do binding do Durable Object, `name` da classe, rotas — dependem do template oficial `agents/mcp` na data do deploy. |
| **API Plati** | Paths e contratos podem mudar; o doc descreve o padrão, não substitui OpenAPI/Swagger oficial ou ambiente de testes. |
| **Escopo “tudo”** | O texto mistura **padrões opcionais** (D1, Vectorize, image proxy, `waitUntil`, tool-logger). “Implementar tudo” sem prioridade vira escopo infinito. |
| **Multi-workspace com N keys** | A §10 deixa explícito que há **trade-offs**; não há uma receita única sem decisão de produto/API. |

**Recomendações para o repo novo:**

1. Coloque no README: **“Escopo MVP = fases abaixo; o resto é opcional.”**
2. Anexe ou linke a **OpenAPI Plati** (ou Postman) que o time usar.
3. Se possível, **referencie um MCP interno mínimo já em produção** como “diff mental”, mesmo sem copiar código — reduz alucinação de wiring.

---

## 22. Escopo MVP e checklist para implementação (repo novo)

Use esta ordem; só avance quando a fase anterior **compilar e responder no Inspector**.

### Fase 0 — Decisões (1 parágrafo no README)

- [ ] **Uma** Plati API key no Worker (um workspace) **ou** estratégia multi-tenant já escolhida (lookup, N deploys, etc.).
- [ ] Lista de **tools** do primeiro release (ex.: só `get_current_time` + `ping`).

### Fase 1 — Worker + DO + MCP vazio

- [ ] Projeto TypeScript + Wrangler + `McpAgent` / `MyMCP` conforme template Cloudflare para MCP.
- [ ] `export default { fetch }` validando `x-api-key` → `MCP_API_KEY`.
- [ ] `extractCustomMeta` + injeção de `__mcp-meta` + `getRequestMetadata`.
- [ ] Tool mínima **`ping`** ou **`get_current_time`** (sem Plati) para validar fim a fim.

### Fase 2 — Integração Plati (leitura)

- [ ] `PLATI_API_URL`, `PLATI_API_KEY`, `PLATI_CHANNEL_ID` em secrets.
- [ ] `getPlatiContact` / `getPhoneFromContactId` + teste manual com `contactId` real ou mock.

### Fase 3 — Integração Plati (escrita)

- [ ] Envio de mensagem `POST .../conversations/{uid}/messages` (texto simples).
- [ ] (Opcional) fallback channel + telefone se 404.

### Fase 4 — Qualidade

- [ ] `error-handler` (mensagem segura vs log técnico).
- [ ] Rate limit mínimo (KV) se exposto publicamente.
- [ ] Logs com `requestId` e truncamento de IDs.

### Fase 5 — Opcionais (só se necessário)

- [ ] Mensagem loading fire-and-forget.
- [ ] D1 / analytics / dashboard.
- [ ] `waitUntil` para tarefas em background.

### Definição de “pronto” (MVP)

- Conectar com **MCP Inspector** ou cliente Plati, chamar tool com headers `x-contact-id` / `x-conversation-id`, ver resposta correta e logs correlacionados **sem** vazar PII.

---

*Documento genérico: ajuste nomes de env (`PLATI_*`, `MCP_API_KEY`), paths e bindings conforme cada `wrangler.jsonc` e versão da API Plati.*
