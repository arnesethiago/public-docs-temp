## API Plati — bloquear / desbloquear contato (API Key)

**Base URL:** `https://api.plati.ai`  
**Versão:** paths com prefixo `/v1`.

### Autenticação

| Header | Valor |
|--------|--------|
| `x-api-key` | Chave da API do workspace |

Para **block**, se enviar JSON:

| Header | Valor |
|--------|--------|
| `Content-Type` | `application/json` |

---

### Bloquear contato

| | |
|---|---|
| **Método** | `PATCH` |
| **Path** | `/v1/channels/{channelUid}/contacts/{contactUid}/block` |

**Body (opcional)** — objeto JSON:

| Campo | Obrigatório | Descrição |
|-------|-------------|-----------|
| `reason` | Não | Motivo do bloqueio (texto livre, até ~500 caracteres na API interna) |

**Exemplo sem motivo:**

```bash
curl -sS -X PATCH \
  "https://api.plati.ai/v1/channels/<CHANNEL_UID>/contacts/<CONTACT_UID>/block" \
  -H "x-api-key: <SUA_API_KEY>"
```

**Exemplo com motivo:**

```bash
curl -sS -X PATCH \
  "https://api.plati.ai/v1/channels/<CHANNEL_UID>/contacts/<CONTACT_UID>/block" \
  -H "Content-Type: application/json" \
  -H "x-api-key: <SUA_API_KEY>" \
  -d '{"reason":"Spam ou comportamento inadequado"}'
```

---

### Desbloquear contato

| | |
|---|---|
| **Método** | `PATCH` |
| **Path** | `/v1/channels/{channelUid}/contacts/{contactUid}/unblock` |
| **Body** | *(nenhum)* |

```bash
curl -sS -X PATCH \
  "https://api.plati.ai/v1/channels/<CHANNEL_UID>/contacts/<CONTACT_UID>/unblock" \
  -H "x-api-key: <SUA_API_KEY>"
```

---

### Parâmetros de URL

| Placeholder | Origem |
|-------------|--------|
| `CHANNEL_UID` | UUID do canal (ex.: `GET /v1/channels`) |
| `CONTACT_UID` | UUID do contato (ex.: listagem ou import de contatos) |

---

### Resposta esperada

Em caso de sucesso, a API devolve o objeto do **contato** atualizado (campos como `status`, `isActive`, etc., conforme o contrato público). Em erro comum: `401` (chave inválida), `403` (sem permissão), `404` (canal/contato não encontrado ou fora do workspace da chave).
