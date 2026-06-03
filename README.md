# Atendimento IA + Chatwoot (n8n)

Primeiro atendimento de suporte com IA, integrado ao [Chatwoot](https://www.chatwoot.com/) e
orquestrado em [n8n](https://n8n.io/). O bot entende a mensagem do cliente, consulta uma base
de conhecimento (RAG), e **sempre passa por aprovação humana** antes de responder. Inclui
transferência para atendimento N1, encerramento automático por inatividade e pesquisa de
satisfação (CSAT).

> ⚠️ **Projeto de referência / estudo.** Os workflows foram exportados de um ambiente real e
> **sanitizados**: domínios, tokens, credenciais, e-mails e dados pessoais foram substituídos por
> placeholders. Antes de usar, ajuste as configurações e crie suas próprias credenciais no n8n.

## Arquitetura

```
Cliente (Chatwoot widget)
        │  webhook: message_created
        ▼
┌─────────────────────────────┐
│ Orquestrador Chatwoot        │  ingestão → contexto → raciocínio IA → aprovação → execução
│  • filtra msg do cliente     │
│  • controle de estado (SQL)  │
│  • buffer de mensagens (Redis, debounce)
│  • AI Agent (RAG + memória)  │  → { acao: responder | transferir | encerrar }
└───────┬──────────┬───────────┘
        │          │
   responder    transferir/encerrar
        │
        ▼
┌─────────────────────────────┐
│ SUB: Aprovação Humana        │  posta sugestão na conversa central + Redis (TTL 30min)
└───────────────┬─────────────┘
                ▼
        Dashboard Aprovações  ── analista clica Aprovar/Reprovar (iframe no Chatwoot)
                │
                ▼
┌─────────────────────────────┐
│ SUB: Callback Aprovação      │  aprovado → responde cliente / reprovado → N1
└─────────────────────────────┘
```

## Workflows

| Arquivo | Papel |
|---|---|
| [`orquestrador-chatwoot.json`](workflows/orquestrador-chatwoot.json) | Fluxo principal: webhook, estado, buffer, AI Agent (RAG + memória), roteamento de ação. |
| [`sub-aprovacao-humana.json`](workflows/sub-aprovacao-humana.json) | Envia a resposta sugerida pela IA para a conversa central de aprovação e guarda a pendência no Redis. |
| [`sub-callback-aprovacao-humana.json`](workflows/sub-callback-aprovacao-humana.json) | Webhooks `/aprovar` e `/reprovar`: executa a decisão do analista. |
| [`dashboard-aprovacoes.json`](workflows/dashboard-aprovacoes.json) | App de dashboard (HTML) embutido na sidebar do Chatwoot, lista pendências e chama aprovar/reprovar. |
| [`sub-resumo-chat.json`](workflows/sub-resumo-chat.json) | Resume a conversa via IA e posta como nota privada no Chatwoot. |
| [`sub-envia-mensagem.json`](workflows/sub-envia-mensagem.json) | Centraliza o envio de mensagens para a API do Chatwoot. |
| [`timeout-de-cliente.json`](workflows/timeout-de-cliente.json) | Encerra a conversa se o cliente não responder em ~10 min. |
| [`csat-ia.json`](workflows/csat-ia.json) | Recebe a nota/comentário do CSAT e registra em planilha. |

> Há uma dependência externa não incluída neste repositório: o subworkflow
> **`SUB: Suporte RAG - Métricas`**, chamado pelo orquestrador para registrar métricas de
> pergunta/resposta. O fluxo funciona sem ele (a chamada tem `onError: continueRegularOutput`).

## Pré-requisitos

- n8n (self-hosted) com os nós LangChain (`@n8n/n8n-nodes-langchain`)
- Chatwoot com um inbox de widget
- Redis (buffer de mensagens e pendências de aprovação)
- Microsoft SQL Server (controle de estado das conversas)
- PostgreSQL com `pgvector` (RAG + memória do chat)
- Conta OpenAI
- (Opcional) Google Sheets para o log de CSAT

## Configuração

1. **Importe** cada arquivo de `workflows/` no seu n8n.
2. **Crie as credenciais** (os IDs foram removidos na sanitização — o n8n vai pedir para vincular):
   - HTTP Header Auth do Chatwoot (header `api_access_token`)
   - OpenAI, PostgreSQL, Microsoft SQL, Redis, (Google Sheets)
3. **Substitua os placeholders** nos workflows:
   - `chatwoot.example.com` / `n8n.example.com` → seus domínios
   - `YOUR_CHATWOOT_API_TOKEN` → seu token do Chatwoot
   - `YOUR_GOOGLE_SHEET_ID` → ID da sua planilha de CSAT
4. **Constantes** que talvez precise ajustar ao seu ambiente:
   - `account_id` do Chatwoot (exemplos usam `1`)
   - ID da **conversa central** de aprovações (exemplos usam `300`)
   - IDs de **teams**: bot = `1`, N1 = `2`
5. **Tabelas** esperadas:
   - SQL Server: `n8n_controle_atendimento_chatwoot`, `chatwoot_auto_close`
   - Postgres: `n8n_rag_support_tickets` (pgvector), `n8n_chat_support_memory`
6. **Webhooks** a configurar no Chatwoot (Automations / Dashboard Apps):
   - `message_created` → `/webhook/chatwoot_message_created`
   - Dashboard App apontando para `/webhook/dashboard`

## Segurança

Os arquivos foram limpos antes do commit, mas **se você exportou de um ambiente seu, rotacione
qualquer token que já tenha sido versionado** — uma vez em git, considere-o comprometido. Nunca
faça commit de credenciais reais; use as credenciais do próprio n8n.

## Licença

[MIT](LICENSE).
