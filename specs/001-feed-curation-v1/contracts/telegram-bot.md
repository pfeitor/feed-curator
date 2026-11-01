# Contract: Telegram Bot Delivery, Feedback & Management Interface

**Feature**: `001-feed-curation-v1` | **Date**: 2026-11-01
**Covers**: `FR-001..006`, `FR-022..030`, `FR-040..041`, `SC-001`, `SC-003`, `SC-006`, `SC-010`

O Telegram (via `getUpdates` de saída, sem webhook e sem portas abertas) é a única interface de entrega e interação do usuário na V1, acessível de forma equivalente em Android, Telegram Web e Desktop.

---

## 1. Autorização e Silêncio Obrigatório (FR-027, SC-010)

- Todo update recebido (`message`, `edited_message`, `callback_query`) passa primeiro pelo filtro de autorização estrito:
  `update.from_user.id in TELEGRAM_ALLOWED_USER_IDS and chat.type == "private"`.
- Se o usuário **não** estiver autorizado ou a conversa não for privada:
  - **0 respostas enviadas** (nem mensagem de "acesso negado", nem `answerCallbackQuery` informativo).
  - **0 mutações no banco de estado**.
  - Apenas contador/log sanitizado de evento descartado (sem registrar conteúdo da mensagem nem tokens).

---

## 2. Formato do Digest Diário (FR-022..024, US3)

### 2.1 Cartão de Item (`1..K`, onde `K <= DIGEST_MAX_ITEMS`, apenas nota `>= DIGEST_MIN_SCORE`)

- O item `1` é enviado com `disable_notification=False`; os itens `2..K` e o rodapé usam `disable_notification=True` (apenas 1 notificação sonora no celular).
- **Texto completo da publicação nunca é enviado** (FR-023).
- **Estrutura de renderização (HTML escapado via `html.escape`)**:
  - Modo **Regra Simples** (ativo até aprovação do LLM):
    ```text
    [1/K] 📰 <b>{titulo_escapado}</b>
    Fonte: {feed_title_escapado} · Categoria: {categoria_escapada}{flag_parcial}
    Motivo (regra): {motivo_uma_linha_escapado}
    🔗 <a href="{url_original_validada_http_https}">Ler original</a>
    ```
  - Modo **LLM Aprovado** (somente após `evaluation_decisions.verdict == 'aprovar_llm'`):
    ```text
    [1/K] ⭐ {nota}/5 · <b>{titulo_escapado}</b>
    Fonte: {feed_title_escapado} · Categoria: {categoria_escapada}{flag_parcial}

    {resumo_curto_ptbr_escapado}

    💡 <i>{motivo_uma_linha_escapado}</i>
    🔗 <a href="{url_original_validada_http_https}">Ler original</a>
    ```
- **Regra de truncamento seguro (US3 Cenário 5)**:
  - Limite máximo do Telegram: `4.096` caracteres após parsing.
  - Se o cartão montado ultrapassar `3.800` caracteres, `resumo_curto_ptbr` (e se necessário `titulo`) é truncado com `…`, preservando **obrigatoriamente** o link `<a href="...">Ler original</a>` (validado apenas para esquemas `http://` ou `https://`) e o teclado inline de ações.
- **Teclado Inline por Item** (1 toque no celular, FR-024):
  - Linha única com 3 botões:
    `[👍 Útil]` (`fb:{delivery_id}:u`) | `[👎 Não útil]` (`fb:{delivery_id}:d`) | `[🔖 Guardar]` (`fb:{delivery_id}:s`)

### 2.2 Cartão de Fechamento do Digest (FR-040)

Enviado após os `K` cartões do dia:

```text
📬 <b>Digest {YYYY-MM-DD}</b> — {K} item(ns) entregues (modo: {regra_simples | llm}{sufixo_sombra}).
Sentiu falta de alguma notícia importante hoje?
```

- **Teclado Inline do Digest**:
  `[⚠️ Faltou algo hoje]` (`dg:{YYYYMMDD}:m`)

### 2.3 Política de Dia Vazio (US3 Cenário 3)

- Quando `0` itens atingem o limiar: nenhuma mensagem de digest é enviada por padrão, e `digest_runs` grava `status='empty_day'` com `empty_reason='no_items_above_threshold'` (satisfazendo o `watchdog-check`).

---

## 3. Gramática de `callback_data` (Limite Telegram: 1–64 bytes ASCII)

| Padrão | Bytes máx. | Semântica | Efeito no Estado |
|---|---|---|---|
| `fb:<delivery_id>:u` | ≤ 18 B | Marcar item como `"útil"` / `"valeu"` | Insere evento em `feedback_events` (`reaction_axis='vote'`, `reaction_value='util'`), substituindo voto anterior no eixo `vote` com histórico preservado (FR-029). |
| `fb:<delivery_id>:d` | ≤ 18 B | Marcar item como `"não útil"` / `"lixo"` | Insere evento em `feedback_events` (`reaction_axis='vote'`, `reaction_value='nao_util'`), substituindo voto anterior no eixo `vote`. |
| `fb:<delivery_id>:s` | ≤ 18 B | Alternar `"guardar para depois"` | Insere evento em `feedback_events` (`reaction_axis='bookmark'`, `guardar`/`desguardar`); **não** conta como `"útil"` nem altera voto `u`/`d` (FR-030). |
| `dg:<YYYYMMDD>:m` | 13 B | Marcar `"faltou algo"` no digest do dia | Insere evento em `feedback_events` (`target_type='digest'`, `reaction_axis='digest_gap'`, `reaction_value='faltou_algo'`) para avaliação do modo sombra (FR-040). |

- Todo callback válido responde com `answerCallbackQuery` curto confirmando a ação (ex.: `"Registrado: 👍 útil (substituiu 👎)"`) e atualiza visualmente o rótulo do botão ativo (`✅ 👍 Útil`) quando possível, sem alterar o texto nem o link original.

---

## 4. Comandos Essenciais de Gestão no Canal (FR-026, FR-028, US5, US6)

| Comando | Argumentos | Comportamento |
|---|---|---|
| `/addfeed <url>` | URL do feed ou do site | Descobre o feed (FR-003), valida XML RSS/Atom, cadastra de forma idempotente e marca todos os itens já existentes no feed como `"já vistos"` (`is_baseline=True`, FR-004). |
| `/feeds` | Opcional: `[active\|paused\|all]` | Lista feeds com estado, falhas, peso a priori, volume observado, janela de cobertura, nota média e tamanho da amostra (FR-006). |
| `/disable <feed_id>` | ID inteiro do feed | Desativa o feed sem apagar histórico nem feedback (FR-005). |
| `/remove <feed_id>` | ID inteiro do feed | Marca o feed como `removed` (exclui da exportação OPML de ativos e de novas coletas, preservando integridade referencial de entregas passadas) (FR-005). |
| `/weight <feed_id> <peso>` | ID + decimal em `[-1.0, 2.0]` | Ajusta `prior_weight` usado exclusivamente pela regra simples (FR-005). |
| `/import` | Anexo `.opml`/`.xml` enviado com legenda `/import` (limite ≤ 1 MB) | Valida tamanho e XML seguro (`defusedxml`), importa feeds de forma idempotente e agenda linha de base `"já vistos"` para os novos (FR-001, US6 Cenário 2). |
| `/export` | Nenhum | Gera e envia como documento anexo `feeds-ativos-YYYY-MM-DD.opml` contendo todos os feeds com `state='active'` (FR-002, US6 Cenário 1). |
| `/cost` | Nenhum | Exibe orçamento mensal, teto diário do dia em `America/Sao_Paulo`, gasto liquidado, reservas pendentes, itens retidos por saldo (`budget_held`) e entregas em estado `incerto` (FR-020, FR-021, FR-028). |
| `/retry_uncertain <delivery_id>` | ID inteiro de entrega em estado `incerto` | Único caminho permitido para reenviar uma entrega `incerto` por ação explícita do usuário (FR-026, US4 Cenário 2). |

---

## 5. Avisos Operacionais ao Usuário (FR-007, FR-020, FR-033, FR-041)

1. **Feed pausado automaticamente (FR-007)**: enviado pelo `run-once` quando um feed atinge `MAX_FEED_FAILURES` falhas consecutivas, informando `feed_id`, título e último erro sanitizado.
2. **Teto diário esgotado (FR-020)**: enviado no máximo 1 vez por dia quando itens ficam em `budget_held`, informando quantos itens aguardam na fila sem rajada compensatória.
3. **Inatividade no modo sombra (FR-041)**: enviado quando `now - max(feedback_events.created_at) >= 7 dias` durante período com `shadow_enabled=True`.
4. **Alerta independente `"digest não enviado hoje"` (FR-033, SC-008)**: enviado **exclusivamente** pelo `watchdog-check` (usando `ALERT_TELEGRAM_BOT_TOKEN` / webhook independente) ou pelo dead-man's-switch externo quando o digest esperado do dia não consta como `confirmed` nem `empty_day`.
