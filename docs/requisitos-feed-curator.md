# Feed Curator — Documento de Requisitos (insumo para `/speckit.specify`)

> **Status:** rascunho para especificação · **Data:** 2026-10-31
> **Origem:** conversa de descoberta sobre excesso de volume de notícias no app Feeder (Android).
> **Como usar este documento:** as seções 1 a 9 descrevem *o quê* e *por quê* (entrada principal do `specify`).
> A seção 10 registra decisões de arquitetura já tomadas, para o `plan`. A seção 13 lista o que ainda
> **depende de decisão** e deve virar `[NEEDS CLARIFICATION]` na SPEC.

---

## 1. Contexto e problema

O usuário acompanha 21 feeds RSS sobre dados, ML e IA (arquivo [feeds-dados-ml-ia-curado.opml](feeds-dados-ml-ia-curado.opml)), lidos hoje
no app **Feeder** para Android. O volume é alto demais para leitura integral, e o leitor não separa o
que é importante do que é ruído.

### 1.1 Linha de base medida (2026-10-31, janela de 30 dias)

Total: **265 posts em 30 dias (~9/dia)** em 21 feeds.

| Feed | Posts/30d | Observação |
|---|---:|---|
| OpenAI News | 71 | Mistura lançamentos com casos de clientes e institucional |
| Real Python | 32 | Tutoriais, não notícia |
| LangChain Blog | 32 | Muito post de produto e de cliente |
| Simon Willison's Weblog | 30 | Alto sinal, mas alta frequência |
| Hugging Face Blog | 25 | |
| Latent Space | 20 | |
| Google Research Blog | 11 | |
| Netflix Tech Blog | 10 | |
| DuckDB Blog | 9 | |
| Interconnects | 7 | |
| Data Engineering Weekly | 6 | |
| Import AI | 4 | |
| dbt Developer Blog | 2 | |
| Armin Ronacher | 2 | |
| Drew Breunig | 2 | |
| Ahead of AI (Raschka) | 1 | |
| Hamel Husain | 1 | |
| Benn Stancil | 0 | |
| Lil'Log (Lilian Weng) | 0 | |
| Chip Huyen | 0 | |
| Eugene Yan | 0 | |

**Leituras relevantes:**
- Quatro feeds (OpenAI, Real Python, LangChain, Simon Willison) geram **62%** do volume.
- Os autores de maior sinal (Weng, Huyen, Yan, Raschka, Hamel, Benn) publicam de 0 a 2 vezes por mês.
  Cortar feeds por volume seria errado; o problema é o ruído nos feeds de alta frequência.
- Quando um lançamento relevante ocorre, vários feeds comentam o mesmo evento no mesmo dia
  (duplicidade temática), o que um filtro por palavra-chave não resolve.

### 1.2 Alternativas descartadas ou rebaixadas

| Alternativa | Motivo |
|---|---|
| Podar feeds e silenciar notificações no Feeder | Útil como paliativo imediato, mas não resolve ranking nem duplicidade temática |
| Filtro por palavra-chave no leitor (bloqueio por título) | Não distingue "importante" de "mesmo tema"; depende de recurso do leitor a confirmar |
| Miniflux / FreshRSS / Inoreader com regras | Filtro por regex, não por importância; Miniflux acrescentaria um segundo sistema sem necessidade neste volume |
| Script simples em cron | Frágil: sem idempotência, sem deduplicação, sem feedback, sem observabilidade |

---

## 2. Objetivos

| ID | Objetivo | Medida de sucesso |
|---|---|---|
| OBJ-01 | Reduzir o volume de leitura diária de ~9 posts/dia para no máximo 5 a 7 itens relevantes | Itens entregues/dia ≤ teto configurado |
| OBJ-02 | Garantir que o que é realmente importante não seja perdido | Recall do ranking contra o conjunto rotulado (meta definida na Fase 2) |
| OBJ-03 | Eliminar repetição do mesmo evento coberto por várias fontes | Um evento = um item entregue, com as demais fontes listadas |
| OBJ-04 | Entregar e permitir operar em qualquer plataforma (mobile Android e iOS, web, desktop), sem fricção e em qualquer rede | Mesma experiência de leitura, feedback e gestão nas plataformas-alvo; sem VPN nem porta exposta |
| OBJ-05 | Aprender o gosto do usuário ao longo do tempo | Taxa de 👍 sobre itens entregues cresce entre rubricas sucessivas |
| OBJ-06 | Operar de forma portátil e barata | Uma imagem de container que roda em qualquer host com Postgres |
| OBJ-07 | Manter o custo de LLM previsível | Gasto diário nunca acima do teto configurado |

---

## 3. Escopo

### 3.1 Dentro do escopo
- Ingestão de feeds RSS/Atom com controle em banco (Postgres).
- Deduplicação em camadas (exata, quase igual, mesmo evento).
- Pontuação de importância por LLM com rubrica versionada.
- Entrega no Telegram: alerta imediato e digest diário.
- Gestão de feeds e operação pelo próprio Telegram (sem front-end).
- Feedback do usuário (👍 👎 🔖) e uso dele para refinar a rubrica.
- Empacotamento em imagem de container multi-arch.
- Observabilidade mínima (heartbeat, custo, falhas de feed).

### 3.2 Fora do escopo (nesta versão)
- Front-end web (ver seção 9; candidato a fase opcional).
- Múltiplos usuários ou multi-tenant.
- Reenvio do texto integral dos artigos pelo Telegram.
- Treinamento de modelo próprio (o aprendizado é por exemplos na rubrica, não por fine-tuning).
- Fontes que não sejam RSS/Atom (newsletters por e-mail, redes sociais, scraping de páginas sem feed).
- Tradução do artigo completo (apenas o resumo é em pt-BR).

---

## 4. Usuário e cenários

**Usuário único:** desenvolvedor brasileiro, hoje lê no Android (Feeder), mas quer acessar de **qualquer
plataforma** (mobile, web, desktop). Aceita infraestrutura própria (VPS, Docker, Postgres) e tem acesso a API
de LLM.

| ID | Cenário |
|---|---|
| CEN-01 | Recebo um alerta no Telegram quando sai algo de nota máxima (ex.: lançamento relevante), com resumo, motivo e link |
| CEN-02 | Recebo toda manhã um digest curto com os itens mais relevantes do dia anterior |
| CEN-03 | Quando vários feeds cobrem o mesmo evento, vejo um item só, com a lista das demais fontes |
| CEN-04 | Descubro um blog novo, envio a URL ao bot e vejo uma prévia (volume e nota média estimados) antes de ativar |
| CEN-05 | Marco um item com 👎 ou 👍 e o sistema passa a refletir meu gosto |
| CEN-06 | Guardo um item para depois com 🔖 |
| CEN-07 | Quero mais contexto antes de abrir o link e toco em "Mais detalhes" |
| CEN-08 | Um feed quebra ou fica ruidoso e o sistema me avisa, em vez de falhar em silêncio |
| CEN-09 | Migro o app para outro servidor levando só o banco e as variáveis de ambiente |
| CEN-10 | Importo ou exporto meus feeds em OPML enviando ou recebendo um arquivo no Telegram |
| CEN-11 | Leio, dou feedback e gerencio feeds a partir de qualquer dispositivo (celular Android ou iOS, navegador, desktop), com o mesmo estado sincronizado |

---

## 5. Requisitos funcionais

Prioridade: **M** = deve, **S** = deveria, **C** = pode.

### 5.1 Gestão de feeds

| ID | Requisito | Prio. |
|---|---|:--:|
| RF-001 | Importar feeds de um arquivo OPML de forma idempotente (reimportar não duplica) | M |
| RF-002 | Exportar os feeds ativos em OPML | M |
| RF-003 | Adicionar feed informando a URL do feed **ou** a URL do site, com autodescoberta (`<link rel="alternate">` e caminhos comuns) | M |
| RF-004 | Ao adicionar um feed, exibir prévia: posts nos últimos 30 dias, amostra dos últimos 10 posts pontuada e nota média, com confirmação explícita antes de ativar | S |
| RF-005 | Ao ativar um feed, registrar os posts existentes como "já vistos" (linha de base), sem enviá-los | M |
| RF-006 | Desativar, remover e ajustar o peso a priori de um feed | M |
| RF-007 | Período de experiência (probation) configurável para feeds novos, com entrega apenas no digest e sugestão de desativar se a nota média for baixa | S |
| RF-008 | Listar feeds com posts/mês, nota média e estado (ativo, probation, pausado) | M |
| RF-009 | Silenciar temporariamente uma fonte | S |
| RF-010 | Pausar automaticamente feed com N falhas consecutivas e avisar o usuário | M |

### 5.2 Ingestão e deduplicação

| ID | Requisito | Prio. |
|---|---|:--:|
| RF-011 | Buscar feeds com GET condicional (`ETag`, `If-Modified-Since`) e backoff em caso de falha | M |
| RF-012 | Extrair o texto completo do artigo quando o feed traz apenas um trecho | M |
| RF-013 | Deduplicação exata por URL canonicalizada (sem `utm_*` e fragmento) e hash de conteúdo | M |
| RF-014 | Deduplicação de quase iguais por similaridade de título (republicações, crossposts) | M |
| RF-015 | Agrupar posts do mesmo evento (cobertos por fontes diferentes) em um cluster, entregue como item único com a lista das demais fontes | M |
| RF-016 | Processar cada post no máximo uma vez, de forma idempotente, mesmo após reinício do serviço | M |

### 5.3 Pontuação

| ID | Requisito | Prio. |
|---|---|:--:|
| RF-017 | Pontuar cada item de 1 a 5 com um LLM, devolvendo: nota, categoria, motivo (uma linha) e resumo curto em pt-BR, em JSON validado por schema | M |
| RF-018 | A rubrica de importância é um arquivo versionado no repositório; cada pontuação registra a versão da rubrica e o modelo usado | M |
| RF-019 | Cada fonte tem peso a priori que influencia a seleção | S |
| RF-020 | O provedor de LLM fica atrás de uma interface, trocável sem alterar o restante do sistema | M |
| RF-021 | Respeitar um teto diário de gasto com LLM; ao atingi-lo, degradar de forma segura (ex.: reter para o dia seguinte) e avisar | M |
| RF-022 | Gerar, sob demanda, um resumo mais longo (~800 caracteres) de um item | S |

### 5.4 Entrega e interação (Telegram)

| ID | Requisito | Prio. |
|---|---|:--:|
| RF-023 | Enviar alerta imediato para itens com nota máxima, com título, fontes, resumo curto, motivo e link da matéria original | M |
| RF-024 | Enviar digest diário em horário configurável, com 5 a 7 itens (teto configurável), título como link e uma linha de resumo | M |
| RF-025 | Todo item entregue contém o link da matéria completa; o texto integral **não** é reenviado | M |
| RF-026 | Botões por item: 👍, 👎, 🔖 e "Mais detalhes" (este substitui o resumo curto pelo longo na própria mensagem) | M |
| RF-027 | Garantir que um item nunca seja entregue duas vezes (unicidade por item e canal) | M |
| RF-028 | Responder apenas a IDs de usuário autorizados; ignorar os demais sem resposta | M |
| RF-029 | Comandos: `/addfeed`, `/feeds`, `/disable`, `/remove`, `/weight`, `/digest`, `/mute`, `/stats`, `/rubrica`, `/import`, `/export`, `/buscar` | M |
| RF-030 | `/import` aceita arquivo `.opml` anexado; `/export` devolve o OPML como anexo | M |
| RF-031 | `/buscar termo` pesquisa o arquivo (inclusive itens descartados) por texto | S |
| RF-032 | `/stats` mostra gasto de LLM (dia e mês), itens entregues e taxa de 👍/👎 | S |

### 5.5 Aprendizado

| ID | Requisito | Prio. |
|---|---|:--:|
| RF-033 | Registrar o feedback (👍 👎 🔖) vinculado ao item, à nota atribuída e à versão da rubrica | M |
| RF-034 | Usar periodicamente os exemplos mais claros de feedback como few-shot na rubrica, mediante aprovação do usuário | S |
| RF-035 | Manter conjunto de ~50 posts rotulados manualmente para medir a precisão do ranking e servir de teste de regressão a cada mudança de rubrica | M |

### 5.6 Operação

| ID | Requisito | Prio. |
|---|---|:--:|
| RF-036 | Modos de execução da mesma imagem: `serve` (contínuo), `run-once` (cron/CronJob), `migrate`, `dry-run` (imprime sem enviar), `healthcheck` | M |
| RF-037 | Impedir execução concorrente de duas instâncias sobre o mesmo banco (advisory lock) | M |
| RF-038 | Gravar heartbeat no banco (último ciclo concluído) e avisar no Telegram se o digest esperado não saiu | M |
| RF-039 | Aplicar migrações de schema de forma versionada e repetível | M |

---

## 6. Requisitos não funcionais

| ID | Categoria | Requisito |
|---|---|---|
| RNF-001 | Portabilidade | Uma única imagem de container, multi-arch (`amd64` e `arm64`), executável em VPS, Raspberry Pi, Fly.io ou Kubernetes |
| RNF-002 | Portabilidade | Postgres externo configurado por `DATABASE_URL`; um `docker-compose.yml` sobe app e banco para uso rápido |
| RNF-003 | Segurança | O container **não expõe nenhuma porta**; só conexões de saída (Telegram, API do LLM, feeds, Postgres) |
| RNF-004 | Segurança | Segredos apenas por variável de ambiente; nunca na imagem nem no repositório |
| RNF-005 | Segurança | Autorização por lista de IDs de usuário do Telegram (`TELEGRAM_ALLOWED_USER_IDS`) |
| RNF-006 | Segurança | Conteúdo dos feeds é não confiável: o LLM roda **sem ferramentas** e só devolve dados validados por schema (defesa contra prompt injection) |
| RNF-007 | Segurança | Imagem executa com usuário não-root |
| RNF-008 | Confiabilidade | Processamento idempotente: reinício, retry ou execução duplicada não geram envio duplicado |
| RNF-009 | Confiabilidade | Retry com backoff para feeds, LLM e Telegram, respeitando limites de taxa do Telegram |
| RNF-010 | Confiabilidade | Falha de um feed ou de uma chamada de LLM não interrompe o ciclo dos demais |
| RNF-011 | Custo | Teto diário de gasto com LLM configurável e aplicado |
| RNF-012 | Custo | Uso de modelo pequeno por padrão para pontuação; modelo maior apenas se justificado por medição |
| RNF-013 | Desempenho | Latência entre a publicação de um post e o alerta de nota máxima: a definir (ver Q-12) |
| RNF-014 | Observabilidade | Logs estruturados; heartbeat consultável; falhas de feed registradas por feed |
| RNF-015 | Observabilidade | Registrar tokens e custo por chamada de LLM em tabela própria |
| RNF-016 | Qualidade | Testes de integração contra Postgres real; fixtures de feeds; conjunto rotulado como teste de regressão do ranking |
| RNF-017 | Qualidade | Rubrica e lista de feeds versionadas; toda mudança é um commit rastreável |
| RNF-018 | Manutenibilidade | Arquivos Python com no máximo 300 linhas, conforme padrão do usuário |
| RNF-019 | Privacidade | Dados ficam no Postgres do usuário; nenhum dado além do conteúdo do post enviado ao provedor de LLM |
| RNF-020 | Usabilidade | Mensagens em pt-BR com acentuação correta; resumos legíveis em tela de celular |
| RNF-022 | Multiplataforma | Todas as funções de leitura, feedback e gestão estão disponíveis em Android, iOS, web e desktop, sem depender de um cliente específico; o estado (feedback, feeds, itens entregues) é único e vive no servidor |
| RNF-023 | Multiplataforma | Nenhuma plataforma exige VPN, instalação de app proprietário do projeto ou porta aberta para funcionar (o front opcional via Tailscale é a exceção declarada) |
| RNF-021 | Portabilidade de dados | Exportação OPML dos feeds a qualquer momento, evitando dependência do app |

---

## 7. Modelo de dados (conceitual)

| Tabela | Papel |
|---|---|
| `feeds` | URL, `ETag`, `Last-Modified`, estado, falhas consecutivas, peso a priori, categoria, data de ativação |
| `entries` | URL canônica (única), hash do conteúdo, título, texto completo, data de publicação, feed de origem |
| `clusters` | Agrupamento de entradas que tratam do mesmo evento |
| `scores` | Nota, categoria, motivo, resumos curto e longo, versão da rubrica, modelo, JSON bruto |
| `deliveries` | Item entregue (única por item e canal), id da mensagem no Telegram, data |
| `feedback` | Reação do usuário, vinculada ao item e à versão da rubrica |
| `llm_usage` | Tokens e custo por chamada e por dia |
| `heartbeat` | Último ciclo concluído, para o `healthcheck` |

Extensão de Postgres prevista: `pg_trgm` (similaridade de título).

---

## 8. Estratégia de deduplicação (três camadas)

1. **Exata:** URL canonicalizada e hash de conteúdo, via constraint `UNIQUE`.
2. **Quase igual:** `pg_trgm` sobre o título, para republicações e crossposts.
3. **Mesmo evento:** o LLM recebe títulos e resumos do dia e agrupa o que cobre a mesma notícia.
   A escolha dispensa embeddings, porque o volume é de ~9 itens/dia e não justifica outro provedor nem
   o modelo local.

---

## 9. Trade-off: Telegram × front-end (Tailscale)

### 9.1 Comparação

| Critério | Telegram | Front-end via Tailscale |
|---|---|---|
| Plataformas cobertas | Android, iOS, desktop (Windows, macOS, Linux) e web (web.telegram.org), com sincronização nativa entre dispositivos | Qualquer navegador, mas só em dispositivos com Tailscale instalado e conectado |
| Notificação e triagem no celular | Nativa, funciona em qualquer rede | Push próprio necessário; PWA no Android é limitado e no iOS ainda mais restrito |
| Acesso fora de casa | Sempre | Só com o app do Tailscale conectado |
| Acesso de dispositivo emprestado ou sem instalação (ex.: computador de terceiros) | Possível via web.telegram.org com login | Inviável sem instalar o Tailscale |
| Conflito de VPN no Android | Nenhum | Android permite uma VPN ativa por vez; conflita com outra VPN do usuário |
| Bateria | Sem custo adicional | Conexão permanente adicional |
| Exposição de rede | Nenhuma porta (long polling) | Sem porta pública com `tailscale serve`, mas exige o serviço de rede |
| Esforço de construção | Baixo | Alto (autenticação, interface, deploy, manutenção) |
| Explorar arquivo, filtrar, buscar histórico, ver descartados, reavaliar notas | Limitado (`/buscar`, `/stats`) | Muito melhor |
| Gestão de feeds | Suficiente via comandos e botões | Mais confortável |

### 9.2 Decisão tomada
**Telegram como interface única da primeira versão**, usando *long polling* (`getUpdates`) em vez de webhook,
o que elimina URL pública, certificado TLS e porta aberta. O app pode rodar atrás de NAT ou CGNAT.

### 9.3 Decisão adiada
Front-end somente leitura (histórico, busca, notas, itens descartados), publicado com `tailscale serve`, como
**fase opcional** após 2 a 3 semanas de uso real. O modelo de dados já comporta a adição sem retrabalho.
Ver Q-01.

**Nota sobre multiplataforma (OBJ-04):** o Telegram já cobre mobile, desktop e web, então o requisito é
atendido sem front próprio. O que o Telegram **não** cobre é uma interface web com navegação rica (filtros,
histórico completo, reavaliação de notas). Se o usuário considerar isso parte de "web", o front deixa de ser
opcional e sobe de prioridade (ver Q-21). Nesse caso, o front precisaria de autenticação própria, e a escolha
entre Tailscale e exposição pública com login passa a pesar na arquitetura (um front acessível de qualquer
navegador, sem Tailscale, contraria o RNF-003 e exigiria rever a decisão DEC-04).

### 9.4 Formato das mensagens

- **Alerta (nota 5):** título, fontes (com "também cobriram"), resumo de 2 a 3 linhas, motivo ("por que te
  interessa"), botões `[🔗 Ler matéria] [👍] [👎] [🔖] [📖 Mais detalhes]`.
- **Digest (nota ≥ 4):** uma mensagem com 5 a 7 itens; título como link e uma linha de resumo; botões por item.
- **Resumo, não íntegra:** a íntegra anularia o objetivo de reduzir volume, esbarra no limite de 4.096
  caracteres por mensagem do Telegram e republicaria conteúdo de terceiros. O texto completo fica no banco só
  para busca.
- **Instant View:** em sites compatíveis, o Telegram abre o link em leitura limpa dentro do app.

---

## 10. Decisões de arquitetura já tomadas (insumo para `/speckit.plan`)

| ID | Decisão | Justificativa |
|---|---|---|
| DEC-01 | Ingestão própria; **sem Miniflux** | Com Postgres aceito e imagem única, um segundo sistema só acrescentaria manutenção a 9 posts/dia |
| DEC-02 | Postgres como fonte da verdade (inclusive feeds) | Mudanças feitas pelo Telegram precisam persistir; imagem sem estado local |
| DEC-03 | Imagem única multi-arch com modos de execução | Roda em qualquer lugar; mesma imagem para serviço contínuo e cron |
| DEC-04 | Telegram por long polling, sem portas abertas | Dispensa exposição de rede e TLS |
| DEC-05 | Deduplicação em três camadas sem embeddings | Volume baixo; evita segundo provedor |
| DEC-06 | LLM atrás de interface (`Scorer`); modelo pequeno por padrão | Troca de provedor sem reescrita; custo previsível |
| DEC-07 | Rubrica e feeds versionados no repositório | Rastreabilidade e reprodutibilidade |
| DEC-08 | LLM sem ferramentas, com saída em JSON validado | Mitiga prompt injection vindo de conteúdo de feeds |
| DEC-09 | Aprendizado por few-shot na rubrica, sem fine-tuning | Simplicidade e transparência |
| DEC-10 | Linguagem Python | Alinhado ao ecossistema do usuário |
| DEC-11 | Fonte da verdade da rubrica de importância fica no repositório; o gosto do usuário a refina por feedback | Evita calibragem implícita e opaca |

---

## 11. Fases propostas

| Fase | Entrega |
|---|---|
| 0 | Repositório (`git init`), esqueleto Python, schema com migrações, Dockerfile, compose; `import-opml` carrega os 21 feeds |
| 1 | Ingestão (GET condicional, texto completo, dedup exata e por título), `dry-run` e esqueleto do bot (autorização, `/feeds`, `/import`, `/export`, `/addfeed` sem prévia) |
| 2 | Rubrica v1, conjunto rotulado de ~50 posts, medição de precisão |
| 3 | Pontuador, clusterização por evento, alerta e digest no Telegram, `/addfeed` com prévia |
| 4 | Botões de feedback, few-shot, health check, teto de custo |
| 5 | CI que gera e publica a imagem multi-arch; testes de integração contra Postgres real |
| 6 *(opcional)* | Front somente leitura via Tailscale (ver Q-01) |

A Fase 0 e a Fase 1 não exigem segredos; a partir da Fase 3 são necessários o token do bot e a chave do LLM.

---

## 12. Critérios de aceite de alto nível

- **CA-01:** Reimportar o mesmo OPML não altera a contagem de feeds.
- **CA-02:** Um mesmo post (inclusive com `utm_*` diferente) nunca gera dois itens.
- **CA-03:** Reiniciar o serviço no meio de um ciclo não produz envio duplicado.
- **CA-04:** Um evento coberto por 3 fontes aparece como um item, listando as demais.
- **CA-05:** Ao ativar um feed novo, nenhum post antigo é enviado.
- **CA-06:** Mensagens de usuário não autorizado não recebem resposta.
- **CA-07:** O container sobe sem nenhuma porta publicada.
- **CA-08:** Com o teto de custo atingido, nenhuma chamada adicional ao LLM é feita no dia, e o usuário é avisado.
- **CA-09:** `curator healthcheck` falha quando o último ciclo é mais antigo que o limite configurado.
- **CA-10:** A medição de precisão do ranking contra o conjunto rotulado é executada em CI a cada mudança de rubrica.
- **CA-11:** Todo item entregue contém o link da matéria original.
- **CA-12:** A mesma imagem executa em `amd64` e `arm64`.

---

## 13. Questões em aberto (pendentes de decisão)

Marcar cada uma como `[NEEDS CLARIFICATION]` na SPEC até que seja resolvida.

| ID | Questão | Impacto | Padrão assumido (se não decidido) |
|---|---|---|---|
| Q-01 | Incluir o front somente leitura via Tailscale? Quando? | Escopo e fases | Fora da primeira versão; reavaliar após 2 a 3 semanas de uso |
| Q-02 | **Qual API de LLM** o usuário tem? Anthropic, OpenAI ou outra? | Primeira implementação do `Scorer` | Anthropic, com interface preparada para trocar |
| Q-03 | Qual o critério de importância em uma ou duas frases (o que interessa, o que não interessa)? | Rubrica v1 | Rubrica a ser redigida na Fase 2 a partir de exemplos |
| Q-04 | Quem rotula os ~50 posts da linha de base, e em que formato (planilha)? | Fase 2 | Gerar a lista; usuário marca sim/não (~15 min) |
| Q-05 | Horário e fuso do digest diário | RF-024 | Manhã, fuso `America/Sao_Paulo`; horário a definir |
| Q-06 | Teto de itens do digest e limiar de nota para o digest e o alerta | RF-023, RF-024 | 5 a 7 itens; alerta nota 5; digest nota ≥ 4 |
| Q-07 | Teto diário de gasto com LLM (valor) | RF-021 | A definir; sugestão de valor baixo, ajustável |
| Q-08 | Duração do período de experiência (probation) de feeds novos e critério de sugestão de desativar | RF-007 | 2 semanas; sugerir desativar se nota média < 2,5 |
| Q-09 | Quantas falhas consecutivas pausam um feed (N) | RF-010 | A definir |
| Q-10 | Onde hospedar: Contabo (já paga até 2026-09-08, a confirmar vigência) ou outro host? | Fase 5 | VPS existente, a confirmar |
| Q-11 | Registry da imagem (GHCR, Docker Hub, privado) e visibilidade do repositório | Fase 5 | A definir |
| Q-12 | Latência aceitável para alerta de nota máxima (ciclo a cada N minutos) | RNF-013 | Ciclo de ingestão a cada 30 a 60 min |
| Q-13 | Política para posts atualizados após a entrega (reenviar, editar a mensagem ou ignorar) | RF-016, RF-027 | Ignorar atualizações; apenas novo post gera item |
| Q-14 | Tratamento de feeds com paywall ou só trecho que não permite texto completo | RF-012 | Pontuar apenas com o trecho disponível e marcar como "parcial" |
| Q-15 | Retenção de dados (por quanto tempo guardar texto completo e itens descartados) | Armazenamento, RNF-019 | Sem expiração inicialmente; revisar |
| Q-16 | Chat privado ou grupo/canal do Telegram para as entregas | RF-028 | Chat privado com o bot |
| Q-17 | Estratégia de backup do Postgres | Confiabilidade | A definir (o OPML cobre só os feeds, não o histórico nem o feedback) |
| Q-18 | Aprovação do few-shot: o usuário revisa cada exemplo antes de entrar na rubrica? | RF-034 | Sim, mediante aprovação |
| Q-19 | Modelo exato para pontuação e para o agrupamento por evento (mesmo modelo ou diferentes?) | RNF-012 | Modelo pequeno para ambos; medir antes de subir |
| Q-20 | Idioma-alvo: resumos sempre em pt-BR mesmo para fontes em inglês? Haverá opção de ver o original? | RNF-020 | Resumo em pt-BR; link leva ao original |
| Q-21 | "Web" do OBJ-04 é satisfeita por Telegram Web/Desktop, ou o usuário quer uma **interface web própria** (leitura, busca, gestão)? | Escopo; reabre a Q-01 e muda a prioridade do front | Telegram Web/Desktop atende a primeira versão; front próprio permanece opcional |
| Q-22 | Plataformas-alvo obrigatórias: Android, iOS, web, desktop? Alguma outra (ex.: smartwatch, e-mail como canal de entrega)? | RNF-022; pode exigir novo canal de entrega | Android, iOS, web e desktop, todas via Telegram |

---

## 14. Riscos

| ID | Risco | Mitigação |
|---|---|---|
| RISCO-01 | Rubrica ruim descarta o que importa (falso negativo) | Conjunto rotulado, medição de recall, itens descartados pesquisáveis via `/buscar` |
| RISCO-02 | Prompt injection via conteúdo de feed | LLM sem ferramentas, saída validada por schema |
| RISCO-03 | Custo de LLM fora de controle | Teto diário aplicado e `/stats` |
| RISCO-04 | Bot descoberto por terceiros | Lista de IDs autorizados; resposta nula a desconhecidos |
| RISCO-05 | Feeds mudam de formato ou quebram | Backoff, pausa automática e aviso no Telegram |
| RISCO-06 | Agrupamento por evento junta itens distintos (ou separa iguais) | Medir no conjunto rotulado; permitir ver as fontes agrupadas |
| RISCO-07 | Perda do histórico e do feedback por falha do banco | Backup do Postgres (Q-17) |
| RISCO-08 | Limites de taxa do Telegram em rajadas de alertas | Fila com respeito ao limite e retry com backoff |

---

## 15. Glossário

| Termo | Significado |
|---|---|
| Alerta | Mensagem imediata para item de nota máxima |
| Digest | Resumo diário agrupado de itens de nota alta |
| Cluster | Conjunto de posts de fontes diferentes sobre o mesmo evento |
| Rubrica | Arquivo versionado com os critérios de importância usados pelo LLM |
| Probation | Período em que um feed novo só entrega no digest, para avaliar sua qualidade |
| Linha de base | Posts existentes no momento da ativação de um feed, registrados como já vistos |
| Long polling | Modo em que o bot busca atualizações por conexão de saída, sem webhook |
| Few-shot | Exemplos reais de feedback incluídos na rubrica para calibrar o LLM |

---

## 16. Referências do repositório

- [feeds-dados-ml-ia-curado.opml](feeds-dados-ml-ia-curado.opml): os 21 feeds atuais, em três categorias (Engenharia de Dados, Machine
  Learning, LLMs e Agentes).
