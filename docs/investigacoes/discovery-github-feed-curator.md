# Discovery de soluções GitHub para o Feed Curator

Data: 2026-10-31. Base: [requisitos-feed-curator.md](../requisitos-feed-curator.md), rascunho de 2026-10-31.

## Recomendação

Construir uma aplicação pequena por composição de bibliotecas, usando os bots existentes como referência de produto e implementação. **Nenhum candidato examinado demonstrou cobrir integralmente os requisitos**, especialmente Postgres como único estado, operação sem portas, orçamento diário persistente, rubrica versionada e entrega resistente a falhas. Esta é uma conclusão da amostra pesquisada, não uma afirmação de inexistência de outra solução.

Priorizar **feedparser + Trafilatura + aiogram + Pydantic**, com persistência própria em Postgres. Avaliar **Instructor** dentro da interface `Scorer`, caso simplifique a troca de provedores. **WHOOPsClaw/Digest_ai_agent**, **amb007/rss-bot** e **alxyrgin/news-digest** são os três repositórios de aplicação que merecem leitura focal antes da implementação. Não adotar um deles como fork integral sem um experimento de aderência.

## Método e limites

- Pesquisa pública no GitHub por combinações de RSS, Atom, LLM, scoring, Telegram, digest, deduplicação e OPML; leitura de README, documentação oficial e trechos de código dos candidatos prioritários.
- Filtros: Python, aproximadamente nove posts/dia, usuário único, decisões DEC-01 a DEC-11 preservadas. Miniflux/FreshRSS, front próprio, embeddings e frameworks com servidores adicionais não são pressupostos da recomendação.
- Estado de manutenção consultado pela [API pública GitHub](https://docs.github.com/en/rest/repos/repos#get-a-repository): `archived`, `pushed_at` e licença detectada. `pushed_at` é sinal de atividade no repositório, não prova de qualidade, release estável ou data exata da última alteração no tronco.
- Não houve instalação, execução de código de terceiros, benchmark, chamada paga a LLM, escrita em banco ou alteração dos requisitos. Capacidades não verificadas permanecem lacunas.
- Links de branch e páginas `latest` podem mudar. Para os dois candidatos principais, os HEADs consultados foram: `amb007/rss-bot@10841e7c70a6639df67b5cba041b578dfde82fa8` e `WHOOPsClaw/Digest_ai_agent@3d527f7f7e68e1b3e20df4e2f1e20a5b445f6fcb`.
- As recomendações e avaliações de esforço são inferências técnicas a partir das fontes citadas; não constituem medição de esforço nem revisão completa de segurança.

## 1. Aplicações existentes

| Projeto | O que oferece e vale estudar | Desalinhamento com os requisitos | Decisão sugerida |
|---|---|---|---|
| [WHOOPsClaw/Digest_ai_agent](https://github.com/WHOOPsClaw/Digest_ai_agent) | Pipeline modular, abstração de LLM, fontes, ranking, feedback e dedup. Há opção Postgres no código. | Padrão SQLite, configuração/estado em arquivos, compose publica porta para callbacks; dedup de banco tem caminho fail-open. Não há evidência suficiente para certificar orçamento diário ou todas as garantias de entrega. | Melhor referência de decomposição e agrupamento; candidato a spike de adaptação, não adoção direta. |
| [amb007/rss-bot](https://github.com/amb007/rss-bot) | Produto muito próximo: RSS/Atom → nota por interesse → digest Telegram; inclui feedback e busca. | SQLite e arquivo principal de cerca de 3.064 linhas; foge de DEC-02 e RNF-018. Layout local e configuração por usuário exigiriam reestruturação. | Melhor referência de fluxo e experiência do bot. Reaproveitar ideias e trechos pequenos licenciados, não o arquivo inteiro. |
| [alxyrgin/news-digest](https://github.com/alxyrgin/news-digest) | aiogram, long polling, middleware de proprietário, bookmarks, reações, SQLAlchemy e Alembic. | Usa SQLite/aiosqlite; não confundir presença de `DATABASE_URL`/Alembic com suporte Postgres validado. Inclui Telegram como fonte e Telegraph, fora do escopo. | Boa referência de handlers, callbacks e organização da persistência. |
| [philoking/cruxwire](https://github.com/philoking/cruxwire) | Ranking, resumos, grupos de notícias e estado de leitura em interface própria. | Dashboard web e Ollama como dependência; direção de produto diferente da interface Telegram e do modelo pequeno via API. | Referência de apresentação dos clusters e critérios de relevância. |
| [soroush-thr/spectrum](https://github.com/soroush-thr/spectrum) | Pipeline Python RSS → resumo/digest → Telegram, com intenção explícita de controlar orçamento. | Execução local e acoplamento ao Perplexity; não demonstra bot operacional completo, Postgres ou feedback aprovado. | Referência secundária de orçamento e composição de digest. |
| [Dzarlax-AI/evening-news](https://github.com/Dzarlax-AI/evening-news) | Agregação, extração, categorização e entrega de notícias; projeto tem evolução arquitetural própria. | README/indexação de busca pode representar arquitetura anterior; árvore atual contém serviços Go. Front público/admin e fontes extras ampliam o escopo. Licença não identificada pela API. | Observar conceitos; não copiar código sem esclarecer licença e versão. |
| [HarrisHan/ai-daily-digest](https://github.com/HarrisHan/ai-daily-digest) | Fluxo editorial de coleta, scoring e resumo de blogs técnicos. | Skill de OpenClaw, não serviço autônomo Python/Postgres. Ausência de chave externa no README significa uso do LLM do agente, não custo de inferência inexistente. Licença não identificada pela API. | Apenas referência editorial e de seleção de fontes. |

### Evidências de código dos três principais

**WHOOPsClaw:** [storage.py](https://github.com/WHOOPsClaw/Digest_ai_agent/blob/3d527f7f7e68e1b3e20df4e2f1e20a5b445f6fcb/newsbrief/core/storage.py) usa SQLite por padrão e `psycopg2` quando a URL começa com `postgres`; isso comprova uma opção de conexão, não paridade operacional testada. [docker-compose.yml](https://github.com/WHOOPsClaw/Digest_ai_agent/blob/3d527f7f7e68e1b3e20df4e2f1e20a5b445f6fcb/docker-compose.yml) publica `127.0.0.1:8080`; [dedup.py](https://github.com/WHOOPsClaw/Digest_ai_agent/blob/3d527f7f7e68e1b3e20df4e2f1e20a5b445f6fcb/newsbrief/processing/dedup.py) devolve os artigos quando a leitura de dedup falha. Há ideias úteis em [semantic_dedup.py](https://github.com/WHOOPsClaw/Digest_ai_agent/blob/3d527f7f7e68e1b3e20df4e2f1e20a5b445f6fcb/newsbrief/processing/semantic_dedup.py), [events.py](https://github.com/WHOOPsClaw/Digest_ai_agent/blob/3d527f7f7e68e1b3e20df4e2f1e20a5b445f6fcb/newsbrief/processing/events.py) e na [interface LLM](https://github.com/WHOOPsClaw/Digest_ai_agent/blob/3d527f7f7e68e1b3e20df4e2f1e20a5b445f6fcb/newsbrief/llm/base.py). Adaptar a lógica ao modelo `entries`/`clusters`/`deliveries` e ao contrato de falhas do Curator.

**amb007:** [rss_bot.py](https://github.com/amb007/rss-bot/blob/10841e7c70a6639df67b5cba041b578dfde82fa8/rss_bot.py) reúne bot, SQLite, ranking e busca. A proximidade funcional não elimina o trabalho de separar módulos, introduzir migrações versionadas, orçamento e persistência Postgres. Seu [README](https://github.com/amb007/rss-bot/blob/10841e7c70a6639df67b5cba041b578dfde82fa8/README.md) também distingue digest de fila de artigos para feedback; a semântica de entrega precisa ser comparada com RF-027 antes de reutilizar.

**alxyrgin:** [main.py](https://github.com/alxyrgin/news-digest/blob/main/src/main.py) inicia `start_polling`; [auth.py](https://github.com/alxyrgin/news-digest/blob/main/src/bot/middlewares/auth.py) ignora silenciosamente mensagens/callbacks de outro proprietário. [database.py](https://github.com/alxyrgin/news-digest/blob/main/src/storage/database.py) e [README](https://github.com/alxyrgin/news-digest#readme) explicitam SQLite. [filter.py](https://github.com/alxyrgin/news-digest/blob/main/src/processor/filter.py) faz parsing JSON com valores de fallback; isso não substitui o schema estrito de RF-017. [token_manager.py](https://github.com/alxyrgin/news-digest/blob/main/src/processor/token_manager.py) conta, trunca e estima custo; isso não comprova um teto diário preventivo.

### Licença e sinais de manutenção das aplicações

Todos os projetos abaixo estavam com `archived=false` na consulta. Datas UTC de `pushed_at`; atividade recente não é garantia de manutenção futura. Cada nome aponta à evidência da API.

| Projeto | Licença detectada | `pushed_at` |
|---|---|---|
| [WHOOPsClaw/Digest_ai_agent](https://api.github.com/repos/WHOOPsClaw/Digest_ai_agent) | MIT | 2026-06-18 |
| [amb007/rss-bot](https://api.github.com/repos/amb007/rss-bot) | MIT | 2026-10-24 |
| [alxyrgin/news-digest](https://api.github.com/repos/alxyrgin/news-digest) | MIT | 2026-03-17 |
| [philoking/cruxwire](https://api.github.com/repos/philoking/cruxwire) | MIT | 2026-06-28 |
| [soroush-thr/spectrum](https://api.github.com/repos/soroush-thr/spectrum) | MIT | 2026-01-04 |
| [Dzarlax-AI/evening-news](https://api.github.com/repos/Dzarlax-AI/evening-news) | Não identificada | 2026-10-31 |
| [HarrisHan/ai-daily-digest](https://api.github.com/repos/HarrisHan/ai-daily-digest) | Não identificada | 2026-02-25 |

MIT permite considerar reutilização preservando os avisos exigidos. Nos casos sem licença identificada, o discovery não assume autorização de copiar ou redistribuir. Conferir sempre a licença do commit e das dependências escolhidas.

## 2. Bibliotecas que reduzem o trabalho

| Camada | Projeto e evidência | Ajuda concreta | O que continua próprio |
|---|---|---|---|
| Parsing RSS/Atom | [feedparser](https://github.com/kurtmckee/feedparser), [GET condicional](https://feedparser.readthedocs.io/en/latest/http-etag/) | Parsing tolerante de feeds; expõe suporte a ETag/Last-Modified e respostas 304. | Persistir validadores no Postgres, timeouts, backoff, pausa por falhas, idempotência e baseline. Preferir fetch HTTP controlado e parsing dos bytes para centralizar política de rede. |
| Descoberta de feed | [feedsearch-crawler](https://github.com/DBeath/feedsearch-crawler#readme) | Busca assíncrona, caminhos comuns via `try_urls`, limites de duração/profundidade e `output_opml`. | Confirmar candidatos, impedir ativação automática e aplicar limites de rede. README atual exige Python ≥3.12; conferir a versão fixada. |
| Extração do artigo | [Trafilatura](https://github.com/adbar/trafilatura) | Extrai texto e metadados de HTML. | Medir nos 21 feeds; fallback ao trecho, estado `parcial`, limite de tamanho e tratamento de páginas indisponíveis. Não garante acesso a paywall. |
| Telegram | [aiogram](https://github.com/aiogram/aiogram), [long polling](https://docs.aiogram.dev/en/latest/dispatcher/long_polling.html) | Async, handlers, callbacks e polling compatível com ausência de porta pública. | Autorização global por ID, estado persistente de diálogos, comandos, edição de digest, entrega e política de retries. |
| Telegram alternativo | [python-telegram-bot](https://github.com/python-telegram-bot/python-telegram-bot) | Framework estabelecido e usado no candidato amb007. | Mesmas responsabilidades de domínio. A [licença atual do tronco](https://github.com/python-telegram-bot/python-telegram-bot/blob/master/LICENSE) é GPL v3; escolher conscientemente versão e licença. Não assumir LGPL com base em referências antigas. |
| JSON de LLM | [Pydantic](https://github.com/pydantic/pydantic) + [Instructor](https://github.com/567-labs/instructor) | Schema de saída e validação; Instructor oferece adapters e retries de validação. | Rubrica versionada, `Scorer`, recusa de conteúdo inválido, contabilização de todas as tentativas e orçamento preventivo. Não habilitar ferramentas para o modelo. |
| LLM alternativo | [Pydantic AI](https://github.com/pydantic/pydantic-ai), [output](https://ai.pydantic.dev/output/) | Tipagem, provedores e saídas estruturadas em diferentes modos. | Evitar adotar runtime de agentes/ferramentas sem necessidade. Para nove posts/dia, Pydantic com SDK do primeiro provedor pode ser suficiente. |
| OPML | [listparser](https://github.com/kurtmckee/listparser) | Parser de listas de assinaturas OPML. | Exportação e round-trip de categorias; unicidade/importação no banco. O parser não transforma importação em operação idempotente. |
| Migrações | [Alembic](https://github.com/sqlalchemy/alembic) | Migrações versionadas se a aplicação optar por SQLAlchemy. | Schema, constraints, advisory lock e política de execução de migrations. |
| Avaliação da rubrica | [promptfoo](https://github.com/promptfoo/promptfoo) | Comparação de prompts/modelos e execução em CI. | Ground truth de relevância e grupos, cálculo de recall/precisão, proteção de custo e aprovação da rubrica. Opcional: adiciona ferramenta Node ao desenvolvimento; não precisa entrar na imagem de produção. |

**Escolha inicial sugerida:** aiogram para Telegram; feedparser para parsing; Trafilatura para extração; Pydantic para contratos. Decidir a inclusão de feedsearch-crawler, listparser e Instructor por um spike pequeno. Não usar simultaneamente dois frameworks de bot ou dois frameworks de LLM. A escolha do SDK inicial continua dependente de Q-02.

### Licenças e atividade das bibliotecas

Snapshot da API pública; todos os listados estavam não arquivados. Não é um inventário das licenças transitivas. Para feedparser, a API retorna `NOASSERTION`; o [arquivo LICENSE](https://github.com/kurtmckee/feedparser/blob/main/LICENSE) contém termos permissivos de estilo BSD com duas condições e seção própria para documentação. Isso não significa ausência de licença.

| Projeto / API | Licença observada | `pushed_at` UTC |
|---|---|---|
| [feedparser](https://api.github.com/repos/kurtmckee/feedparser) | BSD-style conforme LICENSE; API `NOASSERTION` | 2026-10-06 |
| [feedsearch-crawler](https://api.github.com/repos/DBeath/feedsearch-crawler) | MIT | 2026-10-29 |
| [Trafilatura](https://api.github.com/repos/adbar/trafilatura) | Apache-2.0 | 2026-10-30 |
| [aiogram](https://api.github.com/repos/aiogram/aiogram) | MIT | 2026-10-31 |
| [python-telegram-bot](https://api.github.com/repos/python-telegram-bot/python-telegram-bot) | GPL-3.0 | 2026-10-19 |
| [Pydantic](https://api.github.com/repos/pydantic/pydantic) | MIT | 2026-10-31 |
| [Instructor](https://api.github.com/repos/567-labs/instructor) | MIT | 2026-10-26 |
| [Pydantic AI](https://api.github.com/repos/pydantic/pydantic-ai) | MIT | 2026-10-31 |
| [listparser](https://api.github.com/repos/kurtmckee/listparser) | MIT | 2026-10-06 |
| [Alembic](https://api.github.com/repos/sqlalchemy/alembic) | MIT | 2026-10-30 |
| [promptfoo](https://api.github.com/repos/promptfoo/promptfoo) | MIT | 2026-10-31 |

## 3. Alternativas que não encaixam diretamente

[lemon24/reader](https://github.com/lemon24/reader) é uma biblioteca de leitor, não apenas parser; inclui persistência e recursos úteis de agregação. A [documentação](https://reader.readthedocs.io/en/latest/guide.html) informa SQLite como backend. Reaproveitá-la diretamente criaria outro estado ou exigiria adaptação de backend, contrariando DEC-02. Serve como referência de ingestão robusta; preferir feedparser para a solução atual. A [API](https://api.github.com/repos/lemon24/reader) registrou BSD-3-Clause, não arquivado e `pushed_at=2026-10-22`.

Miniflux/FreshRSS/Inoreader já foram rebaixados pelos requisitos; este discovery não recomenda reabrir DEC-01. Workflows n8n e skills OpenClaw podem mostrar a sequência editorial, mas introduzem outro runtime e não demonstram, por si, RF-016/RF-027/RF-037. Clustering por embeddings também não é necessário para o volume informado; preservar DEC-05 até existir medição que justifique revisão.

## 4. Mapa de requisitos para reaproveitamento

“Ajuda” significa bloco reutilizável ou referência, não atendimento completo certificado.

| Requisitos / decisões | Reaproveitamento | Lacuna a implementar e verificar |
|---|---|---|
| RF-001/002/030; DEC-07 | listparser; exportador OPML; exemplos do bot | Importação com unicidade, exportação dos feeds ativos, categorias e anexos autorizados. |
| RF-003/004/005 | feedsearch-crawler; handlers de fontes dos bots | Prévia com custo limitado, confirmação, baseline dos posts antigos e medição de 30 dias com indicação de amostragem. |
| RF-006 a RF-010 | Exemplos de menus/estado de fontes | Peso, probation, mute, pausa automática e métricas persistentes. |
| RF-011/012 | feedparser + fetch controlado + Trafilatura | Retry, validadores HTTP, marca de parcial e isolamento de falhas. |
| RF-013/014/016; DEC-02 | Canonicalização própria + constraints Postgres + `pg_trgm` | Identidade de entrada, URLs, hash, limiares e recuperação após reinício. |
| RF-015; DEC-05 | WHOOPsClaw como referência de grupos; saída LLM validada | Clusterização por evento sem embeddings, fontes preservadas, estabilidade dos IDs e avaliação de falsos merges. |
| RF-017/018/020; DEC-06/08/11 | Pydantic; SDK ou Instructor atrás de `Scorer` | Nota 1–5, pt-BR, versão da rubrica/modelo, recusas e tratamento de saída inválida. |
| RF-021/022/032; RNF-011/015 | Instrumentação de tokens; referências dos bots | Ledger Postgres e reserva preventiva de custo para toda chamada, incluindo retries e comandos. |
| RF-023 a RF-031; DEC-04 | aiogram; UX dos candidatos | Layout, allowlist em todos os handlers, busca, estado de entrega e edição. |
| RF-033/034/035; DEC-09 | Reações/bookmarks dos bots; promptfoo opcional | Feedback ligado à rubrica/nota, aprovação explícita de exemplos e dataset rotulado. |
| RF-036 a RF-039; DEC-03 | Alembic e padrões de CLI/container | Cinco modos, lock, heartbeat, healthcheck e imagem multi-arch sem portas. |

## 5. Pontos que o discovery expõe para a SPEC

1. **Entrega “nunca duas vezes” exige decisão sobre falha ambígua.** A assinatura documentada de [`sendMessage`](https://core.telegram.org/bots/api#sendmessage) não oferece chave de idempotência. Inferência: uma queda após o Telegram aceitar e antes de persistir `message_id` não é resolvida só com `UNIQUE`, outbox ou advisory lock. Definir estado de envio incerto e política de reconciliação/retry; não prometer exactly-once sem demonstrar como tratar essa janela.
2. **Digest único com botões por item precisa de protótipo de formato.** [`callback_data`](https://core.telegram.org/bots/api#inlinekeyboardbutton) aceita 1–64 bytes: usar referência curta ao item, consultada no banco. [`editMessageText`](https://core.telegram.org/bots/api#editmessagetext) edita a mensagem inteira e limita o texto a 4.096 caracteres após parsing. Testar sete itens e expansão de um resumo; diferenciar botões numerados para evitar feedback no item errado.
3. **Contagem de tokens não equivale a orçamento.** Reservar o pior caso antes de cada chamada, cobrindo scoring, agrupamento, prévia, “Mais detalhes” e retries. A [documentação do Instructor](https://python.useinstructor.com/concepts/retrying/) distingue tentativas de validação e retries do SDK; ambos precisam entrar na política. Persistir reservas/uso e definir virada do dia no fuso escolhido.
4. **Duplicidade de título e identidade de evento são problemas distintos.** [`pg_trgm`](https://www.postgresql.org/docs/current/pgtrgm.html) mede similaridade lexical. Não substitui RF-015. Rotular também pares/grupos de notícias para avaliar agrupamento, além de marcar artigos relevantes. Evitar apagar a segunda fonte apenas porque parece duplicata; ela deve continuar ligada ao cluster.
5. **Heartbeat interno não avisa morte total do próprio processo.** Inferência operacional: RF-038 exige verificação por supervisor/serviço externo se a expectativa inclui alertar quando o container ou host inteiro cair.
6. **JSON válido não comprova fidelidade nem elimina prompt injection.** Manter o modelo sem ferramentas, validar IDs de artigos/grupos contra a entrada e limitar tamanho. Avaliar resumo fiel e relevância no conjunto rotulado; não tratar nota alta ou schema válido como evidência suficiente de qualidade.
7. **Prévia de 30 dias pode ser só um limite inferior.** Muitos feeds retêm apenas as entradas recentes. A menos que exista histórico acumulado, RF-004 deve identificar cobertura/quantidade observada; não apresentar contagem parcial como volume exato mensal.

## 6. Composição versus fork

| Caminho | Benefício | Trabalho e risco residual | Parecer |
|---|---|---|---|
| Fork de amb007 | Experiência funcional muito próxima | Separar monólito e trocar persistência/configuração; adaptar semântica de entrega e ranking | Não priorizar como base integral. |
| Fork de WHOOPsClaw | Modularidade e opção Postgres já presentes | Remover escopo extra, estado em arquivos/portas; revisar falhas de dedup, migrations, orçamento e rastreabilidade | Único fork que merece spike se a intenção for acelerar por adaptação. |
| Fork de alxyrgin | Estrutura aiogram, ORM e botões próximos | Migrar SQLite, remover fontes/canais extras, endurecer schema/entrega/orçamento | Útil como referência de módulos, sem evidência de vantagem integral. |
| Composição de bibliotecas | Mantém DEC-01 a DEC-11 e aproveita parsing/extração/transporte existentes | Construir o domínio específico: rubrica, clusters, entregas, feedback e orçamento | **Recomendado para o volume e restrições atuais.** |

## 7. Experimentos curtos antes de fixar dependências

Os experimentos abaixo são propostas; não foram executados neste discovery.

1. **Ingestão e extração:** usar o OPML dos 21 feeds, salvar fixtures e medir parsing, GET condicional e extração em amostra de dez artigos com trechos. Resultado esperado: mapa de falhas/parciais e round-trip OPML; não ativar entregas nem chamar LLM nesta etapa.
2. **UX Telegram:** validar exemplos de alerta, digest de sete itens, botões, “Mais detalhes”, autorização e OPML em long polling. Usar texto fixo para separar formato da qualidade do modelo.
3. **Ranking e clusters:** rotular cerca de 50 posts mais pares/grupos do mesmo evento; comparar rubrica/modelo pequeno com regras simples. Medir recall, precisão, falsos merges, tokens e custo. Definir metas com o usuário antes de aprovar a rubrica.
4. **Falhas e orçamento:** simular restart entre etapas, entrega com resultado incerto, dois processos, limite diário e retries de validação. Só aprovar a arquitetura após documentar a política para cada resultado.
5. **Portabilidade:** após fixar versões, construir a mesma imagem em amd64/arm64, sem portas e como não-root. Verificar disponibilidade de wheels das dependências de extração; atividade do repositório não comprova compatibilidade multi-arch.

Entregável seguinte sugerido: incorporar à SPEC a seleção de bibliotecas, os limites de atendimento e as decisões pendentes. O discovery não substitui a rubrica, o conjunto rotulado ou a validação de confiabilidade exigidos pelos requisitos.
