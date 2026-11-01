# Feed Curator — Requisitos v2 (rascunho para decisão)

> **Status:** proposta em revisão; requisitos finais e solução ainda não aprovados.
> **Data:** 2026-10-31 · **Versão:** 2.0-draft.
> **Origem:** revisão documental da [v1](requisitos-feed-curator.md) à luz do [discovery GitHub](investigacoes/discovery-github-feed-curator.md).
> Esta revisão não executa o [prompt de avaliação](prompt-avaliacao-requisitos-feed-curator.md) integralmente e não o substitui. Não houve nova validação de projetos externos nem experimento técnico.

## 0. Como interpretar esta versão

Esta v2 é uma proposta autônoma para discussão. A v1 permanece preservada como histórico. Criar esta versão não aprova arquitetura, fornecedores, metas ou escopo de implementação.

- **Necessidades:** reduzir ruído, preservar informação relevante, agrupar cobertura repetida e permitir leitura e feedback acessíveis.
- **Requisitos candidatos:** as seções 2 a 6 registram o comportamento desejado. M/S/C indicam prioridade proposta, não aprovação nem fase obrigatória.
- **Diretrizes herdadas:** Python, Postgres, Telegram e imagem única permanecem como base de comparação da v1. Não foram revogados nem ratificados novamente; dependências e forma de reaproveitamento continuam em aberto.
- **Propostas de correção:** as redações alteradas nesta versão são recomendações do agente. Questões sem resposta ficam marcadas na seção 13; proposta ou ausência de resposta não constitui decisão.
- **Fechamento:** somente após registrar as decisões e completar os critérios da seção 18 esta versão poderá originar uma SPEC aprovada. Pode-se realizar trabalho independente das pendências, desde que autorizado e sem adotar silenciosamente suas respostas.

Os IDs OBJ, CEN, RF, RNF, DEC, CA, Q e RISCO da v1 foram preservados. Novas perguntas começam em Q-23. As seções 17 e 18 registram a rastreabilidade e o processo de fechamento.

---

## 1. Contexto e problema

O usuário acompanha 21 feeds RSS sobre dados, ML e IA (arquivo [feeds-dados-ml-ia-curado.opml](feeds-dados-ml-ia-curado.opml)), lidos hoje
no app **Feeder** para Android. O volume é alto demais para leitura integral, e o leitor não separa o
que é importante do que é ruído.

### 1.1 Linha de base informada na v1 (2026-10-31, janela de 30 dias)

Total informado: **265 posts em 30 dias (~9/dia)** em 21 feeds. A v2 não refez essa medição. A cobertura do histórico e o método de coleta precisam ser registrados antes de usar os números como referência de aceite.

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

### 1.2 Alternativas rebaixadas na v1 e limites da comparação

| Alternativa | Motivo |
|---|---|
| Podar feeds e silenciar notificações no Feeder | Útil como paliativo imediato, mas não resolve ranking nem duplicidade temática |
| Filtro por palavra-chave no leitor (bloqueio por título) | Não distingue "importante" de "mesmo tema"; depende de recurso do leitor a confirmar |
| Miniflux / FreshRSS / Inoreader com regras | Filtro por regex, não por importância; Miniflux acrescentaria um segundo sistema sem necessidade neste volume |
| Script simples em cron | Só atende se incorporar persistência, recuperação, deduplicação, feedback e observabilidade; cron por si só não implica fragilidade |

---

## 2. Objetivos

| ID | Objetivo | Medida de sucesso |
|---|---|---|
| OBJ-01 | Reduzir o volume diário de leitura, com teto configurável e sem preencher vagas com conteúdo irrelevante | Proposta inicial: até 5 a 7 itens; teto global e exceções dependem de Q-06/Q-23 |
| OBJ-02 | Minimizar a perda de notícias importantes | Recall e precisão medidos no conjunto rotulado; metas a aprovar em Q-28 |
| OBJ-03 | Eliminar repetição do mesmo evento coberto por várias fontes | Um evento = um item entregue, com as demais fontes listadas |
| OBJ-04 | Entregar e permitir operar em qualquer plataforma (mobile Android e iOS, web, desktop), sem fricção e em qualquer rede | Mesma experiência de leitura, feedback e gestão nas plataformas-alvo; sem VPN nem porta exposta |
| OBJ-05 | Refinar a curadoria a partir de feedback explícito | Comparar rubricas no mesmo conjunto de avaliação e acompanhar feedback sem assumir crescimento garantido de 👍 |
| OBJ-06 | Operar de forma portátil e barata | Uma imagem de container que roda em qualquer host com Postgres |
| OBJ-07 | Manter o custo de LLM previsível | Nenhuma chamada começa sem saldo reservado suficiente; moeda, preços e teto a definir em Q-07/Q-27 |

---

## 3. Escopo

### 3.1 Escopo candidato (sujeito à priorização final)
- Ingestão de feeds RSS/Atom com controle em banco (Postgres).
- Deduplicação em camadas (exata, quase igual, mesmo evento).
- Pontuação de importância por LLM com rubrica versionada.
- Entrega no Telegram: alerta imediato e digest diário.
- Gestão de feeds e operação pelo próprio Telegram (sem front-end).
- Feedback do usuário (👍 👎 🔖) e uso dele para refinar a rubrica.
- Empacotamento em imagem de container multi-arch.
- Observabilidade mínima (heartbeat, custo, falhas de feed).

### 3.2 Exclusões propostas para a primeira versão
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

Prioridade proposta: **M** = deve, **S** = deveria, **C** = pode. Os requisitos ligados a questões abertas são condicionais até sua resolução. O termo “item” na entrega representa um cluster ou uma entrada sem cluster, conforme política a fechar em Q-25.

### 5.1 Gestão de feeds

| ID | Requisito | Prio. |
|---|---|:--:|
| RF-001 | Importar feeds de um arquivo OPML de forma idempotente (reimportar não duplica) | M |
| RF-002 | Exportar os feeds ativos em OPML | M |
| RF-003 | Adicionar feed informando a URL do feed **ou** a URL do site, com autodescoberta (`<link rel="alternate">` e caminhos comuns) | M |
| RF-004 | Antes de ativar, exibir prévia com cobertura temporal observada, até 10 posts pontuados e nota média; identificar contagem parcial ou estimada e exigir confirmação; toda pontuação consome o orçamento de RF-021 | S |
| RF-005 | Ao ativar um feed, registrar os posts existentes como "já vistos" (linha de base), sem enviá-los | M |
| RF-006 | Desativar, remover e ajustar o peso a priori de um feed | M |
| RF-007 | Período de experiência (probation) configurável para feeds novos, com entrega apenas no digest e sugestão de desativar se a nota média for baixa | S |
| RF-008 | Listar feeds com volume observado e respectiva janela/cobertura, nota média e tamanho da amostra, além do estado (ativo, probation, pausado) | M |
| RF-009 | Silenciar temporariamente uma fonte | S |
| RF-010 | Pausar automaticamente feed com N falhas consecutivas e avisar o usuário | M |

### 5.2 Ingestão e deduplicação

| ID | Requisito | Prio. |
|---|---|:--:|
| RF-011 | Buscar feeds com GET condicional (`ETag`, `If-Modified-Since`) e backoff em caso de falha | M |
| RF-012 | Tentar extrair o texto completo quando o feed traz trecho; se indisponível, aplicar a política de conteúdo parcial de Q-14 e registrar a cobertura usada na pontuação | M |
| RF-013 | Identificar duplicatas por URL canonicalizada e hash de conteúdo normalizado, preservando as associações com as fontes; não usar texto vazio ou indisponível como evidência de igualdade | M |
| RF-014 | Detectar possíveis republicações/crossposts por similaridade de título, com limiar avaliado e fontes preservadas; similaridade lexical isolada não confirma identidade de evento | M |
| RF-015 | Agrupar posts do mesmo evento (cobertos por fontes diferentes) em um cluster, entregue como item único com a lista das demais fontes | M |
| RF-016 | Retomar o processamento por etapas persistentes e idempotentes após reinício; retries não duplicam registros ou efeitos já confirmados. Chamadas externas interrompidas seguem política explícita de resultado incerto (Q-24) | M |

### 5.3 Pontuação

| ID | Requisito | Prio. |
|---|---|:--:|
| RF-017 | Pontuar cada item de 1 a 5 com um LLM, devolvendo: nota, categoria, motivo (uma linha) e resumo curto em pt-BR, em JSON validado por schema | M |
| RF-018 | A rubrica de importância é um arquivo versionado no repositório; cada pontuação registra a versão da rubrica e o modelo usado | M |
| RF-019 | Cada fonte tem peso a priori que influencia a seleção | S |
| RF-020 | O provedor de LLM fica atrás de uma interface, trocável sem alterar o restante do sistema | M |
| RF-021 | Aplicar teto diário persistente, reservando custo máximo estimado antes de cada tentativa de LLM, inclusive scoring, clusters, prévias, resumos e retries. Sem saldo suficiente, reter ou recusar a operação e avisar, conforme Q-07/Q-27/Q-31 | M |
| RF-022 | Gerar, sob demanda, um resumo mais longo (~800 caracteres) de um item | S |

### 5.4 Entrega e interação (Telegram)

| ID | Requisito | Prio. |
|---|---|:--:|
| RF-023 | Enviar alerta para nota máxima com título, fontes, resumo, motivo e link, respeitando latência, probation, silenciamento e política de teto global (Q-12/Q-23) | M |
| RF-024 | Enviar digest em horário e fuso configuráveis, com até o teto de itens elegíveis (proposta: 5 a 7), sem mínimo obrigatório; tratar dias sem itens e reaparição de alertas conforme Q-05/Q-06/Q-23 | M |
| RF-025 | Todo item entregue contém o link da matéria completa; o texto integral **não** é reenviado | M |
| RF-026 | Oferecer ações inequívocas por item: 👍, 👎, 🔖 e “Mais detalhes”; apresentação em mensagem única ou múltipla e edição/expansão dependem do experimento de Q-26 | M |
| RF-027 | Não reenviar automaticamente entregas já confirmadas; persistir identidade do item, destino, modalidade e estado de envio. Envios de resultado incerto não recebem retry cego; política de recuperação e repetição alerta/digest depende de Q-23/Q-24/Q-25 | M |
| RF-028 | Responder apenas a IDs de usuário autorizados; ignorar os demais sem resposta | M |
| RF-029 | Disponibilizar os comandos candidatos `/addfeed`, `/feeds`, `/disable`, `/remove`, `/weight`, `/digest`, `/mute`, `/stats`, `/rubrica`, `/import`, `/export`, `/buscar`; priorização final em Q-29, consistente com RF-009/019/031/032 | M |
| RF-030 | `/import` aceita arquivo `.opml` anexado; `/export` devolve o OPML como anexo | M |
| RF-031 | `/buscar termo` pesquisa o arquivo (inclusive itens descartados) por texto | S |
| RF-032 | `/stats` mostra gasto de LLM (dia e mês), itens entregues e taxa de 👍/👎 | S |

### 5.5 Aprendizado

| ID | Requisito | Prio. |
|---|---|:--:|
| RF-033 | Registrar feedback vinculado ao item entregue, à nota e à versão da rubrica; definir alteração de voto e distinguir bookmark de avaliação positiva (Q-30) | M |
| RF-034 | Propor exemplos de feedback para few-shot; aplicar somente após aprovação, revisão de dados enviados e versionamento da rubrica; feedback não altera automaticamente critérios em produção | S |
| RF-035 | Manter cerca de 50 posts rotulados e casos de agrupamento para medir precisão, recall e falsos merges/separações; mudanças de rubrica/modelo executam regressão contra conjunto de avaliação separado dos exemplos few-shot (Q-04/Q-28) | M |

### 5.6 Operação

| ID | Requisito | Prio. |
|---|---|:--:|
| RF-036 | Oferecer na mesma imagem `serve`, `run-once`, `migrate`, `dry-run` e `healthcheck`. `serve` mantém a interação do bot; `run-once` encerra após um ciclo e não substitui o bot contínuo. Efeitos permitidos e eventual gasto de `dry-run` dependem de Q-32 | M |
| RF-037 | Impedir processadores e consumidores concorrentes incompatíveis sobre o mesmo estado por coordenação no Postgres; delimitar locks do ciclo e do polling se forem processos separados (Q-32) | M |
| RF-038 | Registrar último ciclo concluído, última entrega esperada e falhas; avisar quando o processo estiver operacional. Detecção da parada total depende de supervisor externo e da política de Q-33 | M |
| RF-039 | Aplicar migrações de schema de forma versionada e repetível | M |

---

## 6. Requisitos não funcionais

| ID | Categoria | Requisito |
|---|---|---|
| RNF-001 | Portabilidade | Uma imagem com variantes amd64/arm64 verificadas em execução; host e recursos mínimos dependem de Q-10. Não prometer compatibilidade com todo serviço de hospedagem sem teste |
| RNF-002 | Portabilidade | Postgres externo configurado por `DATABASE_URL`; um `docker-compose.yml` sobe app e banco para uso rápido |
| RNF-003 | Segurança | O container **não expõe nenhuma porta**; só conexões de saída (Telegram, API do LLM, feeds, Postgres) |
| RNF-004 | Segurança | Segredos apenas por variável de ambiente; nunca na imagem nem no repositório |
| RNF-005 | Segurança | Autorização por lista de IDs de usuário do Telegram (`TELEGRAM_ALLOWED_USER_IDS`) |
| RNF-006 | Segurança | Tratar feeds como dados não confiáveis; LLM sem ferramentas, saída validada e IDs conferidos contra a entrada. Schema não garante fidelidade ou eliminação de prompt injection |
| RNF-007 | Segurança | Imagem executa com usuário não-root |
| RNF-008 | Confiabilidade | Reinícios/retries preservam estado e não repetem efeitos confirmados; resultado externo incerto recebe tratamento explícito, sem promessa irrestrita de exactly-once (Q-24) |
| RNF-009 | Confiabilidade | Retry com backoff para falhas classificadas como recuperáveis, respeitando limites de taxa, orçamento e política de envio incerto |
| RNF-010 | Confiabilidade | Falha de um feed ou de uma chamada de LLM não interrompe o ciclo dos demais |
| RNF-011 | Custo | Teto diário preventivo aplicado a todas as chamadas/tentativas; critério de cobrança estimada versus faturada, margem e reconciliação definidos em Q-27 |
| RNF-012 | Custo | Uso de modelo pequeno por padrão para pontuação; modelo maior apenas se justificado por medição |
| RNF-013 | Desempenho | Latência entre a publicação de um post e o alerta de nota máxima: a definir (ver Q-12) |
| RNF-014 | Observabilidade | Logs estruturados; heartbeat consultável; falhas de feed registradas por feed |
| RNF-015 | Observabilidade | Registrar operação, tentativas, tokens conhecidos/estimados, preços utilizados, reservas e custo no Postgres; manter chamadas interrompidas como pendentes de reconciliação |
| RNF-016 | Qualidade | Testes de integração contra Postgres real; fixtures de feeds; conjunto rotulado como teste de regressão do ranking |
| RNF-017 | Qualidade | Versionar rubrica e OPML inicial no repositório; banco é fonte dos feeds em operação, com exportação e histórico auditável de mudanças. Não exigir commit automático a cada comando do bot (DEC-07/Q-34) |
| RNF-018 | Manutenibilidade | Arquivos Python com no máximo 300 linhas, conforme padrão do usuário |
| RNF-019 | Privacidade | Persistir dados no banco do usuário; limitar envio ao LLM a conteúdo, rubrica e exemplos aprovados necessários. Não enviar IDs Telegram, segredos ou histórico completo de interações; política final em Q-18/Q-30 |
| RNF-020 | Usabilidade | Mensagens em pt-BR com acentuação correta; resumos legíveis em tela de celular |
| RNF-022 | Multiplataforma | Disponibilizar leitura, feedback e gestão nos dispositivos aprovados em Q-21/Q-22, com estado único no servidor; Telegram é a proposta de interface da primeira versão |
| RNF-023 | Multiplataforma | A primeira versão proposta funciona sem VPN, app próprio ou porta pública de entrada. Front próprio é uma alternativa de escopo, cuja aprovação exige revisar as restrições correspondentes |
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

Extensão candidata: `pg_trgm` para similaridade lexical de títulos. O schema é conceitual, não uma decisão de implementação. Deve permitir associação de fontes sem apagar entradas, histórico de etapas, versões da rubrica, reservas de custo e envios `pendente`/`confirmado`/`incerto`. A identidade de entrega depende de Q-23/Q-25; `UNIQUE` no banco não equivale a transação atômica com o Telegram.

---

## 8. Deduplicação proposta e limites

1. **Exata:** URL canonicalizada e hash normalizado identificam repetição; preservar autoria/origem e não confundir múltiplas fontes com múltiplas entregas.
2. **Quase igual:** similaridade lexical de títulos gera candidatos a republicação; limiar e janela precisam de avaliação.
3. **Mesmo evento:** agrupamento deve preservar fontes e separar notícias diferentes sobre o mesmo tema. LLM sem embeddings é a direção herdada de DEC-05; algoritmo e janela ainda não estão fechados (Q-25).

Aprovar esta estratégia depende de fixtures com duplicatas exatas, títulos parecidos sobre eventos diferentes e títulos distintos sobre o mesmo evento. Não transformar “mesmo assunto” automaticamente em “mesma notícia”.

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

### 9.2 Diretriz herdada, sujeita ao fechamento de escopo
**Telegram como interface única da primeira versão**, usando *long polling* (`getUpdates`) em vez de webhook,
o que elimina URL pública, certificado TLS e porta aberta. O app pode rodar atrás de NAT ou CGNAT.

### 9.3 Decisão adiada
Front-end somente leitura (histórico, busca, notas, itens descartados), publicado com `tailscale serve`, como
**fase opcional** após 2 a 3 semanas de uso real. O custo de adicionar o front depende das interfaces e consultas implementadas; não há garantia de ausência de retrabalho.
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
- **Digest (proposta: nota ≥ 4):** até o teto aprovado, com título/link, uma linha de resumo e ações por item. Mensagem única versus múltiplas permanece em Q-26; não há obrigação de preencher o teto.
- **Resumo, não íntegra:** a íntegra anularia o objetivo de reduzir volume, esbarra no limite de 4.096
  caracteres por mensagem do Telegram e republicaria conteúdo de terceiros. O texto completo fica no banco só
  para busca.
- **Instant View:** em sites compatíveis, o Telegram abre o link em leitura limpa dentro do app.

---

## 10. Diretrizes de arquitetura herdadas e recomendações em avaliação

As DEC-01 a DEC-11 abaixo foram registradas como decisões na v1 e são preservadas como referência. Sua compatibilidade com as pendências deve ser confirmada no fechamento; não são evidência de aprovação das novas redações desta v2.

| ID | Diretriz herdada | Justificativa / limite |
|---|---|---|
| DEC-01 | Ingestão própria; **sem Miniflux** | Com Postgres aceito e imagem única, um segundo sistema só acrescentaria manutenção a 9 posts/dia |
| DEC-02 | Postgres como fonte da verdade (inclusive feeds) | Mudanças feitas pelo Telegram precisam persistir; imagem sem estado local |
| DEC-03 | Imagem única multi-arch com modos de execução | Roda em qualquer lugar; mesma imagem para serviço contínuo e cron |
| DEC-04 | Telegram por long polling, sem portas abertas | Dispensa exposição de rede e TLS |
| DEC-05 | Deduplicação em três camadas sem embeddings | Volume baixo; evita segundo provedor |
| DEC-06 | LLM atrás de interface (`Scorer`); modelo pequeno por padrão | Troca de provedor sem reescrita; custo previsível |
| DEC-07 | Rubrica e OPML inicial versionados; feeds operacionais no banco | Proposta de conciliação com DEC-02; exportações e histórico conforme Q-34 |
| DEC-08 | LLM sem ferramentas, com saída em JSON validado | Mitiga prompt injection vindo de conteúdo de feeds |
| DEC-09 | Aprendizado por few-shot na rubrica, sem fine-tuning | Simplicidade e transparência |
| DEC-10 | Linguagem Python | Alinhado ao ecossistema do usuário |
| DEC-11 | Fonte da verdade da rubrica de importância fica no repositório; o gosto do usuário a refina por feedback | Evita calibragem implícita e opaca |

---

### 10.1 Caminho de construção ainda não decidido

O [discovery](investigacoes/discovery-github-feed-curator.md) recomenda composição com feedparser, Trafilatura, aiogram e Pydantic, avaliando Instructor conforme o provedor. Isso é uma recomendação, não uma dependência aprovada. WHOOPsClaw/Digest_ai_agent é candidato a experimento de adaptação; amb007/rss-bot e alxyrgin/news-digest servem como referências. Não há solução integralmente validada.

Comparar composição, adaptação de projeto existente e redução do escopo inicial em Q-29. Escolher uma biblioteca não comprova idempotência, orçamento, qualidade ou suporte multi-arch da aplicação final.

---

## 11. Sequência de validação proposta

Esta sequência substitui a ordem da v1 como proposta; não é plano de implementação aprovado.

| Etapa | Resultado e condição de avanço |
|---|---|
| 0 — Preparação das decisões | Identificar bloqueadores da seção 18 e fechar os necessários à próxima etapa; registrar responsáveis e experimentos autorizados. Decisões dependentes de medição permanecem abertas até o resultado |
| 1 — Ingestão e estado | Validar OPML, baseline, extração/fallback, identidade, recuperação e schema em Postgres real, sem entrega automática |
| 2 — Avaliação offline | Usuário aprova rubrica e rotula conjunto; medir relevância e clusters, custo e limiares antes de liberar seleção automática |
| 3 — Operação controlada | Validar autorização, callbacks, ledger/reservas, política de envio incerto, locks, migrações, healthcheck, backup/restauração e formato de mensagens antes de entrega real |
| 4 — Primeira versão utilizável | Integrar seleção e entrega; disponibilizar feedback e gestão essencial; verificar imagem no host e arquiteturas exigidos antes de afirmar portabilidade |
| 5 — Evolução por evidência | Avaliar prévia pontuada, probation, pesos, few-shot e front opcional a partir do uso real e das prioridades aprovadas |

Token do bot é necessário para testar interação real, inclusive antes da entrega automática. Chave de LLM é necessária para qualquer avaliação com API paga, incluindo offline, prévia e `dry-run` se autorizado. Fixtures e avaliação de formato podem ser locais. Nenhuma chamada paga ou alteração em banco foi autorizada por este documento; a execução segue a governança vigente.

---

## 12. Critérios de aceite de alto nível

- **CA-01:** Reimportar o mesmo OPML não altera a contagem de feeds.
- **CA-02:** Um post reapresentado com `utm_*` diferente reutiliza a mesma identidade; cobertura por outra fonte mantém a origem e não cria entrega repetida do evento.
- **CA-03:** Injetar falhas antes do envio, após confirmação e antes do registro; efeitos confirmados não se repetem e o caso ambíguo segue a política aprovada em Q-24. Aceite pendente dessa decisão.
- **CA-04:** No conjunto rotulado, um evento coberto por três fontes gera um item com as três fontes; casos sobre eventos diferentes não são agrupados só por terem tema semelhante. Metas em Q-28.
- **CA-05:** Ao ativar um feed novo, nenhum post antigo é enviado.
- **CA-06:** Mensagens de usuário não autorizado não recebem resposta.
- **CA-07:** O container sobe sem nenhuma porta publicada.
- **CA-08:** Se o saldo não cobre a reserva máxima da tentativa, nenhuma chamada de LLM começa, inclusive por retry, clusterização, prévia ou comando. Testar reinício, concorrência e virada do dia segundo Q-27.
- **CA-09:** `curator healthcheck` falha quando o último ciclo é mais antigo que o limite configurado.
- **CA-10:** Mudanças de rubrica/modelo executam avaliação de precisão, recall e agrupamento no conjunto separado de avaliação; promoção bloqueada se falhar nos limites aprovados em Q-28.
- **CA-11:** Todo item entregue contém o link da matéria original.
- **CA-12:** A mesma imagem executa em `amd64` e `arm64`.

---

## 13. Questões em aberto (pendentes de decisão)

Todas as questões abaixo estão **[NEEDS CLARIFICATION]**. Preservar esse estado na SPEC até haver resposta registrada. A coluna de propostas não autoriza assumir respostas nem executar ações dependentes. Algumas questões mantêm valores sugeridos na v1 apenas como ponto de discussão.

| ID | Questão | Impacto | Proposta para discussão (não aprovada) |
|---|---|---|---|
| Q-01 | Incluir o front somente leitura via Tailscale? Quando? | Escopo e fases | Fora da primeira versão; reavaliar após 2 a 3 semanas de uso |
| Q-02 | **Qual API de LLM** o usuário tem? Anthropic, OpenAI ou outra? | Primeira implementação do `Scorer` | Anthropic, com interface preparada para trocar |
| Q-03 | Qual o critério de importância em uma ou duas frases (o que interessa, o que não interessa)? | Rubrica v1 | Rubrica a ser redigida na etapa 2 a partir de exemplos |
| Q-04 | Quem rotula os ~50 posts da linha de base, e em que formato (planilha)? | Etapa 2 | Gerar a lista; usuário rotula relevância e eventos; duração depende da amostra |
| Q-05 | Horário e fuso do digest diário | RF-024 | Manhã, fuso `America/Sao_Paulo`; horário a definir |
| Q-06 | Teto de itens do digest e limiar de nota para o digest e o alerta | RF-023, RF-024 | 5 a 7 itens; alerta nota 5; digest nota ≥ 4 |
| Q-07 | Teto diário de gasto com LLM (valor) | RF-021 | A definir; sugestão de valor baixo, ajustável |
| Q-08 | Duração do período de experiência (probation) de feeds novos e critério de sugestão de desativar | RF-007 | 2 semanas; sugerir desativar se nota média < 2,5 |
| Q-09 | Quantas falhas consecutivas pausam um feed (N) | RF-010 | A definir |
| Q-10 | Onde hospedar e quais recursos/arquiteturas o host oferece? A informação de contratação da v1 está desatualizada e precisa de confirmação. | Operação e portabilidade, etapas 3/4 | VPS existente, a confirmar |
| Q-11 | Registry da imagem (GHCR, Docker Hub, privado) e visibilidade do repositório | Distribuição da primeira versão, etapa 4 | A definir |
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

### 13.1 Questões adicionais da v2

| ID | Decisão pendente | Tipo / bloqueia | Proposta para discussão |
|---|---|---|---|
| Q-23 | Alertas contam no teto diário? Podem reaparecer no digest? O que `/digest` manual faz? Como tratar dias sem itens? | Preferência; seleção/entrega | Contar eventos únicos no teto global, omitir alertas já confirmados do digest automático e não preencher vagas; avaliar se notícia urgente pode exceder o teto |
| Q-24 | Em envio incerto, priorizar evitar duplicata ou evitar perda? Haverá reenvio manual e reconciliação? | Preferência + experimento; entrega real | Reter como incerto e exigir ação explícita para reenviar; aceitar que isso pode omitir uma mensagem que não chegou |
| Q-25 | Qual janela identifica o mesmo evento? Como atualizar clusters e fontes após entrega e preservar sua identidade? | Técnica + preferência; dedup/entrega | Identidade estável do evento; manter fontes adicionadas no arquivo sem novo alerta por padrão; validar em amostra temporal |
| Q-26 | Digest único ou uma mensagem por item? Onde exibir “Mais detalhes” se ultrapassar o limite? | Experimento de UX; callbacks | Comparar sete itens com ações inequívocas; aceitar fallback a mensagem separada apenas se aprovado |
| Q-27 | Qual moeda, fonte de preços, margem, fuso e tratamento de tentativas sem uso retornado sustentam o teto? | Produto + técnica; toda chamada paga | Dia em America/Sao_Paulo, reserva conservadora persistente e reconciliação; distinguir custo estimado do faturado |
| Q-28 | Quais metas mínimas de recall/precisão e erro de agrupamento? Como separar exemplos de calibração e avaliação? | Usuário + experimento; promoção da rubrica | Rotular relevância e eventos; aprovar valores após medir a linha de base, sem inventar percentuais |
| Q-29 | Qual é o recorte inicial e a estratégia: composição ou adaptação? Quais comandos/recursos podem esperar? | Produto + técnica; plano de construção | Composição; manter ingestão, ranking, clusters, entrega, feedback e gestão essencial; negociar prévia, probation, pesos e few-shot |
| Q-30 | Bookmark influencia ranking? Voto pode ser alterado? Quais exemplos/preferências podem ir ao provedor? | Preferência + privacidade; feedback/few-shot | Bookmark independente de 👍; voto substituível e auditável; apenas exemplos aprovados e sem identificação pessoal |
| Q-31 | O que fazer com backlog após limite de custo: prazo de validade, ordem e limites no dia seguinte? | Preferência; custo/seleção | Reter e informar, reavaliar frescor antes de entregar e nunca compensar backlog com rajada acima do teto |
| Q-32 | Bot e ciclos rodam no mesmo processo? Qual alcance de `run-once`, locks e efeitos de `dry-run`? | Técnica; operação | `serve` contínuo integra bot e ciclos; `run-once` sem consumidor de comandos; `dry-run` sem envio e sem custo por padrão, com política de escrita ainda explícita |
| Q-33 | Quem detecta container/host fora do ar? Qual prazo de detecção e canal externo? | Operação; monitoramento | Supervisor externo ao processo; mecanismo/canal a definir sem presumir nova porta pública |
| Q-34 | Como rastrear mudanças de feeds sem transformar o bot em escritor de Git? | Técnica; rastreabilidade | OPML inicial versionado, histórico no banco e exportação sob demanda; snapshots no Git somente por ação explícita |

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

## 16. Referências do projeto

- [feeds-dados-ml-ia-curado.opml](feeds-dados-ml-ia-curado.opml): os 21 feeds atuais, em três categorias (Engenharia de Dados, Machine
  Learning, LLMs e Agentes).

- [Requisitos v1](requisitos-feed-curator.md): histórico preservado; não substituído automaticamente.
- [Discovery GitHub](investigacoes/discovery-github-feed-curator.md): evidências de soluções, limites e recomendações técnicas.
- [Prompt de avaliação](prompt-avaliacao-requisitos-feed-curator.md): roteiro para avaliação completa; ainda referencia a v1 intencionalmente para preservar o pedido original.

## 17. Rastreabilidade das alterações propostas

| Origem / IDs | Mudança nesta v2 | Estado |
|---|---|---|
| OBJ-01/02/05/07 | Teto sem mínimo artificial, qualidade mensurada e aprendizado sem promessa de melhora automática | Proposta; Q-23/Q-27/Q-28 |
| RF-004/008/012 | Histórico observado identificado e fallback de texto parcial | Proposta; Q-14 |
| RF-013/014/015/016/027; RNF-008/009 | Separar entrada, evento e entrega; preservar fontes; tratar efeitos confirmados e envio incerto | Proposta; Q-23/Q-24/Q-25 |
| RF-021; RNF-011/015 | Orçamento antes das tentativas, reservas persistentes e reconciliação | Proposta; Q-07/Q-27/Q-31 |
| RF-024/026/029/033 | Não forçar cinco itens; UX por item e prioridade de comandos/feedback explícitas | Proposta; Q-23/Q-26/Q-29/Q-30 |
| RF-034/035; RNF-019 | Few-shot aprovado, avaliação separada e política realista de dados ao LLM | Proposta; Q-18/Q-28/Q-30 |
| RF-036/037/038; RNF-001 | Distinguir ciclo de bot contínuo, alcance dos locks e monitoramento externo; testar portabilidade | Proposta; Q-10/Q-32/Q-33 |
| DEC-07; RNF-017 | Separar OPML inicial no Git dos feeds operacionais no banco | Proposta; Q-34 |
| Seções 7 a 11; CA-02/03/04/08/10 | Modelo conceitual, decisões herdadas e validações antes de operação real | Proposta; não é autorização para implementar |
| Demais IDs da v1 | Mantidos como candidatos, sem exclusão nem aprovação implícita | Revalidar no fechamento |

## 18. Condições para fechar os requisitos finais

1. Registrar escolhas de interface/plataformas e recorte inicial (Q-01/Q-21/Q-22/Q-29), identificando recursos adiados e suas dependências.
2. Aprovar critérios de importância, rotulagem, métricas e limites de regressão (Q-03/Q-04/Q-28); preservar conjunto de avaliação separado do few-shot.
3. Escolher provedor/modelo e política completa de custo e backlog (Q-02/Q-07/Q-19/Q-27/Q-31) antes de qualquer uso pago autorizado.
4. Definir teto de leitura, relação alerta/digest, identidade de evento e tratamento de entrega incerta (Q-06/Q-12/Q-13/Q-23/Q-24/Q-25).
5. Validar formato, ações e privacidade (Q-16/Q-18/Q-20/Q-26/Q-30), inclusive correspondência de feedback com cada item.
6. Fechar operação, retenção, backup/restauração e monitoramento (Q-05/Q-09/Q-10/Q-11/Q-15/Q-17/Q-32/Q-33/Q-34), antes de produção e conforme cada dependência. Parâmetros de probation (Q-08) só bloqueiam sua implementação se esse recurso for incluído.
7. Para cada questão, registrar resposta, responsável, data, justificativa e IDs afetados; questões condicionais podem ser adiadas junto com o recurso correspondente. Atualizar requisitos e critérios de aceite de forma consistente.
8. Registrar explicitamente aprovação do escopo resultante. Até lá, manter **rascunho** e não usar M/S/C ou recomendações do discovery como autorização de implementação.

Esta versão permite discutir decisões concretas sem apresentar a solução final como já definida.
