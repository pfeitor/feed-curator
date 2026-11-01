# Contract: Pluggable Scorer, Frozen Evaluation Criteria & Backup Manifest

**Feature**: `001-feed-curation-v1` | **Date**: 2026-11-01
**Covers**: `FR-013..021`, `FR-037..042`, `FR-044..045`, `SC-004..006`, `SC-009`

Este documento especifica os contratos de dados entre o núcleo da aplicação e: (1) provedores de pontuação (`Scorer`), (2) arquivo de critérios congelados de avaliação (`frozen-criteria.yaml`) e (3) manifesto de backup cifrado (`backup-manifest.json`).

---

## 1. Contrato do Provedor de Pontuação (`Scorer` — FR-013..016, FR-037)

Qualquer provedor de LLM (Anthropic, OpenAI, Gemini ou mock de teste) e o avaliador determinístico (`SimpleRuleScorer`) implementam a mesma fronteira de entrada/saída, sem expor detalhes de SDK ao restante do sistema (FR-015).

### 1.1 Payload de Entrada (`ScoringInput`) — Fronteira de Privacidade e Untrusted Data

Somente os campos abaixo podem ser enviados ao provedor externo (FR-037):

```json
{
  "entry_id": 1042,
  "title": "Título sanitizado da publicação",
  "content_text": "Texto extraído ou trecho parcial delimitado como dado não confiável",
  "is_partial": false,
  "rubric_version": "rubric-v1",
  "rubric_criteria_ptbr": "Texto aprovado da rubrica...",
  "approved_examples": [
    {
      "title": "Exemplo aprovado 1",
      "excerpt": "Trecho sem PII...",
      "expected_score": 5,
      "reason_ptbr": "Motivo aprovado..."
    }
  ]
}
```

**Regras de segurança obrigatórias na chamada (FR-016, FR-037)**:
1. **Zero ferramentas**: `tools` / `functions` omitidos ou vazios; o modelo não possui capacidade de chamar comandos, rede, banco ou Telegram.
2. **Isolamento de prompt injection**: `title` e `content_text` trafegam dentro de delimitadores explícitos de dados não confiáveis, com instrução de sistema proibindo seguir comandos contidos no artigo.
3. **Proibição de PII/segredos**: nunca enviar `TELEGRAM_CHAT_ID`, `actor_user_id`, histórico bruto de votos não revisados, URLs com tokens privados ou variáveis de ambiente.
4. **Pré-condição contábil (FR-019)**: o adaptador `LLMScorer` recebe um `reservation_id` ativo em `paid_usage_ledger` (`state='reserved'`); chamadas sem reserva válida levantam erro antes de abrir conexão HTTP.

### 1.2 Schema Estrito de Saída (`ScoringOutput` — Pydantic v2, `extra="forbid"`)

```json
{
  "entry_id": 1042,
  "score": 4,
  "category": "Engenharia de Dados",
  "reason_one_line": "Benchmark reproduzível de particionamento em DuckDB com impacto direto em pipelines locais.",
  "summary_ptbr": "O artigo compara estratégias de leitura parquet no DuckDB 1.1, mostrando redução de memória em cargas incrementais e limites práticos em disco local."
}
```

**Validações determinísticas pós-resposta (FR-013, FR-016, Edge Cases)**:
- `entry_id`: inteiro; **deve ser idêntico** ao `ScoringInput.entry_id` (mitiga confusão de item / injection).
- `score`: inteiro estrito em `{1, 2, 3, 4, 5}` (rejeita floats, strings ou fora de faixa).
- `category`: string não vazia (`1..128` chars), sem tags HTML.
- `reason_one_line`: string de linha única (`10..300` chars, sem `\n` ou `\r`).
- `summary_ptbr`: string em português do Brasil (`20..600` chars).
- **Falha de validação**: a saída é rejeitada (`entries.scoring_status = 'invalid_output'`), o item **não** é entregue com dados inválidos, e os tokens consumidos permanecem debitados em `paid_usage_ledger` (`state = 'failed_charged'`). Qualquer nova tentativa exige nova reserva de orçamento.

---

## 2. Contrato de Critérios Congelados (`config/evaluation/frozen-criteria.yaml` — FR-018, FR-042, SC-005)

Arquivo versionado em Git **antes** da execução da triagem offline e do modo sombra:

```yaml
criteria_version: "criteria-v1"
frozen_at: "2026-11-02T12:00:00Z"
author: "paulo"
rare_definition_ptbr: >
  Definição em 1 a 2 frases escrita pelo usuário do que constitui um item raro
  que ele nunca poderia perder.
rare_reference_examples_file: "config/evaluation/rare-examples-v1.yaml" # 5-10 exemplos reais aprovados (excluídos da amostra de teste)
sample_rules:
  min_total_items: 100
  max_total_items: 150
  target_rare_items: 15
  inconclusive_below_rare_items: 10
decision_gates:
  rare_recall_comparison: "llm_strictly_greater_than_simple_rule" # empate => manter_regra
  wilson_confidence_level: 0.95
  max_projected_monthly_cost_usd: 5.00
  ambiguous_outcome_action: "manter_regra_e_estender_sombra"
reported_only_metrics:
  fatigue_max_bad_items_per_7: 2 # apenas reportado, não aprova nem reprova sozinho
shadow_mode:
  min_duration_days: 14
  max_duration_days: 28
  inactivity_warning_days: 7
```

**Invariante de bloqueio anti-adulteração (US2 Cenário 4, FR-018)**:
- Na promoção (`shadow-decide --verdict aprovar_llm`), o sistema verifica:
  1. Árvore limpa para `config/evaluation/frozen-criteria.yaml` e correspondência de `SHA-256` com `frozen_criteria_records`.
  2. Timestamp do último commit Git que tocou `frozen-criteria.yaml` (e `frozen_at`) é **estritamente anterior** a `min(scores.created_at)` do conjunto avaliado do modelo/rubrica candidato. Caso contrário, retorna código de saída `4` e bloqueia a promoção.

---

## 3. Contrato do Manifesto de Backup (`backup-manifest.json` — FR-038, FR-044, FR-045, SC-009)

Armazenado junto ao arquivo de snapshot do banco dentro do repositório cifrado `restic` (nunca no Git nem na imagem):

```json
{
  "manifest_version": 1,
  "backup_id": "20261102T030000Z-prod",
  "created_at": "2026-11-02T03:00:00Z",
  "environment_label": "prod",
  "app_version": "0.1.0",
  "git_commit": "61a693911fe9b4b392d260b9e5b5cc11f90dde77",
  "db_engine": "sqlite",
  "db_filename": "feed_curator.sqlite3",
  "db_sha256": "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855",
  "alembic_revision": "0001_initial_v1",
  "table_counts": {
    "feeds": 21,
    "entries": 312,
    "entry_sources": 319,
    "rubric_versions": 2,
    "scores": 280,
    "digest_runs": 14,
    "deliveries": 68,
    "feedback_events": 51,
    "daily_budget_ledger": 14,
    "paid_usage_ledger": 195,
    "cycle_runs": 14,
    "evaluation_samples": 120,
    "frozen_criteria_records": 1,
    "evaluation_decisions": 0
  }
}
```

**Checklist obrigatório do comando `restore` antes de tocar no volume `/data` (FR-044, FR-045)**:
1. Exibir `environment_label` (`dev` ou `prod`), `created_at` e `alembic_revision` ao operador.
2. Verificar se o banco destino em `/data` já existe e possui linhas em qualquer tabela de domínio; se possuir e `--confirm-overwrite` não foi passado com confirmação explícita, **abortar sem alterar nada**.
3. Conferir `SHA-256` do arquivo extraído em `/tmp` contra `db_sha256`.
4. Executar `PRAGMA integrity_check` no arquivo em `/tmp` (exigindo retorno exato `"ok"`).
5. Conferir `alembic_revision`: se mais nova que o código da imagem atual, **abortar**; se igual, prosseguir; se mais antiga, validar contagens pré-migração e aplicar `alembic upgrade head` na cópia em `/tmp` antes da troca atômica.
6. Conferir `COUNT(*)` de todas as tabelas listadas em `table_counts` contra o manifesto.
7. Somente se os passos 1–6 passarem 100%, mover/copiar atomicamente para o volume `/data`.
