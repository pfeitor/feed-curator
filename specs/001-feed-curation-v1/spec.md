# Feature Specification: Feed Curator V1 — Curadoria diária de feeds

**Feature Branch**: `001-feed-curation-v1` *(identificador da feature; branch única `main`, sem feature branch)*

**Created**: 2026-10-31

**Status**: Draft

**Input**: User description: "carregue a skill speckit-specify do workspace. Preciso criar uma SPEC de uma solução descrita no documento @docs/requisitos-feed-curator-v3.md"

> **Documento de origem**: `docs/requisitos-feed-curator-v3.md` (v3.0-draft, pós-council). Esta spec cobre a **V1 mínima recomendada** (§3.1 do documento). Itens adiados (§3.2) e exclusões (§3.3) estão em *Fora de escopo* nas Assumptions. O documento de origem afirma que seus requisitos ainda são rascunho e que a §19 deve ser cumprida antes de originar uma SPEC; esta spec foi criada por pedido explícito do usuário e **trata as propostas ainda não aprovadas como premissas** (ver Assumptions), mantendo como pendência apenas o que depende de preferência pessoal.

## Clarifications

### Session 2026-10-31 — deliberação do council (delegada pelo usuário)

As questões abaixo foram decididas por um council de 8 membros (Torvalds, Machiavelli, Watts, Sun Tzu, Aurelius, Feynman, Lao Tzu, Elon) em 3 rodadas, a pedido do usuário, por não dependerem de preferência pessoal. **Não são aprovação do usuário**: valores marcados como "provisório" aguardam confirmação (ver *Parâmetros dependentes do usuário* em Assumptions) e devem ser revisitados em uma nova rodada de `/speckit-clarify`.

- Q: Qual resultado faria desistir do scoring por LLM? (FR-018) → A: desistir se o recall nos itens raros do LLM for **menor ou igual** ao da regra simples no mesmo conjunto (empate = a regra fica) **ou** se o custo mensal projetado ultrapassar o orçamento do usuário. Resultado ambíguo mantém a regra e estende o período de observação. "Custo por item relevante" foi descartado como gatilho (8/8 no voto final).
- Q: Quais metas de aprovação? (SC-005) → A: critérios congelados em arquivo versionado **antes** de rodar o LLM; o recall nos itens raros é a única meta dura e é **relativa à regra simples** (sem percentual absoluto inventado); a precisão é só reportada, com teto de fadiga provisório de no máximo 2 itens ruins em 7. "Erro de agrupamento" saiu da meta (agrupamento está fora de escopo).
- Q: Como validar o LLM antes de entregar com base nele? → A: fluxo **híbrido** (8/8): triagem offline barata (~100–150 itens: união dos topos da regra e do LLM + amostra aleatória de controle, rotulagem cega à nota do modelo, ≥15 raros, contagem bruta + intervalo de Wilson, menos de 10 raros = inconclusivo) apenas para descartar o "claramente pior"; a decisão final vem de um **modo sombra** (shadow) sobre o digest real. Até a aprovação, o digest enviado é o da regra simples.
- Q: O que é indispensável antes do primeiro digest real? → A: teto de gasto com reserva prévia, simulação (dry-run), idempotência pela chave do digest do dia, bloqueio simples de concorrência, estado "incerto" com a intenção de envio gravada **antes** do envio, migrações por versão, cópia periódica do estado, e alerta "digest não enviado hoje" por canal independente do pipeline e comprovado derrubando o envio de propósito. **Adiados**: verificação de saúde sofisticada, restauração verificada de backup, modo contínuo (serve).
- Voto vencido registrado: shadow-first puro, sem triagem offline (Elon/Watts/Lao Tzu em parte). Se o usuário não marcar o digest com regularidade, a triagem offline volta a ser um gate de go-live (alerta do Feynman).

### Session 2026-10-31 — respostas do usuário

- Q: Com que regularidade você vai marcar o digest durante o modo sombra? → A: na maioria dos dias, mas não todos (opção B, recomendada). O modo sombra pode durar até 4 semanas, a triagem offline permanece leve (descarta só o "claramente pior") e o aviso de 7 dias sem marcações cobre as lacunas.
- Q: Como definir "raro / nunca poderia perder" para a avaliação? → A: o usuário escreve 1–2 frases de critério e 5–10 exemplos reais de itens que não poderia perder, antes da triagem; o sistema redige a rubrica a partir disso e o usuário a aprova (opção A, recomendada). Essa definição entra nos critérios congelados.
- Q: Quanto esforço de rotulagem você aceita para a triagem offline? → A: ~100–150 itens em 2 sessões (≈1 h), com meta de pelo menos 15 itens raros (opção A, recomendada). Se houver menos de 10 raros, o resultado é "inconclusivo" e o modo sombra completa a amostra.
- Q: Qual o máximo mensal que você aceita gastar com o provedor de LLM? → A: cerca de US$5 por mês, ≈ US$0,17 por dia (opção A, recomendada). Vale como orçamento do usuário em FR-018 e FR-019; o teto diário deriva dele.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Ingerir meus feeds sem reenviar o histórico (Priority: P1)

O usuário importa a lista de feeds que já acompanha (arquivo OPML) e adiciona novos feeds por URL do feed ou do site. O sistema passa a coletar as publicações novas de cada feed; tudo o que já existia no momento da ativação é registrado como "já visto" e nunca é entregue. Textos que só chegam parciais são marcados como parciais. Um feed que falha repetidamente é pausado com aviso, sem derrubar os demais.

**Why this priority**: Sem ingestão confiável e sem linha de base, qualquer etapa seguinte (pontuação, digest) opera sobre dados errados ou inunda o usuário com histórico. É a fundação e pode ser validada sem enviar nada.

**Independent Test**: Importar o OPML do usuário, rodar alguns ciclos de coleta com entrega desligada e conferir: contagem de feeds correta, nenhuma publicação antiga elegível à entrega, duplicatas exatas colapsadas, falha de um feed isolada.

**Acceptance Scenarios**:

1. **Given** um OPML com N feeds já importado, **When** o mesmo OPML é importado de novo, **Then** a contagem de feeds permanece N (importação idempotente).
2. **Given** um feed recém-ativado com publicações antigas, **When** o primeiro ciclo de coleta termina, **Then** todas as publicações existentes ficam registradas como "já vistas" e nenhuma é elegível à entrega.
3. **Given** duas publicações cuja URL difere só por parâmetros de rastreamento (ex.: `utm_*`), **When** ambas são coletadas, **Then** são tratadas como a mesma entrada, preservando ambas as origens quando vierem de fontes distintas.
4. **Given** um site que não é um feed, **When** o usuário o adiciona por URL, **Then** o sistema descobre o feed associado ou informa que não encontrou.
5. **Given** um feed que falha N vezes consecutivas, **When** o limite é atingido, **Then** o feed é pausado e o usuário é avisado; os demais feeds seguem normalmente.
6. **Given** uma publicação cujo texto completo não pôde ser obtido, **When** ela é processada, **Then** é marcada como "parcial" e a cobertura de texto usada fica registrada.

---

### User Story 2 - Validar a curadoria por LLM contra a regra simples antes de entregar com base nela (Priority: P1)

Antes de qualquer entrega escolhida pelo LLM, o usuário faz uma **triagem offline** (≈100–150 publicações rotuladas às cegas, com pelo menos 15 itens "raros") e depois mantém o LLM em **modo sombra** (2–4 semanas) pontuando os mesmos itens do digest real, que continua sendo montado pela regra simples (palavras-chave + peso por fonte). O usuário marca o digest ("valeu", "lixo", "faltou algo"). Os critérios de decisão foram congelados antes da execução. O resultado é um de três: aprovar o LLM, manter a regra, ou estender o modo sombra.

**Why this priority**: O documento de origem condiciona a V1 à prova de que a pontuação por LLM supera uma regra simples; sem isso, o digest pode esconder justamente o que importa (risco de falso negativo) e gastar dinheiro à toa.

**Independent Test**: Com um conjunto rotulado fixo e critérios congelados, rodar a triagem e obter o relatório de recall/precisão/custo do LLM e da regra na mesma amostra, sem enviar nada ao canal; depois ativar o modo sombra e verificar que o digest entregue segue sendo o da regra enquanto as notas do LLM são apenas registradas.

**Acceptance Scenarios**:

1. **Given** um conjunto rotulado com pelo menos 15 itens raros, **When** a triagem roda, **Then** o relatório mostra, para o LLM e para a regra simples na mesma amostra, o recall nos raros (contagem bruta "x de n" com intervalo de confiança), a precisão e o custo, identificando versão da rubrica e modelo.
2. **Given** menos de 10 itens raros no conjunto, **When** a triagem roda, **Then** o resultado é "inconclusivo", a regra permanece e o modo sombra é estendido.
3. **Given** recall do LLM nos raros menor ou igual ao da regra, **When** a decisão é calculada, **Then** o veredito é "manter a regra" e a seleção por LLM permanece desligada.
4. **Given** critérios de aprovação gravados em arquivo versionado, **When** o arquivo é alterado depois de o resultado do LLM ser conhecido, **Then** a promoção é bloqueada (o arquivo precisa ser anterior à execução).
5. **Given** o modo sombra ativo, **When** o digest da regra é enviado, **Then** o LLM pontua os mesmos itens mais uma amostra aleatória de controle sem alterar o que é entregue, e o usuário pode marcar cada item e o digest como um todo ("faltou algo").
6. **Given** 7 dias sem nenhuma marcação do usuário, **When** o prazo vence, **Then** o sistema avisa que o modo sombra não está gerando rótulos.

---

### User Story 3 - Receber um digest diário curto com o que importa (Priority: P1)

Todas as manhãs, no horário e fuso configurados, o usuário recebe no canal de entrega um digest com os itens mais relevantes do período, até um teto configurável, sem preencher vagas com conteúdo irrelevante. Cada item traz título, link para o original e ações de feedback; quando a seleção por LLM estiver aprovada, também traz resumo curto em português e motivo da nota. Em dia sem itens relevantes, o digest não é preenchido artificialmente.

**Why this priority**: É o valor central: reduzir o volume diário de leitura sem perder o raro.

**Independent Test**: Com feeds ativos, aguardar o horário do digest (ou disparar manualmente) e conferir: quantidade ≤ teto, todos com link original, nenhum item repetido de digest anterior, nenhum item de linha de base, e o modo de seleção em uso (regra ou LLM) registrado.

**Acceptance Scenarios**:

1. **Given** vários itens elegíveis no período, **When** chega o horário do digest, **Then** a mensagem contém no máximo o teto configurado de itens, ordenados por relevância pelo critério de seleção ativo (regra simples até a aprovação do LLM; notas do LLM depois), todos acima do limiar.
2. **Given** menos itens acima do limiar do que o teto, **When** o digest é montado, **Then** apenas os itens acima do limiar são incluídos (sem mínimo obrigatório).
3. **Given** nenhum item acima do limiar, **When** chega o horário, **Then** o sistema segue a política definida para dias vazios (por padrão, não envia digest de itens) e isso fica registrado.
4. **Given** um item já entregue com confirmação, **When** o próximo digest é montado, **Then** esse item não é reenviado.
5. **Given** um item cujo resumo exceda o limite de tamanho do canal, **When** a mensagem é montada, **Then** o conteúdo é encurtado ou dividido sem perder o link original nem as ações.

---

### User Story 4 - Operar com custo e entrega sob controle (Priority: P1)

O usuário define um orçamento mensal (e dele deriva um teto diário) de gasto com o provedor de pontuação. O sistema reserva custo antes de cada tentativa e, sem saldo, retém o trabalho e avisa em vez de gastar. Antes de enviar o digest, o sistema grava a intenção de envio; se o envio terminar de forma ambígua (não se sabe se a mensagem chegou), a entrega fica "incerta" e não é reenviada às cegas. Falhas no meio de um ciclo são retomadas sem repetir efeitos já confirmados. Se o digest do dia não chegar, o usuário recebe um alerta por um caminho independente do processo que envia.

**Why this priority**: O documento de origem exige essas proteções antes do primeiro envio real; sem elas, a V1 pode gastar além do teto ou duplicar/perder mensagens. O council reduziu o conjunto ao núcleo proporcional ao dano para um único usuário.

**Independent Test**: Simular teto esgotado, queda no meio do envio, reinício durante o ciclo e dois processadores concorrentes; derrubar o envio de propósito e verificar que o alerta chega; confirmar que nenhuma tentativa paga começa sem reserva e que confirmados não se repetem.

**Acceptance Scenarios**:

1. **Given** o teto diário esgotado, **When** há itens aguardando pontuação, **Then** nenhuma chamada paga é iniciada, o trabalho fica retido e o usuário é avisado.
2. **Given** um envio interrompido antes do registro de confirmação, **When** o sistema reinicia, **Then** a entrega fica em estado "incerta" e só é reenviada por ação explícita do usuário.
3. **Given** uma falha após a confirmação de entrega, **When** o ciclo é retomado, **Then** os itens confirmados não são reenviados.
4. **Given** dois processadores iniciados ao mesmo tempo, **When** ambos tentam atuar, **Then** apenas um processa o mesmo trabalho.
5. **Given** o envio do digest derrubado de propósito, **When** passa o horário esperado, **Then** o alerta "digest não enviado hoje" chega por um canal independente do processo de envio.
6. **Given** a virada do dia (no fuso configurado), **When** o novo dia começa, **Then** o consumo do teto diário reinicia e o trabalho retido segue a política de backlog definida.

---

### User Story 5 - Dar feedback e gerenciar feeds pelo celular (Priority: P2)

O usuário reage a cada item entregue (útil, não útil, guardar para depois), marca no digest "faltou algo" e administra feeds por comandos no canal de entrega, a partir de qualquer dispositivo, com o mínimo de toques. Reações ficam vinculadas ao item, à nota atribuída e à versão da rubrica; o voto pode ser substituído e é auditável. "Guardar" não conta como "útil". Apenas usuários autorizados são atendidos; qualquer outro é ignorado sem resposta. Feedback e comandos são processados ao menos a cada execução diária.

**Why this priority**: Gera o sinal que alimenta o modo sombra e o refino da rubrica, e permite operar sem terminal; a V1 entrega valor mesmo sem refino automático.

**Independent Test**: Reagir a itens de um digest e executar comandos de gestão; conferir registro vinculado, substituição de voto e que um usuário não autorizado não recebe resposta.

**Acceptance Scenarios**:

1. **Given** um item entregue, **When** o usuário marca "útil", **Then** o feedback é registrado vinculado ao item, à nota e à versão da rubrica.
2. **Given** um voto registrado, **When** o usuário muda de opinião, **Then** o novo voto substitui o anterior e o histórico permanece auditável.
3. **Given** um item marcado como "guardar", **When** o feedback é consultado, **Then** ele consta como guardado e **não** como "útil".
4. **Given** uma mensagem de usuário não autorizado, **When** o sistema a recebe, **Then** não responde nem altera estado.
5. **Given** o comando de adicionar, listar, desativar, remover ou ajustar peso de feed, **When** executado pelo usuário autorizado, **Then** a mudança é refletida e confirmada ao usuário.
6. **Given** um digest entregue, **When** o usuário marca "faltou algo", **Then** a marcação fica registrada para o dia e entra na avaliação do modo sombra.

---

### User Story 6 - Levar meu ambiente para outro lugar (Priority: P3)

O usuário exporta a lista de feeds ativos em OPML a qualquer momento, importa um OPML pelo canal de entrega e dispõe de uma cópia periódica do estado persistente para levar a outro host.

**Why this priority**: Evita aprisionamento e protege contra perda de dados, mas não bloqueia o valor central. A restauração verificada fica para depois da primeira versão utilizável.

**Independent Test**: Exportar o OPML, importá-lo em instalação limpa e comparar; confirmar que a cópia periódica do estado é gerada e legível.

**Acceptance Scenarios**:

1. **Given** feeds ativos, **When** o usuário pede a exportação, **Then** recebe um arquivo OPML com todos os feeds ativos.
2. **Given** um arquivo OPML enviado pelo canal de entrega, **When** o usuário o importa, **Then** os feeds são adicionados de forma idempotente.
3. **Given** o ciclo de cópia periódica, **When** ele roda, **Then** é gerada uma cópia datada do estado persistente (feeds, entregas, feedback), disponível para restauração manual.

---

### Edge Cases

- **Feed muda de URL, entra em redirecionamento ou serve conteúdo vazio**: o vazio ou a indisponibilidade não prova igualdade com outra entrada; a entrada não é tratada como duplicata por isso.
- **Publicação atualizada depois de entregue**: ignorada por padrão; apenas publicação nova gera item novo.
- **Texto pago/paywall ou trecho parcial**: pontuar o trecho disponível e marcar como parcial.
- **Saída inválida ou fora do formato pelo provedor de pontuação**: a tentativa é rejeitada e o item não é entregue com dados inválidos; custo da tentativa continua contabilizado.
- **Conteúdo de feed com instruções maliciosas (prompt injection)**: conteúdo é tratado como não confiável; a pontuação não pode disparar ações e identificadores devolvidos são conferidos.
- **Teto esgotado no meio de um lote**: itens restantes ficam retidos e visíveis como pendentes; sem rajada compensatória no dia seguinte.
- **Troca de rubrica ou modelo**: pontuações anteriores permanecem associadas à versão que as gerou.
- **Dia sem nenhum item relevante**: nenhuma vaga é preenchida; o motivo do não-envio é registrado.
- **Limite de tamanho de mensagem e de dados de ação do canal**: mensagens e referências de ação respeitam os limites do canal.
- **Mesmo evento coberto por várias fontes**: nesta versão só a deduplicação exata é tratada; repetições temáticas podem aparecer e são medidas, não agrupadas (ver Fora de escopo).
- **Reinício durante importação, coleta ou envio**: nenhuma etapa já confirmada é repetida.
- **Rate limit do canal de entrega**: envios entram em fila com espera crescente, sem perder itens.
- **Usuário não marca o digest no modo sombra**: após 7 dias sem marcações o sistema avisa; sem rótulos reais, a decisão cai de volta para a triagem offline como gate.
- **O alerta de "digest não enviado" depende do mesmo processo que falhou**: não é aceitável; o alerta precisa de caminho independente e comprovado.
- **Resultado da avaliação ambíguo ou com poucos raros**: nunca aprova o LLM; mantém a regra e estende o modo sombra.
- **Itens que a regra nunca mostra ao usuário**: não geram rótulo no uso real; a amostra aleatória de controle existe para encontrá-los.

## Requirements *(mandatory)*

### Functional Requirements

**Gestão de feeds**

- **FR-001**: O sistema MUST importar feeds de um arquivo OPML de forma idempotente (reimportar não altera contagem nem duplica feeds).
- **FR-002**: O sistema MUST exportar os feeds ativos em OPML a qualquer momento.
- **FR-003**: O usuário MUST poder adicionar um feed informando a URL do feed ou do site, com descoberta automática do feed.
- **FR-004**: Ao ativar um feed, o sistema MUST registrar todas as publicações existentes como "já vistas", sem entregá-las.
- **FR-005**: O usuário MUST poder desativar, remover e ajustar o peso a priori de um feed (o peso alimenta a regra simples; não influencia a nota do LLM).
- **FR-006**: O usuário MUST poder listar os feeds com volume, janela observada, nota média, amostra e estado.
- **FR-007**: O sistema MUST pausar um feed após um número configurável de falhas consecutivas e avisar o usuário.

**Ingestão e deduplicação**

- **FR-008**: O sistema MUST coletar feeds de forma econômica (reaproveitando validadores de cache do servidor) e com espera crescente em caso de falha.
- **FR-009**: O sistema MUST tentar obter o texto completo de cada publicação; se indisponível, MUST usar o trecho disponível, marcá-lo como "parcial" e registrar a cobertura de texto usada.
- **FR-010**: O sistema MUST identificar entradas por URL canonicalizada e hash de conteúdo normalizado, preservando todas as fontes de origem; conteúdo vazio ou indisponível não prova igualdade.
- **FR-011**: A falha de um feed ou de uma chamada individual MUST NOT interromper o ciclo dos demais.
- **FR-012**: O processamento MUST ser idempotente por chave do digest do dia e retomável por etapas persistentes; efeitos externos já confirmados MUST NOT ser repetidos após reinício.

**Pontuação e curadoria**

- **FR-013**: O sistema MUST atribuir a cada publicação elegível uma nota de 1 a 5, uma categoria, um motivo de uma linha e um resumo curto em português do Brasil, em formato estruturado e validado.
- **FR-014**: A rubrica de importância MUST ser versionada; cada pontuação MUST registrar a versão da rubrica e o modelo utilizado.
- **FR-015**: O provedor de pontuação MUST ser substituível sem reescrita do restante do sistema.
- **FR-016**: O conteúdo dos feeds MUST ser tratado como não confiável: o provedor de pontuação não pode ter ferramentas/ações, a saída MUST ser validada e identificadores devolvidos MUST ser conferidos.
- **FR-017**: O sistema MUST permitir uma triagem offline sobre um conjunto rotulado às cegas (≈100–150 itens; união dos topos da regra simples e do LLM mais amostra aleatória de controle; separado do conjunto de exemplos), reportando para o LLM e para a regra simples, na mesma amostra, o recall nos itens raros (contagem bruta e intervalo de confiança), a precisão e o custo. Com menos de 10 itens raros o resultado MUST ser "inconclusivo".
- **FR-018**: A seleção do digest pelo LLM MUST permanecer desligada até a aprovação. O sistema MUST recomendar **desistir do LLM e manter a regra** quando o recall do LLM nos itens raros for menor ou igual ao da regra simples na mesma amostra, ou quando o custo mensal projetado exceder o orçamento do usuário; resultado ambíguo MUST manter a regra e estender o modo sombra. Os critérios (incluindo a definição de "raro", escrita pelo usuário em 1–2 frases com 5–10 exemplos reais) MUST estar gravados em arquivo versionado **antes** da execução; alterá-los depois de conhecido o resultado MUST bloquear a promoção.

**Custo**

- **FR-019**: O sistema MUST manter um orçamento mensal de gasto (do qual deriva um teto diário persistente) e MUST reservar o custo máximo estimado antes de cada tentativa paga (pontuação, retentativas, comandos); sem saldo suficiente, nenhuma tentativa começa.
- **FR-020**: Sem saldo, o sistema MUST reter o trabalho, informar o usuário e MUST NOT executar rajada compensatória no dia seguinte.
- **FR-021**: O sistema MUST registrar cada operação paga (tentativas, tokens estimados ou conhecidos, preços, reservas e custo); tentativas interrompidas MUST ficar pendentes de reconciliação.

**Entrega e interação**

- **FR-022**: O sistema MUST enviar um digest em horário e fuso configuráveis, contendo no máximo o teto configurável de itens, apenas os acima do limiar, sem mínimo obrigatório e sem preencher vagas; a seleção MUST usar a regra simples até a aprovação do LLM e as notas do LLM depois.
- **FR-023**: Todo item entregue MUST conter o link para o original; o texto completo MUST NOT ser reenviado.
- **FR-024**: Cada item MUST oferecer ações inequívocas de "útil", "não útil" e "guardar para depois", com o mínimo de toques; mensagens e referências de ação MUST respeitar os limites do canal.
- **FR-025**: O sistema MUST NOT reenviar entregas já confirmadas; cada entrega MUST persistir item, destino, modalidade e estado (`pendente`, `confirmado` ou `incerto`), e a intenção de envio MUST ser gravada antes do envio.
- **FR-026**: Uma entrega em estado `incerto` MUST NOT receber retentativa automática; o reenvio MUST depender de ação explícita do usuário.
- **FR-027**: O sistema MUST atender somente usuários de uma lista de autorizados; demais mensagens MUST ser ignoradas sem resposta.
- **FR-028**: O usuário MUST poder gerenciar feeds e consultar estado por comandos no canal de entrega (adicionar, listar, desativar, remover, ajustar peso, importar e exportar OPML, consultar custo mínimo); o recorte final de comandos é o essencial listado aqui.

**Feedback**

- **FR-029**: O sistema MUST registrar o feedback vinculado ao item, à nota atribuída e à versão da rubrica; o voto MUST ser substituível e o histórico auditável.
- **FR-030**: "Guardar para depois" MUST NOT ser contabilizado como "útil" nem influenciar o ranking.
- **FR-031**: Alterações de rubrica, exemplos ou modelo em produção MUST depender de aprovação humana explícita; nenhuma alteração automática é permitida.

**Operação**

- **FR-032**: O sistema MUST impedir que processadores concorrentes executem o mesmo trabalho simultaneamente.
- **FR-033**: O sistema MUST registrar cada ciclo (início, fim, falhas por feed) e MUST emitir o alerta "digest não enviado hoje" quando o digest esperado não for confirmado, por um mecanismo **independente do processo que envia o digest**; o disparo do alerta MUST ser comprovado derrubando o envio de propósito antes do primeiro digest real. A detecção de parada total do processo é responsabilidade de um supervisor externo.
- **FR-034**: O sistema MUST oferecer um modo de simulação (sem envio e sem custo por padrão) e um modo de execução única diária. Feedback e comandos pendentes MUST ser processados ao menos a cada execução. O modo contínuo (serviço permanente) fica adiado.
- **FR-035**: A estrutura de dados MUST evoluir por migrações versionadas e repetíveis.
- **FR-036**: Segredos MUST ser fornecidos somente por variáveis de ambiente; o sistema MUST NOT abrir portas de entrada e MUST executar sem privilégios de administrador.
- **FR-037**: Ao provedor de pontuação MUST ser enviado somente conteúdo da publicação, rubrica e exemplos aprovados — nunca identificadores do usuário, segredos ou histórico de conversa.
- **FR-038**: O sistema MUST gerar periodicamente uma cópia datada do estado persistente; a restauração verificada é adiada para depois da primeira versão utilizável.

**Modo sombra**

- **FR-039**: O sistema MUST oferecer um modo sombra em que o LLM pontua os itens do digest real e uma amostra aleatória de controle sem alterar o que é entregue, registrando notas, custo e a seleção que a regra teria feito.
- **FR-040**: O usuário MUST poder marcar o digest como um todo ("faltou algo") além de cada item ("valeu"/"lixo" equivalem a útil/não útil); as marcações MUST alimentar a avaliação do modo sombra.
- **FR-041**: O sistema MUST avisar o usuário quando passarem 7 dias sem nenhuma marcação durante o modo sombra.
- **FR-042**: O sistema MUST registrar a decisão final do modo sombra (aprovar o LLM, manter a regra ou estender) com data, critérios usados e quem decidiu; o critério congelado decide, não a impressão posterior.

### Key Entities *(include if feature involves data)*

- **Feed**: fonte de publicações; atributos: URL, estado (ativo/pausado/removido), contagem de falhas, peso a priori, categoria, validadores de cache, data de ativação.
- **Entrada**: publicação coletada; URL canônica única, hash normalizado, título, texto, indicador de texto parcial, data, fontes de origem; pode estar marcada "já vista" (linha de base).
- **Pontuação**: nota 1–5, categoria, motivo, resumo, versão da rubrica, modelo e resposta bruta do provedor; vinculada a uma Entrada; pode ser de produção ou de modo sombra.
- **Entrega**: registro do envio de um item ao usuário: item, destino, modalidade (digest), estado (`pendente`/`confirmado`/`incerto`), intenção de envio, identificador da mensagem, data.
- **Feedback**: reação do usuário (útil, não útil, guardar, faltou algo), vinculada a Entrega/Entrada, nota e versão da rubrica; histórico de substituições.
- **Registro de uso pago**: operação, tentativas, tokens, preços, reservas e custo por chamada e por dia.
- **Registro de ciclo**: último ciclo concluído e entrega esperada, para o alerta de digest não enviado.
- **Conjunto rotulado**: publicações e casos de repetição com a classificação do usuário (às cegas), usado exclusivamente na avaliação; inclui amostra aleatória de controle.
- **Critérios congelados**: arquivo versionado e datado com as regras de decisão (aprovar / manter regra / estender), anterior à execução.
- **Rubrica**: critério versionado de importância (com exemplos aprovados), que origina as pontuações.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: O usuário lê o digest diário em no máximo 5 minutos e recebe no máximo o teto configurado de itens por dia (padrão 7), sem vagas preenchidas com conteúdo abaixo do limiar.
- **SC-002**: Reimportar o mesmo arquivo de feeds resulta em 0 feeds novos, e ativar um feed com histórico resulta em 0 publicações antigas entregues.
- **SC-003**: 0 itens com entrega confirmada são reenviados em qualquer cenário de reinício ou falha testado; 100% das entregas ambíguas ficam visíveis como "incertas" e nenhuma é reenviada sem ação explícita.
- **SC-004**: Nenhuma tentativa paga começa sem reserva suficiente do orçamento: 0 ocorrências de gasto acima do teto nos testes de esgotamento, reinício, concorrência e virada de dia.
- **SC-005**: Na triagem offline e no modo sombra, o LLM só é aprovado se o recall nos itens raros superar o da regra simples sob critérios congelados antes da execução; a precisão é reportada e comparada ao teto de fadiga provisório (≤ 2 itens ruins em 7). Com menos de 10 raros, o resultado é "inconclusivo" em 100% dos casos.
- **SC-006**: 100% dos itens entregues têm link funcional para o original e 100% das pontuações registram versão de rubrica e modelo.
- **SC-007**: Uma falha em um feed ou em uma chamada isolada não impede que os demais itens do ciclo sejam processados (0 ciclos abortados por falha isolada).
- **SC-008**: Quando o digest esperado não é confirmado, o usuário recebe o alerta por canal independente em até um ciclo, comprovado derrubando o envio de propósito; o usuário também é avisado de feeds quebrados e de teto esgotado.
- **SC-009**: Existe uma cópia datada do estado persistente gerada pelo menos uma vez ao dia, e o sistema inicia em menos de 15 minutos em ambiente novo a partir dos arquivos de implantação e das variáveis de ambiente.
- **SC-010**: Mensagens de usuários não autorizados geram 0 respostas e 0 mudanças de estado.
- **SC-011**: A mesma imagem de execução funciona nas arquiteturas amd64 e arm64, verificada por execução.
- **SC-012**: Ao fim do modo sombra (2–4 semanas) existe uma decisão registrada (aprovar o LLM, manter a regra ou estender) com data, critérios e rótulos de uso real; 0 entregas do período foram escolhidas pelo LLM antes da aprovação.

## Assumptions

**Usuário e plataformas**

- Usuário único (desenvolvedor brasileiro) que lê no Android e quer acesso equivalente em mobile, web e desktop; multiusuário está fora do escopo.
- O canal de entrega é o Telegram, em conversa privada só do usuário; "web" significa o cliente web do próprio Telegram (Q-21/Q-22 — proposta do documento de origem, ainda não aprovada). Uma interface web própria fica fora desta versão.

**Premissas assumidas a partir das propostas do documento (ainda a aprovar; ajustáveis por configuração)**

- Digest único por dia pela manhã em `America/Sao_Paulo`, teto de 5 a 7 itens (padrão 7) e limiar de nota ≥ 4 (Q-05/Q-06).
- Orçamento de gasto contado pelo dia em `America/Sao_Paulo`, com reserva conservadora (Q-07/Q-27); backlog retido sem rajada compensatória (Q-31).
- Dia vazio: não enviar digest de itens e registrar o motivo (Q-23).
- Entrega incerta: reter e exigir ação explícita para reenviar (Q-24).
- Publicação atualizada após a entrega é ignorada; só publicação nova gera item (Q-13).
- Texto parcial/paywall: pontuar o trecho disponível e marcar como parcial (Q-14).
- Resumos em português do Brasil, com link para o original (Q-20).
- Sem expiração inicial de texto e de itens descartados (Q-15); número de falhas para pausar um feed é configurável (Q-09).
- A importância (rubrica) será redigida com o usuário na etapa de avaliação, com exemplos (Q-03); os itens da triagem e os casos de repetição serão rotulados pelo usuário sobre listas geradas pelo sistema (Q-04).
- O mecanismo do supervisor externo e do canal independente do alerta serão definidos no planejamento (Q-17/Q-33).
- Rubrica e lista inicial de feeds ficam versionadas no repositório; o estado operacional de feeds fica no banco, com exportação OPML como instantâneo (Q-34).
- O council deliberou assumindo armazenamento leve (SQLite); o documento de origem mantém Postgres. A escolha de armazenamento está em análise no ADR `docs/adr/0001-armazenamento-do-estado.md` (status: proposta) e não altera os requisitos acima (bloqueio de concorrência, cópia periódica e migrações valem para qualquer um).

**Parâmetros dependentes do usuário (padrões provisórios do council; confirmar em nova rodada de `/speckit-clarify`)**

- Orçamento mensal de LLM: **~US$5/mês, confirmado pelo usuário** (≈ US$0,17 por dia); o teto diário deriva dele.
- Tolerância a itens ruins por digest: **≤ 2 em 7 provisório**; serve como teto de fadiga reportado, não como gate de aprovação.
- Definição de "raro / nunca poderia perder": **definida pelo usuário** em 1–2 frases e 5–10 exemplos reais, escritos antes da triagem e congelados; a aprovação do LLM continua relativa à regra simples, sem percentual absoluto de tolerância.
- Esforço de rotulagem: **~100–150 itens em duas sessões (≈1 h), confirmado pelo usuário**; meta de pelo menos 15 itens raros.
- Disposição de marcar o digest ("valeu/lixo/faltou algo"): **confirmada pelo usuário — na maioria dos dias, não todos**; modo sombra de 2 a até 4 semanas com triagem offline leve. Se isso mudar para "provavelmente não", a triagem offline volta a ser gate de go-live.
- Os alvos de SC-001 (5 minutos de leitura) e SC-009 (15 minutos para subir em ambiente novo) são padrões propostos por esta spec, não valores do documento de origem.

**Restrições herdadas da base mantida (para planejamento)**

- Base de comparação do documento: Python, Postgres, Telegram por long polling, imagem única multi-arquitetura e provedor de LLM atrás de uma interface trocável, sem ferramentas e com saída validada. Essas escolhas são decisões já tomadas, registradas aqui para o planejamento; não fazem parte do comportamento especificado.
- Arquivos Python limitados a 300 linhas é convenção do usuário, não requisito funcional.

**Dependências**

- Chave de API do provedor de pontuação (necessária para qualquer avaliação paga) e token do bot do canal de entrega (necessário para interação real). Esta spec não autoriza chamadas pagas nem escrita em banco antes de aprovação explícita do usuário.
- Host/recursos de execução e registry de imagem ainda por decidir (Q-10/Q-11).

**Fora de escopo desta feature (adiados por evidência — §3.2/§3.3 do documento e deliberação do council)**

- Alertas imediatos de nota máxima: só depois de recall comprovado e de fechados teto, latência e reaparição (Q-06/Q-12/Q-23).
- Agrupamento de eventos (cluster) e similaridade de títulos: só com limiar e janela avaliados (Q-25/Q-28); nesta versão há deduplicação exata e as repetições temáticas são medidas.
- Prévia pontuada de feed novo, período de observação automático (probation), silenciamento temporário, peso a priori influente, resumo longo sob demanda ("mais detalhes"), busca em itens descartados, estatísticas ricas, exemplos automáticos no prompt.
- Verificação de saúde sofisticada, restauração verificada de backup e modo contínuo (serviço permanente) — adiados para depois da primeira versão utilizável.
- Front-end web próprio (inclusive acesso por VPN privada), multiusuário, reenvio de texto integral, ajuste fino de modelo, fontes que não sejam RSS/Atom e tradução integral.
