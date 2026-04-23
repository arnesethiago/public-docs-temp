# API Plati — contatos no workspace (API Key)

**Base URL de produção:** `https://api.plati.ai`  
**Versão:** paths prefixados com `/v1`.

---

## Autenticação

- Header: `x-api-key: <sua-api-key>`
- A chave pertence ao **workspace**. O backend resolve o workspace a partir da key (não é necessário enviar `workspaceUid` na URL para esses fluxos).

---

## 1. Listar canais do workspace

Usado para obter o **`channelUid`** antes de listar ou criar contatos (contatos são sempre por **canal**).

| | |
|---|---|
| **Método / path** | `GET /v1/channels` |
| **Body** | *(nenhum)* |

### Query params (opcionais)

| Param | Descrição |
|-------|-----------|
| `type` | Filtrar por tipo de canal |
| `status` | Filtrar por status |
| `isActive` | boolean |
| `search` | Busca por nome/descrição |
| `cursor` | Paginação (cursor) |
| `limit` | Tamanho da página |
| `direction` | `before` \| `after` |

### Resposta

Objeto JSON com:

| Campo | Descrição |
|-------|-----------|
| `channels` | Array de canais |
| `hasNext` | Há próxima página |
| `hasPrevious` | Há página anterior |
| `nextCursor` | Cursor da próxima página (se houver) |
| `previousCursor` | Cursor da página anterior (se houver) |

Cada canal inclui, entre outros: `uid`, `name`, `type`, `status`, `isActive`, `configuration`, `settings`, `metadata`, `workspace` (com `uid`, nome, etc.).

**Escopo da API key:** `read` ou `read:write` (leitura).

---

## 2. Listar contatos de um canal

| | |
|---|---|
| **Método / path** | `GET /v1/channels/{channelUid}/contacts` |
| **Body** | *(nenhum)* |

### Query params (opcionais)

Incluem: `identifierType`, `status`, `isActive`, `isVerified`, `marketingOptIn`, `segment`, `search`, `deletedOnly`, `cursor`, `limit`, `direction`.

### Resposta

| Campo | Descrição |
|-------|-----------|
| `contacts` | Array de contatos |
| `hasNext` | Há próxima página |
| `hasPrevious` | Há página anterior |
| `nextCursor` | Próximo cursor |
| `previousCursor` | Cursor anterior |

Cada contato costuma incluir: `uid`, `createdAt`, `updatedAt`, `displayName`, `firstName`, `lastName`, `identifier`, `identifierType`, `externalId`, `status`, `isActive`, `isVerified`, `lastInteractionAt`, `firstInteractionAt`, `tags`, `segment`, `marketingOptIn`, `metadata`, entre outros.

---

## 3. Obter um contato específico

| | |
|---|---|
| **Método / path** | `GET /v1/channels/{channelUid}/contacts/{contactUid}` |
| **Body** | *(nenhum)* |

### Resposta

Um único objeto de contato (mesma ideia dos itens da listagem).  
`metadata` é um objeto livre (chave/valor): pode refletir dados de canal (ex.: WhatsApp), integrações ou campos que você enviou na importação.

---

## 4. Incluir contatos (criar ou atualizar)

**`POST /v1/channels/{channelUid}/contacts/import`** — array `contacts` com **1 a 1000** itens por requisição. Para alterar um contato que já existe, use também **`PUT /v1/channels/{channelUid}/contacts/{contactUid}`** quando fizer sentido.

| | |
|---|---|
| **Método / path** | `POST /v1/channels/{channelUid}/contacts/import` |
| **Headers** | `Content-Type: application/json`, `x-api-key: ...` |

### Corpo da requisição

Objeto JSON de primeiro nível:

| Campo | Obrigatório | Default | Descrição |
|-------|-------------|---------|-----------|
| `contacts` | **Sim** | — | Array de contatos (1 a **1000** entradas) |
| `updateExisting` | Não | `false` | Atualizar se já existir |
| `skipErrors` | Não | `true` | Continuar apesar de erros por linha |
| `createIdentities` | Não | `false` | Criar identidades automaticamente |

Cada elemento de `contacts`:

| Campo | Obrigatório | Notas |
|-------|-------------|--------|
| `identifier` | Sim | Telefone, e-mail, etc. |
| `identifierType` | Sim | `PHONE`, `EMAIL`, `USERNAME`, `USER_ID`, `EXTERNAL_ID`, `CUSTOM` |
| `externalId` | Não | ID externo |
| `displayName`, `firstName`, `lastName` | Não | |
| `avatarUrl` | Não | |
| `status` | Não | ex.: `ACTIVE` |
| `isVerified` | Não | |
| `metadata` | Não | Objeto livre |
| `preferences` | Não | Objeto livre |
| `customFields` | Não | ex.: CPF, empresa |
| `language`, `timezone` | Não | |
| `marketingOptIn`, `notificationsEnabled` | Não | |
| `tags`, `segment` | Não | |
| `lastInteractionAt` | Não | ISO 8601 ou `null` |

### Resposta (import)

Além dos contadores do lote, a API devolve um array **`contacts`** com os registros efetivamente criados ou atualizados. Cada item inclui pelo menos:

| Campo em `contacts[]` | Descrição |
|------------------------|-----------|
| `uid` | **`contactUid`** gerado pela Plati (use nas URLs `GET`/`PUT` desse contato) |
| `identifier` | Identificador normalizado |
| `identifierType` | Tipo do identificador |
| `displayName` | Nome de exibição (se houver) |
| `externalId` | Seu ID externo (se enviou no import) |

Ou seja: **o fluxo que “gera” o `contactUid` é o import** — inclusive com um único objeto em `contacts`. Não é possível enviar o UUID do contato na criação; ele é definido pelo servidor. Para guardar o ID do seu CRM/ERP, use o campo **`externalId`** (e/ou `metadata`).

| Campo (nível raiz) | Descrição |
|--------------------|-----------|
| `total` | Total processado no lote |
| `success` | Novos contatos criados |
| `updated` | Contatos já existentes atualizados |
| `skipped` | Ignorados (ex.: duplicado sem `updateExisting`) |
| `errors` | Falhas |
| `errorDetails` | Opcional: lista `{ identifier, error }` |
| `contacts` | Lista dos contatos criados/atualizados com **`uid`** |

### Cuidado: rota com `contactUid` na URL que **não** cria contato

Existe **`POST /v1/conversations/contact/{contactUid}`**, que **cria uma conversa vazia** para um contato que **já existe**. O `{contactUid}` deve ser um UID **já conhecido** (por exemplo obtido na resposta do import ou no `GET` de listagem). Essa rota **não** substitui o import e **não** cria pessoa/contato no canal.

### Exemplo `curl` — um contato com `metadata`

```bash
curl -sS -X POST "https://api.plati.ai/v1/channels/<CHANNEL_UID>/contacts/import" \
  -H "Content-Type: application/json" \
  -H "x-api-key: <SUA_API_KEY>" \
  -d '{
    "contacts": [
      {
        "identifier": "+5511999999999",
        "identifierType": "PHONE",
        "displayName": "João Silva",
        "metadata": {
          "source": "crm",
          "campaignId": "spring-2026",
          "leadScore": 82
        },
        "customFields": {
          "cpf": "123.456.789-00",
          "company": "Acme"
        },
        "tags": ["vip"],
        "segment": "enterprise"
      }
    ],
    "updateExisting": true,
    "skipErrors": true
  }'
```

---

## Escopos da API Key

Valores possíveis: `read`, `write`, `read:write`.

- Operações de **leitura** (`GET` em canais/contatos): precisam de permissão de leitura (`read` ou `read:write`).
- Operações de **escrita** (import, atualizar contato, etc.): precisam de `write` ou `read:write`, conforme a política da sua chave.

---

## Notas práticas

1. Obtenha sempre o **`channelUid`** com `GET /v1/channels` antes de chamar rotas de contatos.
2. Na **listagem**, `metadata` pode ser extenso (canal, integrações, campos customizados).
3. **Paginação:** use `cursor` / `nextCursor` / `previousCursor` quando `hasNext` ou `hasPrevious` for verdadeiro.
4. **Segurança:** trate a API key como segredo; revogue e recrie se vazar.
