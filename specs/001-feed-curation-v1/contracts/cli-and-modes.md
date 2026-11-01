# Contract: CLI, Container Modes & Environment Isolation

**Feature**: `001-feed-curation-v1` | **Date**: 2026-11-01
**Covers**: `FR-032..036`, `FR-038`, `FR-043..046`, `NFR-007`, `NFR-012..014`

Uma única imagem de container (`feed-curator:<version>`) expõe todos os modos operacionais via entrypoint CLI (`python -m feed_curator <command>`), sem abrir nenhuma porta de rede de entrada (`EXPOSE` ausente; zero `ports:` no Compose).

---

## 1. Subcomandos da CLI

| Subcomando | Serviço / Contexto | Efeitos Externos | Descrição e Garantias |
|---|---|---|---|
| `migrate` | `app` (pré-ciclo / setup) | Escrita apenas no volume SQLite/DB | Aplica migrações Alembic pendentes (`upgrade head`) de forma idempotente (FR-035). |
| `run-once [--date YYYY-MM-DD]` | `app` (timer diário) | Feeds HTTP, LLM (se sombra/aprovado e com reserva), Telegram | Adquire lease `pipeline_write` (FR-032), drena comandos/feedback pendentes do Telegram, coleta feeds ativos, deduplica, pontua respeitando reserva prévia de teto (FR-019), envia digest pela seleção ativa (`simple_rule` até aprovação do LLM) e aciona `DEADMAN_PING_URL` se confirmado ou dia vazio válido (FR-012, FR-034). |
| `dry-run [--with-paid-llm]` | `app` (validação / dev) | Coleta feeds (ou fixtures); **zero envio** ao Telegram; **zero gasto** por padrão | Simula o ciclo completo e imprime relatório estruturado em `stdout` sem enviar mensagens nem chamar API paga (salvo flag explícita `--with-paid-llm` sujeita ao teto diário) (FR-034). |
| `sync-telegram` | `app` (timer curto 15–30 min, opcional) | Apenas `getUpdates` / respostas a comandos autorizados | Consome reações e comandos do celular em < 2 s sem manter processo permanente (`serve` adiado — FR-034) e sem gasto de LLM. |
| `import-opml --file <path>` | `app` (bootstrap local) | Apenas banco | Importa feeds de um arquivo OPML local de forma idempotente, marcando o estado inicial para coleta de baseline (FR-001, FR-004). |
| `export-opml [--out <path>]` | `app` | `stdout` ou arquivo em `/tmp`/volume | Exporta todos os feeds ativos em OPML 2.0 válido (FR-002). |
| `triage build-sample` | `app` (avaliação offline) | LLM (com reserva de teto) + arquivo cego | Constrói amostra cega de 100–150 itens (união dos topos da regra e do LLM + controle aleatório, disjunta dos exemplos da rubrica) (FR-017). |
| `triage evaluate --labels <path>` | `app` (avaliação offline) | Somente leitura / registro local | Valida hash/data de `frozen-criteria.yaml` e emite relatório pareado LLM vs regra simples (recall bruto `x/n` nos raros + Wilson 95%, precisão, fadiga e custo projetado; `< 10` raros => `inconclusivo`) (FR-017, FR-018, SC-005). |
| `shadow-report` | `app` | Somente leitura | Emite métricas acumuladas do modo sombra (itens entregues, amostra de controle, marcações `"faltou algo"`, dias sem marcação e recomendação pelos critérios congelados) (FR-039..041). |
| `shadow-decide --verdict <aprovar_llm\|manter_regra\|estender_sombra> --by <name>` | `app` (ação humana explícita) | Banco (`evaluation_decisions`) | Registra decisão final auditável; bloqueia `aprovar_llm` se `frozen-criteria.yaml` for posterior às pontuações ou se critérios duros falharem (FR-018, FR-031, FR-042). |
| `watchdog-check [--date YYYY-MM-DD]` | `watchdog` (timer independente pós-horário do digest) | Leitura somente-leitura do DB + canal de alerta independente | Verifica se o digest do dia está `confirmed` ou `empty_day` sem importar o pipeline principal; caso contrário, emite `"digest não enviado hoje"` pelo token/webhook independente (FR-033, SC-008). |
| `backup` | `backup` (serviço separado) | Leitura consistente do DB + `restic` + `rclone` (Drive próprio) | Gera snapshot SQLite online em `/tmp`, cria `backup-manifest.json`, cifra com `restic` e sincroniza com a pasta dedicada do Google Drive (FR-038, FR-043). |
| `restore --snapshot <id\|latest> [--confirm-overwrite]` | `backup` (execução manual explícita) | `rclone` + `restic` + validação em `/tmp` + volume DB | Baixa backup cifrado, valida SHA-256, `PRAGMA integrity_check`, revisão Alembic e contagens por tabela contra o manifesto, exibe `environment_label` (`dev`/`prod`) e recusa banco populado sem `--confirm-overwrite`. **Nunca executa automaticamente** (FR-044, FR-045). |

---

## 2. Códigos de Saída (Exit Codes)

| Código | Significado |
|---|---|
| `0` | Execução concluída com sucesso (inclui dia vazio válido ou falhas isoladas de feeds individuais tratadas conforme FR-011). |
| `2` | Erro de validação de argumentos, configuração ausente ou segredo obrigatório não definido. |
| `3` | Lease de concorrência ocupado por outro processador ativo (FR-032 — saída limpa sem corromper estado). |
| `4` | Violação de gate de segurança/integridade (hash divergente no restore, banco populado sem confirmação, `frozen-criteria.yaml` violado). |
| `5` | Alerta operacional detectado pelo `watchdog-check` (digest do dia ausente ou `partial_uncertain` após disparo do alerta). |
| `1` | Falha fatal inesperada de infraestrutura (ex.: volume de estado inacessível ou corrompido). |

---

## 3. Isolamento de Variáveis de Ambiente por Serviço (FR-036, FR-043)

Nenhum segredo pode ter valor literal padrão (fallback hardcoded) no código-fonte. Na ausência de variável obrigatória para o modo invocado, o processo falha imediatamente (`fail-closed`, código `2`).

### 3.1 Serviço `feed-curator-app`

- `APP_ENV`: `dev` | `prod` (obrigatório)
- `TZ`: padrão `America/Sao_Paulo`
- `DATABASE_URL`: ex.: `sqlite:////data/feed_curator.db` (ou `postgresql+psycopg://...` se fallback ADR-0001)
- `TELEGRAM_BOT_TOKEN`: token do bot principal de entrega/interação (obrigatório fora de `dry-run`/`migrate`)
- `TELEGRAM_ALLOWED_USER_IDS`: lista separada por vírgula de IDs inteiros autorizados (obrigatório; ex.: `123456789`)
- `TELEGRAM_CHAT_ID`: chat privado autorizado de destino do digest
- `LLM_BASE_URL` / `LLM_MODEL` / `LLM_API_KEY`: endpoint HTTPS OpenAI-compatível (primeiro adaptador confirmado: API Z.AI `api.z.ai`, ex.: modelo `glm-5.3-flash`) e chave de API; obrigatórias (`fail-closed`, sem fallback hardcoded e mascaradas em qualquer erro/log) sempre que modo sombra, triagem com LLM ou seleção LLM aprovada estiverem habilitados
- `LLM_TIMEOUT_S`: timeout por requisição HTTP ao `Scorer` (padrão `60`, `max_retries=0`)
- `LLM_INPUT_PRICE_PER_1M_USD` / `LLM_OUTPUT_PRICE_PER_1M_USD`: preços unitários vigentes do modelo configurado (obrigatórios quando LLM habilitado, persistidos em micro-USD por tentativa em `paid_usage_ledger` — FR-019, FR-021)
- `LLM_MONTHLY_BUDGET_USD`: padrão `5.00` (deriva teto diário persistente em `America/Sao_Paulo`)
- `DIGEST_MAX_ITEMS`: padrão `7` (`1..15`)
- `DIGEST_MIN_SCORE`: padrão `4` (`1..5`)
- `MAX_FEED_FAILURES`: padrão `5`
- `DEADMAN_PING_URL`: URL HTTPS opcional de dead-man's-switch externo (chamada apenas após ciclo diário confirmado)
- **PROIBIDO neste serviço**: `RESTIC_PASSWORD`, `RESTIC_REPOSITORY`, `RCLONE_CONFIG_*`, `ALERT_TELEGRAM_BOT_TOKEN`.

### 3.2 Serviço `feed-curator-backup` (FR-043)

- `APP_ENV`: `dev` | `prod` (gravado no `backup-manifest.json`)
- `DATABASE_URL`: acesso ao arquivo do banco no volume compartilhado
- `RESTIC_REPOSITORY`: caminho do repositório cifrado dedicado do `feed-curator`
- `RESTIC_PASSWORD`: senha exclusiva do repositório `restic` do `feed-curator` (distinta do stack de trading)
- `RESTIC_CACHE_DIR`: `/tmp/restic-cache` (em `tmpfs`)
- `RCLONE_CONFIG_GDRIVE_TYPE` / `RCLONE_CONFIG_GDRIVE_TOKEN` / `RCLONE_CONFIG_GDRIVE_ROOT_FOLDER_ID`: credenciais exclusivas da pasta Google Drive própria do `feed-curator`
- **PROIBIDO neste serviço**: `LLM_API_KEY`, `TELEGRAM_BOT_TOKEN`.

### 3.3 Serviço / Timer `feed-curator-watchdog` (FR-033)

- `DATABASE_URL`: acesso somente-leitura (`?mode=ro` no SQLite) ao volume de estado
- `ALERT_TELEGRAM_BOT_TOKEN` + `ALERT_TELEGRAM_CHAT_ID` (ou `ALERT_WEBHOOK_URL`): canal independente do bot/processo principal

---

## 4. Contrato de Container e Isolamento na VPS (NFR-007, NFR-013, NFR-014)

- **Usuário**: UID/GID não-root parametrizado no build (`APP_UID`/`APP_GID` correspondendo ao usuário sem privilégios da VPS).
- **Filesystem**: `read_only: true` obrigatório; escrita permitida exclusivamente em `/data` (volume nomeado `feed_curator_data`) e `/tmp` (`tmpfs`, `noexec,nosuid,nodev`, limite `64m`).
- **Capacidades e Rede**: `cap_drop: [ALL]`, `security_opt: [no-new-privileges:true]`, rede bridge própria `feed_curator_egress` (somente saída HTTPS 443; zero portas publicadas; sem vínculo com redes ou volumes do projeto de trading), limites explícitos de recursos (ex.: `mem_limit: 256m`, `cpus: 0.50`).
