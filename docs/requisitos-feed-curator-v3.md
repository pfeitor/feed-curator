# Feed Curator — Requisitos v3 (rascunho pós-council)

> **Status:** rascunho em revisão; requisitos finais e solução ainda **não aprovados**.
> **Data:** 2026-11-01 · **Versão:** 3.0-draft.
> **Origem:** consolidação da [v2](requisitos-feed-curator-v2.md) à luz do veredito do council de 2026-11-01 (9 membros: lao-tzu, torvalds, elon, aristotle, ada, feynman, socrates, sun-tzu, aurelius), do [discovery GitHub](investigacoes/discovery-github-feed-curator.md) e do [prompt de avaliação](prompt-avaliacao-requisitos-feed-curator.md).
> **Histórico:** [v1](requisitos-feed-curator.md) e v2 permanecem preservadas. Criar esta v3 não aprova arquitetura, fornecedores, metas ou escopo de implementação.

## 0. Como interpretar esta versão

- **Veredito incorporado:** discovery **aprovado** como hipótese-guia; rascunho de requisitos **reprovado** como contrato. Esta v3 aplica a dieta e as correções exigidas pelo council, mas continua rascunho.
- **Necessidades:** reduzir ruído, preservar o raro, agrupar cobertura repetida, ler e dar feedback sem fricção.
- **Requisitos candidatos:** seções 2 a 6 registram comportamento desejado. M/S/C é prioridade proposta, não aprovação.
- **Base mantida para comparação:** Python, Postgres, Telegram por long polling, imagem única, LLM sem ferramentas com JSON validado. Postgres mantido pela maioria do council; minoria SQLite registrada em §10.1 sem reabrir decisão por purismo.
- **Sem autorização implícita:** nenhuma redação, proposta ou ausência de resposta autoriza implementar, chamar LLM pago, escrever em banco ou executar spikes. Segue a governança vigente.
- **Fechamento:** somente após cumprir a seção 19 esta v3 poderá originar SPEC. IDs OBJ/CEN/RF/RNF/DEC/CA/Q/RISCO da v1/v2 preservados. Questões novas começam em Q-35.

---

## 1. Veredito consolidado do council

### 1.1 Consenso

**Discovery aprovado (caminho B: composição). Rascunho reprovado como contrato — não vai para `/speckit.specify` sem as correções e os experimentos desta v3.**
Compor com `feedparser + Trafilatura + aiogram + Pydantic`, Postgres próprio, Telegram long polling, LLM sem tools com JSON validado. Nenhum fork integral: WHOOPsClaw só referência de agrupamento, amb007 só UX, alxyrgin só handlers.

### 1.2 Pontos de acordo (9/9 ou ampla maioria)

1. Linha de base §1.1 da v1 é boa; problema é ruído nos alta-frequência + duplicidade temática, não volume bruto.
2. RF-027 "nunca duas vezes" é falso como escrito — `sendMessage` não tem chave de idempotência. Exigir at-least-once + reconciliação e estado `incerto`.
3. Alerta vs digest indefinido; teto sem retries/prévia/cluster é furado; `serve` vs `run-once` incompatível; heartbeat não vigia morte total; DB vs repo em conflito; RNF-019 vs few-shot em conflito.
4. Proteções (custo, idempotência, lock, baseline, regressão, backup) antes do primeiro alerta real.
5. MVP valida curadoria primeiro: ingestão + dedup exata + scoring + digest + feedback mínimo, batch-first.

### 1.3 Desacordos

- **Postgres vs SQLite:** maioria mantém Postgres já decidido + contratos tipados; minoria (elon, lao-tzu) prefere SQLite/cron mínimo para validar valor antes da infra.
- **Ordem do corte:** codar V1 mínima já (elon/torvalds) vs experimento sem código + rubrica antes de codar (lao-tzu/feynman/socrates).
- **Tamanho do corte:** ~1/3 (torvalds) vs ~70% (elon).

### 1.4 Minority report

V1 de fim de semana — cron 1x/dia, SQLite, 1 chamada LLM em lote, digest único, sem cluster/alerta/comandos. Papel não filtra ruído; só produto em produção itera. O conselho incorpora a dieta como direção, mas rejeita trocar Postgres/Pydantic/aiogram por decreto sem medir.

---

## 2. Objetivos (revisados)

| ID | Objetivo | Medida de sucesso |
|---|---|---|
| OBJ-01 | Reduzir o volume diário de leitura, com teto configurável e sem preencher vagas com irrelevante | Eventos únicos entregues/dia ≤ teto global (Q-06/Q-23); sem mínimo obrigatório |
| OBJ-02 | Minimizar a perda de notícias importantes | Recall/precisão no conjunto de avaliação, com metas aprovadas em Q-28; recall nos raros domina precisão |
| OBJ-03 | Eliminar repetição do mesmo evento | Um evento = um item, demais fontes listadas; falsos merges/separações medidos em Q-28 |
| OBJ-04 | Entregar e operar em mobile, web e desktop sem fricção de rede | Condicional a Q-21/Q-22: Telegram cobre transporte; riqueza de exploração (busca/histórico/reavaliação) só com front |
| OBJ-05 | Refinar a curadoria a partir de feedback explícito | Comparar rubricas no mesmo conjunto de avaliação; acompanhar feedback sem prometer crescimento de 👍 |
| OBJ-06 | Operar de forma portátil e barata | Uma imagem que roda no host aprovado em Q-10 com Postgres |
| OBJ-07 | Manter custo de LLM previsível | Nenhuma tentativa começa sem reserva suficiente; moeda/preços/teto/virada em Q-07/Q-27 |

---

## 3. Escopo

### 3.1 V1 mínima recomendada (valida curadoria antes de sofistichar)

- Ingestão RSS/Atom com GET condicional, baseline "já vistos", texto parcial sinalizado.
- Dedup exata por URL canonicalizada + hash; similaridade e cluster só com limiar e avaliação (Q-25/Q-28).
- Scoring 1–5 por LLM com rubrica versionada, motivo 1 linha + resumo pt-BR.
- Entrega: digest diário (+ alerta só após recall provado e Q-06/Q-12/Q-23 fechados).
- Feedback 👍👎🔖 vinculado a item/nota/rubrica; comandos essenciais.
- Idempotência, lock, ledger com reserva, heartbeat consultável, backup.
- Imagem única sem portas, long polling.

### 3.2 Adiar para evolução por evidência (pós-uso real 2–3 semanas)

Prévia pontuada completa (RF-004), probation automática (RF-007), mute (RF-009), peso a priori influente (RF-019), resumo longo default (RF-022), `/buscar` em descartados (RF-031), `/stats` rico (RF-032), few-shot automático (RF-034 só manual aprovado), front Tailscale (Fase 6).

### 3.3 Exclusões (mantidas da v2)

Front web nesta versão, multi-usuário, reenvio de íntegra, fine-tuning, fontes não-RSS/Atom, tradução integral.

---

## 4. Usuário e cenários

Usuário único: desenvolvedor brasileiro, lê no Android (Feeder), quer mobile/web/desktop. Aceita VPS/Docker/Postgres, tem API de LLM.

| ID | Cenário |
|---|---|
| CEN-01 | Alerta de nota máxima com resumo, motivo e link — condicional a Q-06/Q-12/Q-23 |
| CEN-02 | Digest matinal curto com os mais relevantes do dia anterior |
| CEN-03 | Mesmo evento em um item, demais fontes listadas |
| CEN-04 | Prévia de feed novo com cobertura observada antes de ativar — condicional a Q-27 |
| CEN-05 | 👍👎 reflete gosto via rubrica versionada aprovada |
| CEN-06 | 🔖 guarda para depois, sem influenciar ranking (Q-30) |
| CEN-07 | "Mais detalhes" sob demanda — condicional a Q-26/Q-27 |
| CEN-08 | Feed quebrado/ruidoso gera aviso, não silêncio |
| CEN-09 | Migração levando banco + env |
| CEN-10 | Import/export OPML via Telegram |
| CEN-11 | Mesmo estado em qualquer dispositivo via Telegram — riqueza de exploração condicional a Q-21 |

---

## 5. Requisitos funcionais

M = deve, S = deveria, C = pode. Itens ligados a Q abertas são condicionais. "Item" = cluster ou entrada sem cluster, conforme Q-25.

### 5.1 Gestão de feeds

| ID | Requisito | Prio. |
|---|---|:--:|
| RF-001 | Importar OPML de forma idempotente | M |
| RF-002 | Exportar feeds ativos em OPML | M |
| RF-003 | Adicionar feed por URL do feed ou do site, com autodescoberta | M |
| RF-004 | Prévia com cobertura temporal observada, até 10 posts pontuados e nota média; declarar contagem parcial/estimada; exigir confirmação; toda pontuação consome orçamento RF-021 | S |
| RF-005 | Ao ativar, registrar existentes como "já vistos", sem enviar | M |
| RF-006 | Desativar, remover e ajustar peso a priori | M |
| RF-007 | Probation configurável, só digest, com sugestão de desativar — **adiado da V1** | S |
| RF-008 | Listar feeds com volume/janela/cobertura, nota média, amostra e estado | M |
| RF-009 | Silenciar temporariamente — **adiado da V1** | S |
| RF-010 | Pausar após N falhas consecutivas e avisar | M |

### 5.2 Ingestão e deduplicação

| ID | Requisito | Prio. |
|---|---|:--:|
| RF-011 | GET condicional + backoff | M |
| RF-012 | Tentar texto completo; se indisponível, aplicar Q-14, marcar "parcial" e registrar cobertura usada | M |
| RF-013 | Identidade por URL canonicalizada + hash normalizado; preservar fontes; vazio/indisponível não prova igualdade | M |
| RF-014 | Similaridade de título gera candidatos; limiar avaliado; similaridade ≠ identidade de evento — **V1 só se Q-25/Q-28 aprovarem** | M |
| RF-015 | Cluster de mesmo evento em item único com fontes; IDs estáveis; sem apagar segunda fonte; falsos merges medidos — **V1 só se Q-25/Q-28 aprovarem** | M |
| RF-016 | Retomada por etapas persistentes idempotentes; efeitos confirmados não repetem; externa interrompida segue Q-24 | M |

### 5.3 Pontuação

| ID | Requisito | Prio. |
|---|---|:--:|
| RF-017 | Pontuar 1–5 com LLM: nota, categoria, motivo 1 linha, resumo curto pt-BR, JSON validado por schema | M |
| RF-018 | Rubrica versionada no repo; cada pontuação registra versão + modelo | M |
| RF-019 | Peso a priori por fonte — **adiado como influente na V1** | S |
| RF-020 | Provedor LLM atrás de interface trocável | M |
| RF-021 | Teto diário persistente com reserva de custo máximo antes de cada tentativa (scoring, clusters, prévias, resumos, retries, comandos). Sem saldo, reter/recusar e avisar (Q-07/Q-27/Q-31) | M |
| RF-022 | Resumo longo ~800 chars sob demanda — **adiado como default; condicional a Q-26/Q-27** | S |

### 5.4 Entrega e interação (Telegram)

| ID | Requisito | Prio. |
|---|---|:--:|
| RF-023 | Alerta de nota máxima respeitando latência, probation, silenciamento e teto global — **só após Q-06/Q-12/Q-23 e recall provado** | M |
| RF-024 | Digest em horário/fuso configuráveis, até o teto, sem mínimo; tratar dias vazios e reaparição de alertas (Q-05/Q-06/Q-23); alertas confirmados omitidos do digest automático por padrão | M |
| RF-025 | Todo item com link original; íntegra não reenviada | M |
| RF-026 | Ações inequívocas por item 👍👎🔖"Mais detalhes"; layout único vs múltiplo e expansão em Q-26; respeitar 4096 chars e `callback_data` 1–64B com referência curta | M |
| RF-027 | Não reenviar entregas confirmadas; persistir item/destino/modalidade/estado (`pendente`/`confirmado`/`incerto`). Incerto não recebe retry cego; recuperação em Q-23/Q-24/Q-25 | M |
| RF-028 | Só IDs autorizados; demais ignorados sem resposta | M |
| RF-029 | Comandos candidatos `/addfeed /feeds /disable /remove /weight /digest /mute /stats /rubrica /import /export /buscar`; recorte final em Q-29 | M |
| RF-030 | `/import` aceita `.opml`; `/export` devolve anexo | M |
| RF-031 | `/buscar` no arquivo inclusive descartados — **adiado da V1** | S |
| RF-032 | `/stats` custo/itens/👍👎 — versão rica **adiada**; mínimo de custo em Q-27 | S |

### 5.5 Aprendizado

| ID | Requisito | Prio. |
|---|---|:--:|
| RF-033 | Registrar feedback vinculado a item/nota/rubrica; voto substituível e auditável; 🔖 ≠ 👍 (Q-30) | M |
| RF-034 | Few-shot só com proposta, aprovação, revisão de dados e versionamento; sem alteração automática em produção — **manual na V1** | S |
| RF-035 | ~50 posts rotulados + casos de agrupamento; regressão de rubrica/modelo em conjunto de avaliação separado do few-shot (Q-04/Q-28) | M |

### 5.6 Operação

| ID | Requisito | Prio. |
|---|---|:--:|
| RF-036 | Mesma imagem com `serve / run-once / migrate / dry-run / healthcheck`. `serve` integra bot + ciclos; `run-once` encerra sem consumir comandos; `dry-run` sem envio e sem custo por padrão (Q-32) | M |
| RF-037 | Coordenação no Postgres contra processadores/consumidores incompatíveis; escopo de locks em Q-32 | M |
| RF-038 | Registrar ciclo, entrega esperada e falhas; avisar quando operacional. Morte total exige supervisor externo (Q-33) | M |
| RF-039 | Migrações versionadas e repetíveis | M |

---

## 6. Requisitos não funcionais

| ID | Categoria | Requisito |
|---|---|---|
| RNF-001 | Portabilidade | Imagem amd64/arm64 verificada em execução; host/recursos em Q-10; sem promessa sem teste |
| RNF-002 | Portabilidade | Postgres via `DATABASE_URL`; `docker-compose.yml` para uso rápido |
| RNF-003 | Segurança | Sem porta exposta; só saída (Telegram, LLM, feeds, Postgres) |
| RNF-004 | Segurança | Segredos só via env |
| RNF-005 | Segurança | Allowlist `TELEGRAM_ALLOWED_USER_IDS` |
| RNF-006 | Segurança | Feeds não confiáveis; LLM sem ferramentas, saída validada, IDs conferidos; schema ≠ fidelidade |
| RNF-007 | Segurança | Usuário não-root |
| RNF-008 | Confiabilidade | Reinício/retry preserva estado; incerto explícito; sem promessa de exactly-once (Q-24) |
| RNF-009 | Confiabilidade | Backoff só para recuperáveis, respeitando taxa/orçamento/envio incerto |
| RNF-010 | Confiabilidade | Falha de um feed/chamada não para o ciclo |
| RNF-011 | Custo | Teto preventivo em todas as tentativas; cobrança estimada vs faturada em Q-27 |
| RNF-012 | Custo | Modelo pequeno por padrão; maior só com medição |
| RNF-013 | Desempenho | Latência post→alerta em Q-12 |
| RNF-014 | Observabilidade | Logs estruturados; heartbeat consultável; falhas por feed |
| RNF-015 | Observabilidade | Ledger Postgres: operação, tentativas, tokens estimados/conhecidos, preços, reservas, custo; interrompidas = pendentes de reconciliação |
| RNF-016 | Qualidade | Integração contra Postgres real; fixtures; rotulados como regressão |
| RNF-017 | Qualidade | Rubrica + OPML inicial versionados; banco = feeds em operação; exportação + histórico auditável; sem commit automático por comando (Q-34) |
| RNF-018 | Manutenibilidade | Arquivos Python ≤300 linhas (convenção do usuário, não garantia funcional) |
| RNF-019 | Privacidade | Dados no banco do usuário; ao LLM só conteúdo + rubrica + exemplos aprovados; sem IDs Telegram/segredos/histórico (Q-18/Q-30) |
| RNF-020 | Usabilidade | pt-BR correto; legível em celular |
| RNF-022 | Multiplataforma | Leitura/feedback/gestão nos dispositivos de Q-21/Q-22, estado único no servidor; Telegram = proposta V1 |
| RNF-023 | Multiplataforma | V1 sem VPN/app próprio/porta de entrada; front próprio reabre RNF-003/DEC-04 |
| RNF-021 | Portabilidade de dados | Exportação OPML a qualquer momento |

---

## 7. Modelo de dados (conceitual)

| Tabela | Papel |
|---|---|
| `feeds` | URL, `ETag`, `Last-Modified`, estado, falhas, peso, categoria, ativação |
| `entries` | URL canônica única, hash, título, texto, data, origem |
| `clusters` | Evento, membros, identidade estável |
| `scores` | Nota, categoria, motivo, resumos, rubrica/modelo, JSON bruto |
| `deliveries` | Item/destino/modalidade/estado, `message_id`, data |
| `feedback` | Reação vinculada a item/nota/rubrica |
| `llm_usage` | Tokens/custo/reservas por chamada e dia |
| `heartbeat` | Último ciclo, para `healthcheck` |

Candidata `pg_trgm` só com limiar avaliado. Schema deve suportar fontes múltiplas sem apagar entradas, etapas, versões, reservas e estados `pendente`/`confirmado`/`incerto`. `UNIQUE` ≠ atomicidade com Telegram.

---

## 8. Deduplicação e limites

1. **Exata:** URL + hash; preserva autoria.
2. **Quase igual:** léxica gera candidatos; exige limiar + janela + avaliação.
3. **Evento:** preserva fontes; separa tema ≠ notícia; LLM sem embeddings é direção (DEC-05), algoritmo/janela em Q-25.

Aprovação exige fixtures: duplicatas exatas, títulos parecidos de eventos diferentes, títulos distintos do mesmo evento.

---

## 9. Telegram × front-end

Mantida comparação da v2. **Telegram interface única da V1** por long polling. Front somente leitura via `tailscale serve` é fase opcional pós 2–3 semanas. Se "web" em OBJ-04 significar navegação rica, front vira essencial e reabre RNF-003/DEC-04 (Q-01/Q-21).

Formato: alerta (nota 5) com fontes/"também cobriram", resumo 2–3 linhas, motivo, botões. Digest até o teto, título/link + 1 linha + ações por item; layout em Q-26. Íntegra não reenviada (4096 chars). Texto completo no banco só para busca.

---

## 10. Diretrizes de arquitetura

| ID | Diretriz mantida como base | Limite |
|---|---|---|
| DEC-01 | Ingestão própria; sem Miniflux | Segundo sistema injustificado a 9/dia |
| DEC-02 | Postgres fonte da verdade (inclusive feeds) | Telegram muta estado; imagem sem estado |
| DEC-03 | Imagem única multi-arch com modos | Mesmo artefato para contínuo e cron |
| DEC-04 | Telegram long polling, sem portas | Sem TLS/porta pública |
| DEC-05 | Dedup 3 camadas sem embeddings | Volume baixo; validar em Q-25/Q-28 |
| DEC-06 | LLM atrás de `Scorer`; pequeno por padrão | Troca sem reescrita |
| DEC-07 | Rubrica + OPML inicial no Git; operacional no banco | Conciliação em Q-34; sem commit por comando |
| DEC-08 | LLM sem ferramentas, JSON validado | Mitiga injection; não prova fidelidade |
| DEC-09 | Few-shot, sem fine-tuning | Transparência; só com aprovação |
| DEC-10 | Python | Ecossistema do usuário |
| DEC-11 | Rubrica no repo refinada por feedback | Evita calibragem opaca |

### 10.1 Caminhos (discovery mantido)

- **A. Fork integral:** não. Monólito SQLite / estado em arquivos / porta / dedup fail-open; custo de adaptação > composição.
- **B. Composição:** sim. `feedparser + Trafilatura + aiogram + Pydantic`; `feedsearch-crawler/listparser/Instructor/Alembic` só após spike; nunca dois frameworks de bot/LLM; `python-telegram-bot` exige decisão GPL-3.0.
- **C. Escopo reduzido:** obrigatório para validar valor antes de sofistichar. Minoria SQLite/cron registrada; Postgres mantido sem reabrir por purismo.

---

## 11. Sequência de validação (proteções antes do primeiro envio)

| Etapa | Resultado e condição de avanço |
|---|---|
| 0 — Decisões | Fechar bloqueadores da §19 necessários à próxima etapa |
| 1 — Ingestão e estado | OPML, baseline, extração/fallback, identidade, recuperação, schema em Postgres real, sem entrega automática |
| 2 — Avaliação offline | Rubrica aprovada + rotulados + pares/grupos; medir relevância/clusters/custo/limiares antes de seleção automática |
| 3 — Operação controlada | Auth, callbacks, ledger/reservas, envio incerto, locks, migrações, healthcheck, backup/restauração, formato de mensagens, supervisor externo |
| 4 — Primeira versão utilizável | Seleção + entrega; feedback e gestão essencial; imagem verificada no host/arquiteturas |
| 5 — Evolução por evidência | Prévia, probation, pesos, few-shot, front — só com uso real |

Token do bot necessário para interação real; chave LLM para qualquer avaliação paga. Nenhuma chamada paga ou escrita em banco autorizada por este documento.

---

## 12. Matriz de disposição (council)

- **Manter na V1:** RF-001/002/003/005/006/008/010/011/012/013/016/017/018/020/023*/024/025/026*/027*/028/030/033/035/036/037/038/039 (* com redação corrigida e condicionais).
- **Simplificar:** RF-021 (contador+reserva+stop), RF-026 (só 👍👎🔖; expansão em Q-26), RF-004 (cobertura observada), RF-038 (heartbeat consultável; alerta externo fora da V1).
- **Adiar:** RF-007, RF-009, RF-014/RF-015 (só com Q-25/Q-28), RF-019, RF-022 default, RF-031, RF-032 rico, RF-034 automático, front.
- **Remover da V1:** modos/locks excedentes se só `serve`; RNF-018 como norma bloqueante; multi-canal.
- **Esclarecer antes de codar:** alerta⊆digest, retry incerto, fuso/virada, N de RF-010, Q-02/Q-05/Q-06/Q-07, correspondência feedback↔item.

---

## 13. Critérios de aceite + experimentos

- **CA-01:** Reimportar OPML não altera contagem.
- **CA-02:** `utm_*` diferente reutiliza identidade; outra fonte preserva origem sem nova entrega do evento.
- **CA-03:** Falhas antes/depois do envio e antes do registro; confirmados não repetem; ambíguo segue Q-24.
- **CA-04:** No rotulado, 3 fontes do mesmo evento = 1 item com 3 fontes; temas semelhantes não agrupam. Metas Q-28.
- **CA-05:** Feed novo não envia antigos.
- **CA-06:** Não autorizado sem resposta.
- **CA-07:** Sobe sem porta publicada.
- **CA-08:** Sem reserva suficiente, nenhuma tentativa LLM começa (inclui retry/cluster/prévia/comando); testar reinício/concorrência/virada (Q-27).
- **CA-09:** `healthcheck` falha com ciclo além do limite.
- **CA-10:** Mudança de rubrica/modelo roda precisão/recall/agrupamento em conjunto separado; promoção bloqueada fora dos limites Q-28.
- **CA-11:** Todo entregue tem link original.
- **CA-12:** Mesma imagem em `amd64`/`arm64`.

Experimentos propostos (não executados): EXP-01 ingestão/extração com fixtures; EXP-02 UX Telegram com texto fixo; EXP-03 ranking vs regra simples + clusters; EXP-04 falhas/orçamento; EXP-05 portabilidade multi-arch.

---

## 14. Questões em aberto **[NEEDS CLARIFICATION]**

Ausência de resposta não é aprovação.

| ID | Questão | Impacto | Proposta (não aprovada) |
|---|---|---|---|
| Q-01 | Front Tailscale? Quando? | Escopo | Fora V1; reavaliar pós 2–3 semanas |
| Q-02 | Qual API de LLM? | `Scorer` | Anthropic; interface trocável |
| Q-03 | Critério de importância em 1–2 frases? | Rubrica | Redigir na etapa 2 com exemplos |
| Q-04 | Quem rotula ~50 + grupos, em que formato? | Etapa 2 | Gerar lista; usuário marca relevância/eventos |
| Q-05 | Horário/fuso digest | RF-024 | Manhã `America/Sao_Paulo` |
| Q-06 | Teto e limiares alerta/digest | RF-023/024 | 5–7; alerta 5; digest ≥4 |
| Q-07 | Teto $ diário | RF-021 | Valor baixo ajustável |
| Q-08 | Probation e critério de desativar | RF-007 | 2 sem; <2,5 (só se recurso incluído) |
| Q-09 | N falhas p/ pausar | RF-010 | A definir |
| Q-10 | Host/recursos/arquiteturas (dado v1 desatualizado) | Operação | VPS a confirmar |
| Q-11 | Registry e visibilidade repo | Distribuição | A definir |
| Q-12 | Latência alerta (ciclo N min) | RNF-013 | 30–60 min |
| Q-13 | Posts atualizados pós-entrega | RF-016/027 | Ignorar; só novo post gera item |
| Q-14 | Paywall/trecho parcial | RF-012 | Pontuar trecho, marcar parcial |
| Q-15 | Retenção texto/descartados | Armazenamento | Sem expiração inicial |
| Q-16 | Privado vs grupo/canal | RF-028 | Privado |
| Q-17 | Backup Postgres | Confiabilidade | A definir |
| Q-18 | Few-shot com revisão por exemplo? | RF-034 | Sim |
| Q-19 | Modelo scoring vs cluster | RNF-012 | Pequeno p/ ambos; medir |
| Q-20 | pt-BR sempre? Ver original? | RNF-020 | Resumo pt-BR; link original |
| Q-21 | "Web" = Telegram Web ou interface própria? | Escopo | Telegram Web na V1; próprio opcional |
| Q-22 | Plataformas obrigatórias? Outro canal? | RNF-022 | Android/iOS/web/desktop via Telegram |
| Q-23 | Alertas no teto? Reaparecem? `/digest` manual? Dias vazios? | Entrega | Únicos no teto; omitir confirmados; sem preencher |
| Q-24 | Incerto: evitar duplicata ou perda? Reenvio manual? | Entrega | Reter incerto; ação explícita p/ reenviar |
| Q-25 | Janela de evento? Atualizar clusters pós-entrega? | Dedup | Identidade estável; fonte nova sem novo alerta default |
| Q-26 | Digest único vs 1/item? "Mais detalhes" onde? | UX | Comparar 7 itens; fallback só se aprovado |
| Q-27 | Moeda/preços/margem/fuso/tentativas sem uso no teto? | Custo | Dia `America/Sao_Paulo`, reserva conservadora |
| Q-28 | Metas recall/precisão/erro agrupamento? Separar calibração/avaliação? | Rubrica | Medir baseline primeiro, sem inventar % |
| Q-29 | Recorte inicial e composição vs adaptação? Comandos adiáveis? | Plano | Composição; essencial primeiro |
| Q-30 | 🔖 influencia ranking? Voto alterável? O que vai ao provedor? | Feedback | 🔖≠👍; voto substituível; só aprovados sem PII |
| Q-31 | Backlog pós-teto: validade/ordem/limites dia seguinte? | Custo | Reter+informar; sem rajada compensatória |
| Q-32 | Bot e ciclos mesmo processo? `run-once`/locks/`dry-run`? | Operação | `serve` integra; `run-once` sem comandos; `dry-run` sem custo |
| Q-33 | Quem detecta morte total? Prazo/canal? | Monitoramento | Supervisor externo; sem presumir porta |
| Q-34 | Rastrear feeds sem bot escritor de Git? | Rastreabilidade | OPML seed + histórico banco + export; snapshot só explícito |
| Q-35 | Qual resultado mínimo nos rotulados/spikes faria desistir do scoring LLM e manter só poda manual no Feeder? | Go/no-go | Definir antes da etapa 2; ex.: recall nos raros < regra simples |

---

## 15. Riscos

| ID | Risco | Mitigação |
|---|---|---|
| RISCO-01 | Falso negativo descarta o importante | Rotulados, recall, `/buscar` (quando existir) |
| RISCO-02 | Prompt injection | Sem ferramentas, schema, IDs conferidos |
| RISCO-03 | Custo fora de controle | Teto com reserva + `/stats` mínimo |
| RISCO-04 | Bot descoberto | Allowlist; nulo a desconhecidos |
| RISCO-05 | Feeds quebram/mudam | Backoff, pausa, aviso |
| RISCO-06 | Merge/split errado | Medir; exibir fontes |
| RISCO-07 | Perda banco | Backup Q-17 + restauração testada |
| RISCO-08 | Rate limit Telegram | Fila + backoff |

---

## 16. Glossário

Alerta, Digest, Cluster, Rubrica, Probation, Linha de base, Long polling, Few-shot (v2) + **Envio incerto:** `sendMessage` aceito sem confirmação persistida; exige reconciliação, não retry cego.

---

## 17. Referências

- OPML, v1, v2, discovery, prompt de avaliação (v1 referenciada intencionalmente no original).

## 18. Rastreabilidade v2 → v3

| Origem | Mudança v3 |
|---|---|
| Veredito council | §1; matriz §12; V1 mínima §3.1; adiados §3.2 |
| OBJ-01/02/05/07 | Sem mínimo artificial; recall nos raros; sem promessa 👍; reserva Q-27 |
| RF-004/008/012 | Cobertura observada + parcial explícito |
| RF-013–016/027; RNF-008/009 | Entrada/evento/entrega separados; `incerto`; Q-24/Q-25 |
| RF-021; RNF-011/015 | Reserva pré-tentativa + reconciliação; Q-27/Q-31 |
| RF-023/024/026/029/033 | Sem preencher; UX inequívoca Q-26; comandos em Q-29; voto/🔖 Q-30 |
| RF-034/035; RNF-019 | Manual aprovado; avaliação separada; dados mínimos |
| RF-036/037/038; RNF-001 | `serve` vs `run-once`; locks escopados; supervisor externo; Q-32/Q-33 |
| DEC-07; RNF-017 | Seed Git vs verdade banco; Q-34 |
| Q-35 | Critério de desistência do LLM (Socrates) |
| Demais IDs | Mantidos como candidatos |

## 19. Condições para fechar requisitos finais

1. Recorte inicial e plataformas (Q-01/Q-21/Q-22/Q-29) com adiados e dependências.
2. Importância/rotulagem/métricas/regressão (Q-03/Q-04/Q-28/Q-35); avaliação separada do few-shot.
3. Provedor/modelo/custo/backlog (Q-02/Q-07/Q-19/Q-27/Q-31) antes de qualquer uso pago.
4. Teto, alerta/digest, evento, incerto (Q-06/Q-12/Q-13/Q-23/Q-24/Q-25).
5. Formato/ações/privacidade (Q-16/Q-18/Q-20/Q-26/Q-30).
6. Operação/retenção/backup/monitoramento (Q-05/Q-09/Q-10/Q-11/Q-15/Q-17/Q-32/Q-33/Q-34). Q-08 só bloqueia se probation incluída.
7. Para cada Q: resposta, responsável, data, justificativa, IDs afetados; condicionais adiadas com o recurso.
8. Aprovação explícita do escopo. Até lá, **rascunho**; M/S/C e discovery não autorizam implementação.
