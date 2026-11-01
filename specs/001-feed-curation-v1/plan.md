# Implementation Plan: Feed Curator V1 — Curadoria diária de feeds

**Branch**: `001-feed-curation-v1` *(identificador da feature; branch única `main`)* | **Date**: 2026-11-01 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `specs/001-feed-curation-v1/spec.md`

**Note**: Este plano foi gerado pelo workflow `/speckit-plan` a partir da especificação da V1, do `docs/adr/0001-armazenamento-do-estado.md` e do discovery `docs/investigacoes/discovery-github-feed-curator.md`.

## Summary

Entregar um curador diário de feeds RSS/Atom para usuário único via Telegram, operando em modo batch sem portas de entrada (`run-once` + sincronização curta de comandos `sync-telegram`, sem daemon contínuo na V1). A solução garante linha de base `"já vistos"` na ativação de feeds, deduplicação exata por URL canônica e hash de conteúdo não vazio preservando múltiplas origens, controle atômico de orçamento de LLM (~US$ 5,00/mês ≈ US$ 0,17/dia em micro-dólares com reserva prévia), protocolo de entrega com intenção durável e estado `incerto` sem retry cego, e alerta `"digest não enviado hoje"` por watchdog/canal independente + dead-man's-switch externo. A seleção do digest permanece sob uma **regra simples determinística** até que o LLM supere o recall da regra nos itens raros em triagem offline cega e modo sombra (2–4 semanas) sob critérios congelados antes da execução. Todo o runtime (app, watchdog e backup cifrado `restic` + `rclone`) compartilha uma única imagem Docker mínima, não-root e com raiz somente leitura (`read_only: true`), construída localmente e transferida diretamente para a VPS compartilhada via `docker image save/load` sem registry.

## Technical Context

**Language/Version**: Python 3.12 (`python:3.12-slim-bookworm` fixado por digest SHA-256)

**Primary Dependencies**: `httpx` (fetch HTTP controlado com ETag/Last-Modified e filtro anti-SSRF), `feedparser` (parsing RSS/Atom sobre bytes), `trafilatura` (extração de artigo completo + `lxml.html` para autodescoberta de feed), `defusedxml` (importação OPML endurecida contra XXE), `aiogram` v3 (cliente Telegram Bot API via `getUpdates` em lote), `pydantic` v2 (validação estrita de saída do `Scorer` e configurações), `sqlalchemy` 2.0 Core + `alembic` (acesso a dados neutro de motor e migrações versionadas); binários `restic` e `rclone` na imagem para o serviço separado de backup

**Storage**: SQLite (`WAL`, `synchronous=FULL`, `foreign_keys=ON`) em volume Docker nomeado dedicado (`feed_curator_data`) via `DATABASE_URL` neutro de dialeto (padrão provisório `Opção A` de `docs/adr/0001-armazenamento-do-estado.md`, com fallback PostgreSQL `Opção B1/B2` preservado até o spike de aceite de ~1 h)

**Testing**: `pytest`, `pytest-asyncio`, `ruff` (lint + regras de segurança Bandit `S`), `mypy` (tipagem estrita), `trivy` (varredura obrigatória de CVEs na imagem) e scripts automatizados de inspeção de filesystem exportado (`SC-013`) e execução `--read-only` (`NFR-007`)

**Target Platform**: Container Linux (`linux/amd64` confirmado na VPS ARPHost compartilhada; imagem verificada em `linux/amd64` e `linux/arm64` conforme `SC-011`; desenvolvimento/testes locais em Docker Linux/WSL2)

**Project Type**: CLI / batch worker containerizado sem portas de entrada (`run-once`, `dry-run`, `sync-telegram`, `triage`, `watchdog-check`, `backup`, `restore`) com interface conversacional via Telegram Bot API

**Performance Goals**: Ciclo diário (~21 feeds, ~9 publicações/dia) concluído em < 2 minutos; leitura do digest em ≤ 5 minutos (`SC-001`, teto padrão de 7 itens); drenagem `sync-telegram` em < 2 segundos; restauração guiada e subida em ambiente novo em < 15 minutos (`SC-009`)

**Constraints**: Zero portas de rede abertas/publicadas (`FR-036`, `NFR-014`); filesystem raiz somente leitura (`read_only: true`) com escrita restrita ao volume `/data` e `/tmp` em `tmpfs` (`NFR-007`); execução não-root; orçamento mensal ≤ US$ 5,00 (teto diário ≈ US$ 0,17 em `America/Sao_Paulo`) com reserva preventiva atômica (`FR-019`); tamanho da imagem ≤ 300 MB provisório (`NFR-011`); bloqueio de deploy por CVE `CRITICAL` corrigível (`NFR-010`); entrega direta `docker image save/load` sem registry e sem build na VPS (`NFR-013`); isolamento total de rede, volumes, banco, recursos e segredos em relação ao stack de trading na mesma VPS (`NFR-014`); convenção do usuário de ≤ 300 linhas por arquivo Python

**Scale/Scope**: Usuário único; 21 feeds iniciais (~9 posts/dia, ~3,2 mil entradas/ano, ~65–160 MB/ano); 6 User Stories (`US1..US4` P1, `US5..US6` P2), 46 requisitos funcionais (`FR-001..046`), 14 requisitos não funcionais (`NFR-001..014`) e 14 critérios de sucesso (`SC-001..014`)

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

- **Constituição ativa**: `.specify/memory/constitution.md` (**v1.0.0**, ratificada em `2026-11-01`).
- **Avaliação de conformidade com os Princípios I–V e Seções Operacionais**:

| Princípio / Seção Constitucional | Avaliação Pré-Fase 0 | Reavaliação Pós-Fase 1 (Design) | Evidência nos Artefatos |
|---|---|---|---|
| **I. Curadoria Guiada por Evidência e Controle Humano (`FR-018`, `SC-005`, `SC-012`)** | PASS | PASS | `SimpleRuleScorer` é o seletor padrão; `aprovar_llm` exige `frozen-criteria.yaml` commitado antes das notas, recall nos raros estritamente superior e aprovação humana explícita ([data-model.md](data-model.md) §2.8, [scorer-and-evaluation.md](contracts/scorer-and-evaluation.md) §2). |
| **II. Proteção Preventiva de Custo, Estado Durável e Falha Segura (`FR-019..026`, `FR-032..033`)** | PASS | PASS | Reserva em micro-USD via `BEGIN IMMEDIATE` sem retries ocultos ([research.md](research.md) R-03); transição `pendente -> incerto` commitada antes de `sendMessage` e reenvio só via `/retry_uncertain` ([data-model.md](data-model.md) §4.1); lease `processor_leases` e `watchdog-check` independente ([cli-and-modes.md](contracts/cli-and-modes.md) §3.3). |
| **III. Fronteira de Confiança Zero, Privacidade e Privilégio Mínimo (`FR-016`, `FR-036..037`, `FR-043`, `NFR-007`)** | PASS | PASS | LLM sem ferramentas e com conferência de `entry_id`; `defusedxml` anti-XXE; bloqueio anti-SSRF; `html.escape`; segredos `fail-closed` segregados (`app` sem `RESTIC_*`/`RCLONE_*`); zero portas, não-root e `read_only: true` ([quickstart.md](quickstart.md) §3, [cli-and-modes.md](contracts/cli-and-modes.md) §3–§4). |
| **IV. Simplicidade Proporcional, Modularidade e Portabilidade de Estado (`ADR-0001`, `FR-035`, `FR-044..045`)** | PASS | PASS | Composição enxuta sem daemon permanente; `DATABASE_URL` neutro (SQLAlchemy 2.0 Core + Alembic); módulos particionados ≤ 300 linhas; exportação OPML, backup cifrado com manifesto e `restore` exclusivamente explícito ([data-model.md](data-model.md), [scorer-and-evaluation.md](contracts/scorer-and-evaluation.md) §3). |
| **V. Verificação Antes de Entrega e Higiene Reprodutível de Artefatos (`NFR-001..012`, `SC-013..014`)** | PASS | PASS | Spike obrigatório de concorrência/`SIGKILL`/restore (`test_storage_spike.py`), falha deliberada do alerta independente, multi-stage build em commit limpo, inspeção do tar exportado, Trivy bloqueante para `CRITICAL` corrigível e mocks locais restritos a `127.0.0.1` ([quickstart.md](quickstart.md) §2–§3). |
| **Restrições Operacionais & Fluxo de Gates (`NFR-013..014`)** | PASS | PASS | Projeto Compose, rede, volumes e limites isolados do stack de trading na VPS; entrega direta `save/load` com checagem de disco livre e dupla conferência (SHA-256 + config ID); zero dados/segredos no Git ([research.md](research.md) R-08). |

## Project Structure

### Documentation (this feature)

```text
specs/001-feed-curation-v1/
├── plan.md              # Este arquivo (saída de /speckit-plan)
├── research.md          # Fase 0: decisões técnicas e alternativas (/speckit-plan)
├── data-model.md        # Fase 1: esquema neutro, entidades e máquinas de estado (/speckit-plan)
├── quickstart.md        # Fase 1: guia de validação ponta a ponta e segurança (/speckit-plan)
├── contracts/           # Fase 1: contratos externos e de fronteira (/speckit-plan)
│   ├── cli-and-modes.md
│   ├── telegram-bot.md
│   └── scorer-and-evaluation.md
└── tasks.md             # Fase 2: lista ordenada de tarefas (/speckit-tasks — NÃO criado aqui)
```

### Source Code (repository root)

```text
config/
├── evaluation/
│   ├── frozen-criteria.yaml       # Critérios congelados antes da avaliação do LLM (FR-018)
│   └── rare-examples-v1.yaml      # 5-10 exemplos reais aprovados pelo usuário (fora da amostra cega)
├── rubrics/
│   └── rubric-v1.yaml             # Rubrica versionada em pt-BR + exemplos sanitizados (FR-014)
├── rules/
│   └── simple-rule-v1.yaml        # Palavras-chave e faixas da regra simples determinística
└── seeds/
    └── initial-feeds.opml         # Semente inicial dos 21 feeds curados (NFR-002)

alembic/
├── env.py
└── versions/
    └── 0001_initial_v1.py         # Esquema inicial neutro SQLite/Postgres (FR-035)

src/feed_curator/
├── __init__.py
├── __main__.py                    # Entrypoint CLI (python -m feed_curator)
├── config.py                      # Settings Pydantic v2 fail-closed por modo (FR-036, FR-043)
├── clock.py                       # Chaves diárias/mensais em America/Sao_Paulo
├── cli/
│   ├── app_commands.py            # migrate, run-once, dry-run, sync-telegram, import/export-opml
│   ├── eval_commands.py           # triage, shadow-report, shadow-decide
│   ├── backup_commands.py         # backup, restore guiado (FR-044, FR-045)
│   └── watchdog_command.py        # watchdog-check isolado sem importar pipeline/aiogram (FR-033)
├── db/
│   ├── engine.py                  # Conexão DATABASE_URL, PRAGMAs WAL/FULL e BEGIN IMMEDIATE
│   ├── schema.py                  # Tabelas SQLAlchemy 2.0 Core neutras de dialeto
│   ├── leases.py                  # Exclusão mútua processor_leases (FR-032)
│   └── repositories/
│       ├── feeds_repo.py
│       ├── entries_repo.py
│       ├── scores_repo.py
│       ├── deliveries_repo.py
│       ├── feedback_repo.py
│       ├── budget_repo.py
│       └── evaluation_repo.py
├── ingestion/
│   ├── url_safety.py              # Canonicalização de URL, limpeza utm_* e filtro anti-SSRF
│   ├── http_fetcher.py            # GET condicional ETag/Last-Modified e backoff (FR-008)
│   ├── opml.py                    # Importação defusedxml idempotente e exportação OPML 2.0
│   ├── discovery.py               # Autodescoberta de feed a partir de URL de site (FR-003)
│   ├── parser.py                  # Wrapper feedparser + hash normalizado não vazio (FR-010)
│   └── extractor.py               # Extração trafilatura + fallback parcial e cobertura (FR-009)
├── scoring/
│   ├── contracts.py               # Protocolo Scorer + ScoringInput/ScoringOutput Pydantic
│   ├── simple_rule.py             # SimpleRuleScorer determinístico (palavras-chave + peso)
│   ├── llm_provider.py            # OpenAICompatibleScorer (API Z.AI /chat/completions, max_retries=0, sem tools, ID check)
│   └── budget_guard.py            # Reserva preventiva em micro-USD e reconciliação (FR-019..021)
├── evaluation/
│   ├── wilson.py                  # Contagem bruta + intervalo de confiança de Wilson (95%)
│   ├── frozen_gate.py             # Verificação SHA-256 e precedência temporal Git (FR-018)
│   ├── triage.py                  # Geração de amostra cega 100-150 itens e relatório pareado
│   └── shadow.py                  # Amostra de controle, aviso de 7 dias e registro de decisão
├── delivery/
│   ├── formatter.py               # Escape HTML, truncamento <= 3800/4096 chars e teclados inline
│   ├── dispatcher.py              # Protocolo pendente -> incerto -> confirmado (FR-025, FR-026)
│   ├── auth_filter.py             # Allowlist TELEGRAM_ALLOWED_USER_IDS com silêncio total (FR-027)
│   ├── update_drainer.py          # Consumo getUpdates em lote com cursor persistente
│   └── handlers/
│       ├── callbacks.py           # fb:<id>:u|d|s e dg:<date>:m (guardar != util)
│       └── commands.py            # /addfeed /feeds /disable /remove /weight /import /export /cost /retry_uncertain
├── pipeline/
│   └── daily_cycle.py             # Orquestração retomável por etapas e ping dead-man's-switch
└── backup/
    ├── sqlite_snapshot.py         # Backup online consistente em /tmp + PRAGMA integrity_check
    ├── manifest.py                # Geração e auditoria de backup-manifest.json
    └── transport.py               # Subprocessos seguros shell=False para restic e rclone

deploy/
├── Dockerfile                     # Multi-stage slim-bookworm, não-root, sem dev/test/docs
├── .dockerignore                  # Lista estrita de exclusão do contexto de build (NFR-005)
├── docker-compose.yml             # Serviços isolados app, backup e watchdog (read_only, sem portas)
└── systemd/                       # Unidades/timers de referência na VPS (run-once, sync, watchdog, backup)

scripts/
├── build-and-verify-image.sh      # Árvore limpa, inspeção tar (SC-013), tamanho, read-only e Trivy
└── export-and-ship-image.sh       # Checagem de disco na VPS, save/scp/load e dupla checagem de hash/config ID

tests/
├── unit/
├── contract/
└── integration/
    └── test_storage_spike.py      # Spike obrigatório do ADR-0001 (1000 reservas, kill -9, restore)
```

**Structure Decision**: Projeto Python único (`src/feed_curator`) empacotado em uma única imagem multi-modo, dividido em pacotes pequenos (`db`, `ingestion`, `scoring`, `evaluation`, `delivery`, `pipeline`, `backup`, `cli`) para manter todos os arquivos abaixo da convenção de 300 linhas e isolar o `watchdog_command.py` de qualquer importação do pipeline principal ou do `aiogram` (`FR-033`). Os diretórios `tests/`, `docs/`, `specs/`, `.specify/`, `.agents/`, `.claude/` e `.commandcode/` permanecem fora do contexto de build via `.dockerignore` (`NFR-001..005`, `SC-013`), entrando na imagem apenas `src/`, `alembic/` e `config/`.

## Verification & Security Gates

O plano completo de validação funcional e de segurança encontra-se em [quickstart.md](quickstart.md) (§2 e §3) e exige aprovação sequencial nos seguintes gates antes da operação real:

1. **Gate de Armazenamento (`ADR-0001`)**: aprovação do spike automatizado (`test_storage_spike.py`) comprovando zero estouro em 1.000 reservas concorrentes, consistência sob `SIGKILL` na gravação de intenção de envio e restauração íntegra de backup, além da confirmação de disco local do volume na VPS.
2. **Gate de Segurança de Código e Dados ([quickstart.md](quickstart.md) §3)**: validação de URLs com bloqueio anti-SSRF; parsing OPML exclusivo via `defusedxml` (anti-XXE) com limite de 1 MB; escapamento `html.escape(..., quote=True)` em todas as mensagens Telegram; queries 100% parametrizadas em SQLAlchemy Core; zero segredos ou fallbacks literais no código; isolamento estrito de variáveis entre `app`, `backup` e `watchdog`; execução de `restic`/`rclone` com `shell=False` e argumentos validados por allowlist; mocks HTTP de teste restritos a `127.0.0.1` (nunca `0.0.0.0`). Controles exclusivos de aplicações web com navegador/cookies/senhas (CSP, CSRF, CORS, Argon2, OAuth/MFA próprios, antivírus de anexos binários) estão formalmente justificados em [quickstart.md](quickstart.md) §3 como não aplicáveis à arquitetura sem portas e sem interface web da V1.
3. **Gate de Higiene e Segurança da Imagem (`NFR-001..014`, `SC-011`, `SC-013`, `SC-014`)**: build apenas em commit limpo publicado; `0` arquivos proibidos ou metadados locais no filesystem exportado; teste de execução com `--read-only` e `--cap-drop ALL`; medição de tamanho (`<= 300 MB` provisório); varredura `trivy image` sem CVE `CRITICAL` corrigível; verificação em `amd64`/`arm64`; checagem prévia de disco livre na VPS e dupla conferência (SHA-256 do `.tar.gz` + digest de configuração da imagem) no `docker image load`.
4. **Gate Operacional Pré-Go-Live (`FR-033`, `SC-008`)**: derrubada deliberada do envio do digest comprovando chegada do alerta `"digest não enviado hoje"` pelo canal independente antes da ativação do primeiro digest real.

## Complexity Tracking

> **Nenhuma violação constitucional detectada.** A tabela abaixo registra apenas a justificativa das separações operacionais exigidas diretamente pelos requisitos da especificação.

| Elemento Arquitetural | Por que é necessário | Alternativa mais simples descartada porque |
|---|---|---|
| 3 invocações/serviços Compose (`app`, `backup`, `watchdog`) a partir da mesma imagem | Exigido por `FR-033` (alerta independente do processo de envio), `FR-043` (app proibido de receber credenciais do destino de backup) e `NFR-012` (mesma imagem sem credenciais embutidas). | Rodar backup e alerta dentro do processo `app` violaria `FR-033` e `FR-043`, expondo credenciais do Drive ao parser de feeds não confiáveis e silenciando o alerta caso o `app` falhe. |
| Acesso neutro SQLAlchemy Core + Alembic em vez de `sqlite3` direto | Exigido por `FR-035` (migrações versionadas) e `ADR-0001` D-1 (manter saída barata para PostgreSQL via `DATABASE_URL` até o fechamento das condições de aceite). | SQL cru em `sqlite3` acoplaria dialeto e transações fora da camada de banco, encarecendo fallback para Postgres caso o disco na VPS ou o spike exijam `Opção B1/B2`. |
