# Phase 0 Research: Feed Curator V1 — Curadoria diária de feeds

**Feature**: `001-feed-curation-v1` | **Date**: 2026-11-01
**Inputs**: `specs/001-feed-curation-v1/spec.md`, `docs/adr/0001-armazenamento-do-estado.md`, `docs/investigacoes/discovery-github-feed-curator.md`, `docs/requisitos-feed-curator-v3.md`

Este documento consolida as pesquisas técnicas da Fase 0 do planejamento, resolvendo todas as lacunas do contexto técnico (`NEEDS CLARIFICATION`), escolhas de bibliotecas, padrões de integração e restrições operacionais antes da modelagem de dados e contratos da Fase 1.

---

## R-01: Linguagem, Runtime Base e Compatibilidade Multi-Arquitetura

- **Decision**: Python 3.12 sobre imagem base `python:3.12-slim-bookworm` fixada por digest SHA-256 em build multi-stage (`Dockerfile`), com gerenciamento determinístico de lockfile (`uv.lock` / `requirements.lock` com hashes) e pacotes de sistema restritos a `ca-certificates`, `tzdata`, `restic` e `rclone` (sem compiladores na etapa final).
- **Rationale**:
  - Python 3.12 possui wheels pré-compilados estáveis para `linux/amd64` (arquitetura confirmada da VPS ARPHost: `x86_64`) e `linux/arm64` (SC-011) em todas as bibliotecas críticas (`lxml`/`trafilatura`, `pydantic-core`, `sqlalchemy`, `aiogram`, `httpx`).
  - O Debian Bookworm traz SQLite ≥ 3.40 na biblioteca padrão (`sqlite3`), garantindo suporte nativo a `VACUUM INTO`, API de backup online, modo `WAL` e `PRAGMA integrity_check`.
  - Mantém a imagem final abaixo do limite provisório de 300 MB descompactados (NFR-011, SC-014), mesmo incluindo `restic` e `rclone` na mesma imagem para o serviço separado de backup (NFR-012).
- **Alternatives considered**:
  - `python:3.13-slim`: descartada por menor maturidade de wheels binários multi-arch em dependências transitivas de extração HTML (`lxml`/Cython) no momento do travamento.
  - `python:3.12-alpine`: descartada porque `musl` frequentemente exige compilação local de extensões C (`lxml`, `pydantic-core`), aumentando risco de divergência multi-arch e lentidão de build.

---

## R-02: Armazenamento de Estado, Atomicidade e Bloqueio de Concorrência (ADR-0001)

- **Decision**: Adotar **SQLite em volume nomeado dedicado** (`WAL`, `synchronous=FULL`, `foreign_keys=ON`, `busy_timeout=5000`) como padrão provisório de implantação (Opção A do `docs/adr/0001-armazenamento-do-estado.md`), acessado exclusivamente via **SQLAlchemy 2.0 Core + Alembic** por `DATABASE_URL` neutro de dialeto (preservando saída limpa para PostgreSQL — Opção B1/B2 — condicionada ao spike de aceite de ~1 h do ADR-0001). O bloqueio contra processadores concorrentes (FR-032) usa uma tabela de lease atômica (`processor_leases`) combinada com transação `BEGIN IMMEDIATE` no SQLite (ou `UPDATE ... WHERE expires_at < now` transacional compatível com ambos os motores).
- **Rationale**:
  - Com ~9 publicações/dia (~3,2 mil entradas/ano, ~65–160 MB/ano) e usuário único na mesma VPS compartilhada com ~18 serviços de trading, o SQLite elimina um container permanente de banco, reduz consumo de RAM/disco na VPS (NFR-014) e atende à preferência explícita do usuário por zero serviço extra.
  - `BEGIN IMMEDIATE` no SQLite serializa escritores antes da leitura de saldo, garantindo reserva atômica de orçamento (FR-019, SC-004) e aquisição de lease sem corrida entre dois containers `run-once` disparados simultaneamente.
  - Manter SQLAlchemy Core puro (sem tipos exclusivos de Postgres como `JSONB`/`ARRAY` nem SQL cru específico fora da fábrica de engine) permite validar o spike obrigatório do ADR-0001 nos dois motores e migrar para Postgres via `DATABASE_URL` caso qualquer condição de aceite falhe.
- **Alternatives considered**:
  - PostgreSQL dedicado em container lateral (Opção B1): mantido como fallback documentado do ADR-0001 se o disco do volume na VPS não for local ou se o spike de `kill -9`/concorrência refutar o SQLite; rejeitado como padrão inicial pelo custo fixo de RAM/serviço extra na VPS compartilhada.
  - PostgreSQL compartilhado com o stack de trading (Opção C) ou suporte simultâneo a dois motores em produção (Opção D): eliminados pelo ADR-0001 (acoplamento de raio de dano/credenciais e duplicação permanente de matriz de testes).

---

## R-03: Provedor de Pontuação (`Scorer`), Saída Estruturada e Contabilização de Custo

- **Decision**: Implementar o protocolo `Scorer` (FR-015) com chamada HTTP direta via `httpx` (ou SDK mínimo oficial sem runtime de agentes) e validação estrita com **Pydantic v2** sobre saída JSON estruturada (`response_format` / JSON Schema estrito, `tools=[]` desabilitado), sem usar retries automáticos ocultos de bibliotecas intermediárias. O adaptador padrão de referência aponta para modelo pequeno de baixo custo configurável por ambiente (`LLM_PROVIDER`, `LLM_MODEL`, `LLM_API_KEY`, ex.: `claude-3-5-haiku` ou `gpt-4o-mini`), mantendo `SimpleRuleScorer` (determinístico, custo zero) como seletor ativo até eventual aprovação do LLM (FR-018, FR-022).
- **Rationale**:
  - Bibliotecas como `Instructor` executam retentativas automáticas de validação internamente; se não forem desfeitas e reescritas, essas retentativas consomem tokens sem passar antes pela reserva preventiva no ledger (violando FR-019, FR-021 e SC-004, conforme apontado em `docs/investigacoes/discovery-github-feed-curator.md` §5.3). Controlar cada tentativa explicitamente no loop do caso de uso garante `1 reserva atômica em micro-USD -> 1 chamada HTTP -> 1 liquidação ou reconciliação`.
  - A ~9 itens/dia (+ amostra de controle no modo sombra, total ≤ 15–20 chamadas/dia), um modelo pequeno consome < US$ 0,02/dia, deixando ampla margem dentro do orçamento confirmado pelo usuário de ~US$ 5,00/mês (≈ US$ 0,17/dia, FR-019).
  - Defesa contra prompt injection (FR-016, FR-037): nenhuma ferramenta/função é exposta ao modelo; o prompt separa instruções de sistema e conteúdo não confiável delimitado; o `entry_id` retornado no JSON é conferido contra o `entry_id` enviado; apenas texto da publicação, rubrica e exemplos aprovados saem para a API.
- **Alternatives considered**:
  - `Instructor` com retries internos habilitados: rejeitado pelo risco de cobrança não reservada previamente em retentativas de schema e por adicionar abstração desnecessária para um único schema Pydantic de 5 campos.
  - `Pydantic AI` / frameworks de agentes (`LangChain`, `LlamaIndex`): rejeitados por inflarem a imagem (risco ao teto de 300 MB da NFR-011) e introduzirem superfície desnecessária de execução de ferramentas (FR-016).

---

## R-04: Coleta HTTP, Parsing RSS/Atom, Extração de Artigo e OPML Seguro

- **Decision**:
  - **Fetch HTTP controlado**: `httpx` com timeouts explícitos (conexão 5 s, leitura 15 s), limite de redirecionamentos (máx. 5), validação estrita de esquema (`http`/`https`) e bloqueio de IPs privados/loopback/link-local (prevenção de SSRF na autodescoberta de feeds), persistindo `ETag` e `Last-Modified` por feed (FR-008) e backoff exponencial com pausa automática após `N` falhas consecutivas configuráveis (padrão `N=5`, FR-007).
  - **Parsing RSS/Atom**: `feedparser` operando exclusivamente sobre os bytes já baixados pelo `httpx` (nunca deixando o `feedparser` abrir conexões de rede próprias).
  - **Descoberta de feed (FR-003)**: inspetor leve próprio sobre `httpx` + parser HTML (`lxml.html` já presente via `trafilatura`) buscando `<link rel="alternate" type="application/rss+xml|application/atom+xml">` e sufixos canônicos (`/feed`, `/rss`, `/atom.xml`, `/index.xml`, `/feed.xml`), validando o candidato via `feedparser`.
  - **Extração de texto completo (FR-009)**: `trafilatura` (`favor_recall=True`, sem execução de JS); quando o texto extraído falhar, vier vazio ou ficar menor que o resumo do feed / limiar mínimo de cobertura, usar fallback para o trecho do feed, marcar `is_partial=True` e registrar `text_coverage` (`full_extracted`, `feed_summary_fallback` ou `title_only`).
  - **OPML (FR-001, FR-002)**: leitura com `defusedxml.ElementTree` (bloqueando DTD, entidades externas/XXE e expansão exponencial de entidades) e geração determinística com `xml.etree.ElementTree`, preservando categorias (`outline` pai) e garantindo idempotência por URL de feed normalizada.
- **Rationale**:
  - Reutiliza `lxml` já trazido por `trafilatura`, evitando instalar `feedsearch-crawler` e `listparser` como dependências extras com políticas próprias de rede ou XML não endurecido.
  - `defusedxml` elimina vulnerabilidades de XXE na importação de arquivos `.opml` enviados pelo Telegram (US6 Cenário 2), atendendo às diretrizes obrigatórias de segurança para parsing XML.
- **Alternatives considered**:
  - `feedsearch-crawler`: descartado para evitar crawler assíncrono irrestrito e dependências adicionais quando 1 página HTML + caminhos padrão resolvem blogs técnicos com controle estrito de SSRF e timeout.
  - `listparser`: descartado em favor de `defusedxml` (~50 linhas para OPML 1.0/2.0) para garantir endurecimento explícito contra XXE/DTD e round-trip exato na exportação.

---

## R-05: Regra Simples Determinística, Triagem Offline e Estatística do Modo Sombra

- **Decision**:
  - **Regra simples (`SimpleRuleScorer`)**: função determinística configurada em arquivo versionado (`config/rules/simple-rule-v1.yaml`) que pontua cada entrada combinando correspondência lexical normalizada (termos de alto sinal `+2`, termos de interesse `+1`, termos de ruído/marketing `-2` em título e texto) somada ao `prior_weight` do feed (padrão `1.0`, ajustável em `[-1.0, +2.0]`, FR-005), mapeada para nota inteira `1..5` (limiar de digest `≥ 4`), mantendo pontuação bruta e data de publicação como desempate determinístico.
  - **Amostragem da triagem offline (FR-017)**: gerador CLI (`triage build-sample`) seleciona `100..150` itens históricos/coletados (excluindo estritamente os 5–10 exemplos da rubrica) compostos pela união do topo da regra simples, topo do LLM e amostra aleatória uniforme de controle (~30% da amostra). Exporta planilha/JSONL cego (sem notas, sem origem da seleção e em ordem embaralhada com semente registrada) para rotulagem do usuário em três classes (`raro`, `relevante`, `irrelevante`).
  - **Cálculo e gates congelados (FR-018, SC-005)**: avaliador (`triage evaluate` / `shadow-report`) lê `config/evaluation/frozen-criteria.yaml`, confere que seu commit/timestamp e hash SHA-256 antecedem a primeira execução avaliada do LLM (bloqueando promoção se alterado a posteriori — US2 Cenário 4), exige `n_raros >= 10` (abaixo disso: veredito obrigatório `inconclusivo`, mantendo a regra e estendendo o modo sombra), calcula recall nos raros em contagem bruta (`x/n`) + **intervalo de confiança de Wilson (95%)**, precisão / taxa de itens ruins por 7 (teto informativo `≤ 2/7`) e projeção de custo mensal. Só recomenda `aprovar_llm` se `recall_raros_llm > recall_raros_regra` (empate mantém a regra) **e** `custo_mensal_projetado <= orcamento_mensal_usd` (US$ 5,00).
- **Rationale**:
  - Segue integralmente a deliberação 8/8 do council e as 4 respostas do usuário na spec: foco exclusivo em falsos negativos de itens raros como gate duro relativo à regra simples, sem percentuais arbitrários inventados, e proteção mecânica contra ajuste de meta após ver o resultado.
- **Alternatives considered**:
  - `promptfoo` como dependência do projeto: descartado para não introduzir runtime Node.js no workspace/CI quando o cálculo de recall bruto + intervalo de Wilson + comparação pareada contra `SimpleRuleScorer` leva < 100 linhas em Python puro testado com `pytest`.

---

## R-06: Interação Telegram sem Modo Contínuo (`run-once` + `sync-telegram`), Layout do Digest e Estado `incerto`

- **Decision**:
  - **Consumo de comandos e feedback sem daemon permanente (FR-034)**: usar `aiogram` v3 (cliente `Bot` + roteamento de updates em lote via `getUpdates(offset=last_update_id + 1, timeout=0)`) sem iniciar loop infinito `start_polling`. O offset `last_update_id` é persistido no banco somente após o commit do processamento de cada update. A drenagem roda obrigatoriamente como Etapa 1 do `run-once` diário **e** pode rodar em execuções curtas agendadas (`sync-telegram`, ex.: a cada 15–30 min via timer do host, < 2 s por execução, custo zero de LLM, zero portas abertas), garantindo resposta rápida no celular e evitando a expiração de 24 h da fila do Telegram.
  - **Layout do digest e ações inequívocas (FR-022..026, US3 Cenário 5)**: enviar o digest diário como lote ordenado de até `K` cartões de item (padrão `K <= 7`, apenas nota `≥ 4`, sem mínimo artificial) seguidos por 1 cartão-resumo de fechamento do digest com o botão global `[⚠️ Faltou algo hoje]`. O primeiro cartão notifica normalmente; os cartões seguintes do mesmo lote usam `disable_notification=True` (1 único alerta sonoro no celular). Cada cartão de item traz título, fonte, resumo curto em pt-BR + motivo (quando LLM aprovado; no modo regra simples, trecho curto/categoria), link original e teclado inline próprio `[👍 Útil] [👎 Não útil] [🔖 Guardar]` com `callback_data` curto (`fb:<delivery_id>:u|d|s`, ≤ 24 bytes, bem abaixo do limite de 64 bytes). Se um item exceder 3.800 caracteres, o resumo é truncado com reticências preservando integralmente título, link original e teclado (nunca estourando 4.096 caracteres).
  - **Protocolo de intenção durável e estado `incerto` (FR-025, FR-026, SC-003)**: para cada item selecionado, o sistema cria `Entrega` em `pendente`; imediatamente antes da chamada HTTP `sendMessage`, faz commit transacional mudando `state='incerto'` e `intent_recorded_at`. Se `sendMessage` retornar HTTP 200 com `message_id`, atualiza para `state='confirmado'` + `channel_message_id`. Se ocorrer timeout, queda de conexão, erro HTTP 5xx ou morte do processo (`kill -9`) após o commit de `incerto`, o registro permanece `incerto`, nunca sofre retry automático, aparece como incerto ao usuário e só pode ser reenviado por comando explícito (`/retry_uncertain <id>`).
- **Rationale**:
  - `1 cartão por item + disable_notification=True` resolve três riscos críticos identificados no discovery (`docs/investigacoes/discovery-github-feed-curator.md` §5.1 e §5.2): (1) elimina erro de toque em 21 botões numerados empilhados no fim de uma mensagem única; (2) evita estouro do limite de 4.096 caracteres do Telegram ao juntar 7 resumos + links; (3) dá atomicidade 1:1 entre `Entrega` e `sendMessage` (se a rede cair no item 4, os itens 1–3 já estão `confirmado` e apenas o item 4 fica `incerto`).
- **Alternatives considered**:
  - Mensagem única com 7 itens e 22 botões numerados (`1👍 1👎 1🔖`...): rejeitada pela alta fricção/erro de toque em tela mobile, fragilidade ao limite de 4.096 caracteres e impossibilidade de isolar falha parcial por item.
  - `python-telegram-bot`: rejeitado pela licença GPL-3.0 no tronco atual (apontada no discovery §2), preferindo `aiogram` v3 (MIT).

---

## R-07: Supervisor Externo e Canal Independente para o Alerta "Digest Não Enviado Hoje"

- **Decision**: Arquitetura de supervisão em **duas camadas independentes do pipeline de envio** (FR-033, SC-008):
  1. **Watchdog local independente no host (`systemd` timer + modo `watchdog-check` / script isolado)**: agendado no host da VPS para rodar após a janela esperada do digest (ex.: digest às `07:00 America/Sao_Paulo`, watchdog às `07:30`). Lê em modo somente-leitura o banco de estado (`cycle_runs` / `digests` para a data atual): se não houver registro do dia com `confirmed` nem `empty_day` justificado, dispara `"⚠️ Alerta: digest não enviado hoje (<YYYY-MM-DD>)"` usando cliente HTTP mínimo (`urllib`/`httpx` direto, **sem importar o pipeline de curadoria nem o dispatcher do bot**) através de credencial/canal separado (`ALERT_TELEGRAM_BOT_TOKEN` dedicado de monitoramento e/ou `ALERT_WEBHOOK_URL`).
  2. **Dead-man's-switch externo (`DEADMAN_PING_URL`, ex.: Healthchecks.io / Uptime Kuma externo)**: o `run-once` só faz um ping HTTPS de saída (`GET/POST`) ao fim de um ciclo diário concluído (`confirmed` ou `empty_day` válido). Se a VPS inteira desligar, o Docker travar, o disco encher ou a rede da VPS cair (morte total do processo/host), o serviço externo não recebe o ping até a tolerância diária e alerta o usuário por e-mail/Telegram externo — sem exigir nenhuma porta aberta na VPS (FR-036, NFR-014).
  - **Prova obrigatória pré-produção**: antes do primeiro digest real, executar o teste de falha deliberada bloqueando o envio do digest e comprovando o recebimento do alerta independente (FR-033, SC-008).
- **Rationale**:
  - Cumpre estritamente FR-033 e o Edge Case da spec (*"O alerta de 'digest não enviado' depende do mesmo processo que falhou: não é aceitável"*): uma falha de código, dependência, credencial do bot principal ou crash no `run-once` é capturada pelo `watchdog-check` com token separado, enquanto a morte total da VPS/container é capturada pelo dead-man's-switch externo sem violar a proibição de portas de entrada.
- **Alternatives considered**:
  - Heartbeat verificado dentro do próprio processo `run-once`: rejeitado por não disparar se o processo ou agendador não rodar.
  - Endpoint HTTP `/health` exposto na VPS para monitoramento externo: rejeitado por violar FR-036 e NFR-014 (*"MUST NOT abrir portas de entrada"* / *"MUST NOT publicar portas no host"*).

---

## R-08: Backup Isolado (`restic` + `rclone`), Restauração Guiada e Higiene/Entrega Direta da Imagem

- **Decision**:
  - **Serviço de backup isolado (FR-038, FR-043..046, NFR-012)**: a mesma imagem executa o subcomando `backup` em um serviço Compose separado (`feed-curator-backup`), que monta o volume do banco, `/tmp` em `tmpfs` e recebe apenas as variáveis `RESTIC_*`, `RCLONE_CONFIG_*` e `APP_ENV` (`dev` ou `prod`). O serviço principal (`feed-curator-app`) **não** recebe nenhuma variável do `restic`/`rclone`. O backup gera snapshot consistente via API online `sqlite3.Connection.backup()` em `/tmp`, calcula SHA-256, lê a revisão Alembic e contagens por tabela, grava `backup-manifest.json`, cifra no repositório `restic` próprio e sincroniza via `rclone` (configurado 100% por variáveis de ambiente e `RESTIC_CACHE_DIR=/tmp/restic-cache`, compatível com raiz somente leitura `read_only: true`).
  - **Restauração explícita e guiada (FR-044, FR-045)**: subcomando interativo/explícito `restore --snapshot <id|latest> [--confirm-overwrite]` executado manualmente (`docker compose run --rm feed-curator-backup restore ...`). Nunca roda na inicialização automática do app. Baixa snapshot e manifesto para `/tmp`, valida SHA-256 do arquivo, executa `PRAGMA integrity_check`, confere compatibilidade da revisão Alembic (revisão mais antiga aplica `alembic upgrade head` após validação; revisão mais nova aborta), confere contagens de todas as tabelas contra o manifesto, exibe claramente o `environment_label` (`dev`/`prod`) do backup e recusa sobrescrever banco já populado sem flag/confirmação explícita. Qualquer divergência aborta antes de substituir o arquivo no volume.
  - **Pipeline de imagem mínima e entrega direta à VPS (NFR-001..014, SC-013, SC-014)**:
    1. `.dockerignore` estrito em lista branca/exclusão explícita bloqueando `.git`, `.venv`, `.specify`, `.agents`, `.claude`, `.commandcode`, `docs/`, `specs/`, `tests/`, `*.db`, `*.sqlite*`, `.env*`, caches e backups.
    2. Build local multi-stage somente a partir de árvore Git limpa em commit publicado do tronco (`git status --porcelain` vazio), injetando `APP_VERSION`, `GIT_COMMIT` e `APP_UID`/`APP_GID` da VPS sem vazar caminhos, hostnames ou e-mails locais (NFR-006, NFR-009).
    3. Gates locais obrigatórios pós-build: (a) inspeção estática do tar exportado do filesystem da imagem (`docker create` + `docker export`) reprovando listagem vazia ou presença de qualquer item proibido (SC-013); (b) teste funcional com `--read-only --tmpfs /tmp --cap-drop ALL --security-opt no-new-privileges` (NFR-007); (c) medição de tamanho contra o teto provisório de 300 MB (NFR-011); (d) varredura `trivy image` registrada, bloqueando envio à VPS se houver CVE `CRITICAL` com correção disponível (`--ignore-unfixed` separado do relatório completo, NFR-010).
    4. Entrega direta sem registry (NFR-013, NFR-014, precedente SPEC-077): `docker image save | gzip` gerando pacote `.tar.gz` + `.sha256` + manifesto com o digest de configuração da imagem (`docker image inspect --format '{{.Id}}'`), verificação prévia de espaço livre em disco na VPS (abortando se margem pós-carga ficar abaixo do piso seguro da VPS compartilhada, ≥ 20 GiB livres), transferência por `scp`, conferência dupla (SHA-256 do arquivo + digest de configuração pós-`docker image load`), preservação da tag anterior para rollback imediato e Compose isolado (projeto, volumes, limites de CPU/RAM próprios, `network_mode` restrito a saída sem tocar rede/banco/volumes do trading e zero portas publicadas).
- **Rationale**:
  - Satisfaz simultaneamente todas as 14 NFRs de higiene/segurança/isolamento, viabiliza a migração do estado gerado nos testes em Docker no Windows/WSL para a VPS sem expor dados no Git ou na imagem (repositório público), e evita qualquer impacto nos ~18 serviços de trading da VPS compartilhada.
- **Alternatives considered**:
  - GitHub Container Registry (GHCR): descartado por decisão explícita do usuário (NFR-013).
  - Build da imagem diretamente na VPS (`git pull` + `docker build`): proibido pela NFR-013 por exigir código-fonte/ferramentas de build na VPS e consumir CPU/RAM/disco do host compartilhado com o trading.
  - Restore automático ao detectar volume vazio: proibido por FR-045 e US6 Cenário 6 pelo risco de mascarar falha de montagem de volume ou sobrescrever ambiente errado silenciosamente.
