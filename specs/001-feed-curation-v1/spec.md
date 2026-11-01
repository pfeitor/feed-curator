# Feature Specification: Feed Curator V1 — Curadoria diária de feeds

**Feature Branch**: `001-feed-curation-v1` *(identificador da feature; nenhuma branch git criada — o diretório não é um repositório git)*

**Created**: 2026-10-31

**Status**: Draft

**Input**: User description: "carregue a skill speckit-specify do workspace. Preciso criar uma SPEC de uma solução descrita no documento @docs/requisitos-feed-curator-v3.md"

> **Documento de origem**: `docs/requisitos-feed-curator-v3.md` (v3.0-draft, pós-council). Esta spec cobre a **V1 mínima recomendada** (§3.1 do documento). Itens adiados (§3.2) e exclusões (§3.3) estão em *Fora de escopo* nas Assumptions. O documento de origem afirma que seus requisitos ainda são rascunho e que a §19 deve ser cumprida antes de originar uma SPEC; esta spec foi criada por pedido explícito do usuário e **trata as propostas ainda não aprovadas como premissas** (ver Assumptions), mantendo as questões que de fato bloqueiam o escopo como pendências.

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

### User Story 2 - Validar a curadoria contra exemplos rotulados antes de ligar a seleção automática (Priority: P1)

Antes de qualquer envio real, o usuário rotula um conjunto de publicações (≈50) e casos de repetição de evento. O sistema pontua esse conjunto com a rubrica versionada e reporta relevância (recall nos itens raros, precisão), custo e comparação contra uma regra simples. O resultado decide se a seleção automática pode ser ligada ou se o projeto deve manter apenas a poda manual no leitor atual.

**Why this priority**: O documento de origem condiciona a V1 à prova de que a pontuação por LLM supera uma regra simples; sem isso, o digest pode esconder justamente o que importa (risco de falso negativo).

**Independent Test**: Com um conjunto rotulado fixo, rodar a avaliação offline e obter o relatório de recall/precisão/custo e o veredito go/no-go, sem enviar nenhuma mensagem ao canal de entrega.

**Acceptance Scenarios**:

1. **Given** um conjunto rotulado separado do conjunto usado para exemplos, **When** a avaliação roda com uma rubrica e um modelo, **Then** o relatório informa recall (com destaque nos itens raros), precisão e custo total, identificando versão da rubrica e modelo.
2. **Given** uma rubrica ou modelo novo, **When** a avaliação roda no mesmo conjunto, **Then** o resultado é comparável ao anterior e a promoção para uso real fica bloqueada se os resultados estiverem fora dos limites aprovados.
3. **Given** o resultado da avaliação, **When** ele não supera a regra simples no critério de desistência definido, **Then** o relatório indica "no-go" e a seleção automática permanece desligada.

---

### User Story 3 - Receber um digest diário curto com o que importa (Priority: P1)

Todas as manhãs, no horário e fuso configurados, o usuário recebe no canal de entrega um digest com os itens mais relevantes do período, até um teto configurável, sem preencher vagas com conteúdo irrelevante. Cada item traz título, link para o original, resumo curto em português, motivo da nota e ações de feedback. Em dia sem itens relevantes, o digest não é preenchido artificialmente.

**Why this priority**: É o valor central: reduzir o volume diário de leitura sem perder o raro.

**Independent Test**: Com feeds ativos e a seleção automática aprovada (US2), aguardar o horário do digest (ou disparar manualmente) e conferir: quantidade ≤ teto, todos com link original, nenhum item repetido de digest anterior, nenhum item de linha de base.

**Acceptance Scenarios**:

1. **Given** vários itens pontuados no período, **When** chega o horário do digest, **Then** a mensagem contém no máximo o teto configurado de itens, ordenados por relevância, todos acima do limiar de nota.
2. **Given** menos itens acima do limiar do que o teto, **When** o digest é montado, **Then** apenas os itens acima do limiar são incluídos (sem mínimo obrigatório).
3. **Given** nenhum item acima do limiar, **When** chega o horário, **Then** o sistema segue a política definida para dias vazios (por padrão, não envia digest de itens) e isso fica registrado.
4. **Given** um item já entregue com confirmação, **When** o próximo digest é montado, **Then** esse item não é reenviado.
5. **Given** um item cujo resumo exceda o limite de tamanho do canal, **When** a mensagem é montada, **Then** o conteúdo é encurtado ou dividido sem perder o link original nem as ações.

---

### User Story 4 - Operar com custo e entrega sob controle (Priority: P1)

O usuário define um teto diário de gasto com o provedor de pontuação. O sistema reserva custo antes de cada tentativa e, sem saldo, retém o trabalho e avisa em vez de gastar. Se um envio terminar de forma ambígua (não se sabe se a mensagem chegou), o sistema marca a entrega como "incerta" e não reenvia às cegas. Falhas no meio de um ciclo são retomadas sem repetir efeitos já confirmados. O estado do sistema é consultável e há aviso se os ciclos pararem.

**Why this priority**: O documento de origem exige essas proteções (custo, idempotência, bloqueio de concorrência, saúde, backup) antes do primeiro envio real; sem elas, a V1 pode gastar além do teto ou duplicar/perder mensagens.

**Independent Test**: Simular teto esgotado, queda no meio do envio, reinício durante o ciclo e dois processadores concorrentes; verificar que nenhuma tentativa paga começa sem reserva, que confirmados não se repetem e que o estado de saúde reflete o atraso.

**Acceptance Scenarios**:

1. **Given** o teto diário esgotado, **When** há itens aguardando pontuação, **Then** nenhuma chamada paga é iniciada, o trabalho fica retido e o usuário é avisado.
2. **Given** um envio interrompido antes do registro de confirmação, **When** o sistema reinicia, **Then** a entrega fica em estado "incerta" e só é reenviada por ação explícita do usuário.
3. **Given** uma falha após a confirmação de entrega, **When** o ciclo é retomado, **Then** os itens confirmados não são reenviados.
4. **Given** dois processadores iniciados ao mesmo tempo, **When** ambos tentam atuar, **Then** apenas um processa o mesmo trabalho.
5. **Given** o último ciclo concluído mais antigo que o limite configurado, **When** a verificação de saúde é consultada, **Then** ela reporta falha.
6. **Given** a virada do dia (no fuso configurado), **When** o novo dia começa, **Then** o consumo do teto reinicia e o trabalho retido segue a política de backlog definida.

---

### User Story 5 - Dar feedback e gerenciar feeds pelo celular (Priority: P2)

O usuário reage a cada item entregue (útil, não útil, guardar para depois) e administra feeds por comandos no canal de entrega, a partir de qualquer dispositivo. Reações ficam vinculadas ao item, à nota atribuída e à versão da rubrica; o voto pode ser substituído e é auditável. "Guardar" não conta como "útil". Apenas usuários autorizados são atendidos; qualquer outro é ignorado sem resposta.

**Why this priority**: Gera o sinal para refinar a rubrica e permite operar sem terminal, mas a V1 já entrega valor sem refino automático.

**Independent Test**: Reagir a itens de um digest e executar comandos de gestão; conferir registro vinculado, substituição de voto e que um usuário não autorizado não recebe resposta.

**Acceptance Scenarios**:

1. **Given** um item entregue, **When** o usuário marca "útil", **Then** o feedback é registrado vinculado ao item, à nota e à versão da rubrica.
2. **Given** um voto registrado, **When** o usuário muda de opinião, **Then** o novo voto substitui o anterior e o histórico permanece auditável.
3. **Given** um item marcado como "guardar", **When** o feedback é consultado, **Then** ele consta como guardado e **não** como "útil".
4. **Given** uma mensagem de usuário não autorizado, **When** o sistema a recebe, **Then** não responde nem altera estado.
5. **Given** o comando de adicionar, listar, desativar, remover ou ajustar peso de feed, **When** executado pelo usuário autorizado, **Then** a mudança é refletida e confirmada ao usuário.

---

### User Story 6 - Levar meu ambiente para outro lugar (Priority: P3)

O usuário exporta a lista de feeds ativos em OPML a qualquer momento, importa um OPML pelo canal de entrega e consegue restaurar o sistema em outro host a partir de um backup do banco mais as variáveis de ambiente.

**Why this priority**: Evita aprisionamento e protege contra perda de dados, mas não bloqueia o valor central.

**Independent Test**: Exportar o OPML, importá-lo em instalação limpa e comparar; restaurar um backup em ambiente novo e verificar feeds, histórico de entregas e feedback.

**Acceptance Scenarios**:

1. **Given** feeds ativos, **When** o usuário pede a exportação, **Then** recebe um arquivo OPML com todos os feeds ativos.
2. **Given** um arquivo OPML enviado pelo canal de entrega, **When** o usuário o importa, **Then** os feeds são adicionados de forma idempotente.
3. **Given** um backup válido, **When** restaurado em ambiente novo, **Then** feeds, entregas e feedback reaparecem íntegros.

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

## Requirements *(mandatory)*

### Functional Requirements

**Gestão de feeds**

- **FR-001**: O sistema MUST importar feeds de um arquivo OPML de forma idempotente (reimportar não altera contagem nem duplica feeds).
- **FR-002**: O sistema MUST exportar os feeds ativos em OPML a qualquer momento.
- **FR-003**: O usuário MUST poder adicionar um feed informando a URL do feed ou do site, com descoberta automática do feed.
- **FR-004**: Ao ativar um feed, o sistema MUST registrar todas as publicações existentes como "já vistas", sem entregá-las.
- **FR-005**: O usuário MUST poder desativar, remover e ajustar o peso a priori de um feed (o peso é apenas registrado nesta versão e não influencia a nota).
- **FR-006**: O usuário MUST poder listar os feeds com volume, janela observada, nota média, amostra e estado.
- **FR-007**: O sistema MUST pausar um feed após um número configurável de falhas consecutivas e avisar o usuário.

**Ingestão e deduplicação**

- **FR-008**: O sistema MUST coletar feeds de forma econômica (reaproveitando validadores de cache do servidor) e com espera crescente em caso de falha.
- **FR-009**: O sistema MUST tentar obter o texto completo de cada publicação; se indisponível, MUST usar o trecho disponível, marcá-lo como "parcial" e registrar a cobertura de texto usada.
- **FR-010**: O sistema MUST identificar entradas por URL canonicalizada e hash de conteúdo normalizado, preservando todas as fontes de origem; conteúdo vazio ou indisponível não prova igualdade.
- **FR-011**: A falha de um feed ou de uma chamada individual MUST NOT interromper o ciclo dos demais.
- **FR-012**: O processamento MUST ser retomável por etapas persistentes e idempotentes; efeitos externos já confirmados MUST NOT ser repetidos após reinício.

**Pontuação e curadoria**

- **FR-013**: O sistema MUST atribuir a cada publicação elegível uma nota de 1 a 5, uma categoria, um motivo de uma linha e um resumo curto em português do Brasil, em formato estruturado e validado.
- **FR-014**: A rubrica de importância MUST ser versionada; cada pontuação MUST registrar a versão da rubrica e o modelo utilizado.
- **FR-015**: O provedor de pontuação MUST ser substituível sem reescrita do restante do sistema.
- **FR-016**: O conteúdo dos feeds MUST ser tratado como não confiável: o provedor de pontuação não pode ter ferramentas/ações, a saída MUST ser validada e identificadores devolvidos MUST ser conferidos.
- **FR-017**: O sistema MUST permitir uma avaliação offline sobre um conjunto rotulado (separado do conjunto de exemplos), reportando recall (destacando itens raros), precisão e custo, e MUST bloquear a promoção de rubrica/modelo cujos resultados fiquem fora dos limites aprovados.
- **FR-018**: A seleção automática para entrega MUST permanecer desligada até a avaliação offline atender ao critério de aprovação. O critério go/no-go de desistência do scoring por LLM em favor de poda manual é [NEEDS CLARIFICATION: qual resultado mínimo nos itens rotulados faria desistir do scoring por LLM e manter apenas a poda manual no leitor atual — ex.: recall nos itens raros inferior ao de uma regra simples?].

**Custo**

- **FR-019**: O sistema MUST manter um teto diário de gasto persistente e MUST reservar o custo máximo estimado antes de cada tentativa paga (pontuação, retentativas, comandos); sem saldo suficiente, nenhuma tentativa começa.
- **FR-020**: Sem saldo, o sistema MUST reter o trabalho, informar o usuário e MUST NOT executar rajada compensatória no dia seguinte.
- **FR-021**: O sistema MUST registrar cada operação paga (tentativas, tokens estimados ou conhecidos, preços, reservas e custo); tentativas interrompidas MUST ficar pendentes de reconciliação.

**Entrega e interação**

- **FR-022**: O sistema MUST enviar um digest em horário e fuso configuráveis, contendo no máximo o teto configurável de itens, apenas os acima do limiar de nota, sem mínimo obrigatório e sem preencher vagas.
- **FR-023**: Todo item entregue MUST conter o link para o original; o texto completo MUST NOT ser reenviado.
- **FR-024**: Cada item MUST oferecer ações inequívocas de "útil", "não útil" e "guardar para depois"; mensagens e referências de ação MUST respeitar os limites do canal.
- **FR-025**: O sistema MUST NOT reenviar entregas já confirmadas; cada entrega MUST persistir item, destino, modalidade e estado (`pendente`, `confirmado` ou `incerto`).
- **FR-026**: Uma entrega em estado `incerto` MUST NOT receber retentativa automática; o reenvio MUST depender de ação explícita do usuário.
- **FR-027**: O sistema MUST atender somente usuários de uma lista de autorizados; demais mensagens MUST ser ignoradas sem resposta.
- **FR-028**: O usuário MUST poder gerenciar feeds e consultar estado por comandos no canal de entrega (adicionar, listar, desativar, remover, ajustar peso, importar e exportar OPML, consultar custo mínimo); o recorte final de comandos é o essencial listado aqui.

**Feedback**

- **FR-029**: O sistema MUST registrar o feedback vinculado ao item, à nota atribuída e à versão da rubrica; o voto MUST ser substituível e o histórico auditável.
- **FR-030**: "Guardar para depois" MUST NOT ser contabilizado como "útil" nem influenciar o ranking.
- **FR-031**: Alterações de rubrica, exemplos ou modelo em produção MUST depender de aprovação humana explícita; nenhuma alteração automática é permitida.

**Operação**

- **FR-032**: O sistema MUST impedir que processadores concorrentes executem o mesmo trabalho simultaneamente.
- **FR-033**: O sistema MUST registrar cada ciclo (início, fim, falhas por feed) e expor uma verificação de saúde que reporte falha quando o último ciclo concluído exceder o limite configurado; a detecção de parada total do processo é responsabilidade de um supervisor externo.
- **FR-034**: O sistema MUST oferecer um modo de execução sem envio e sem custo por padrão (simulação) e um modo de execução única que encerra sem consumir comandos do usuário.
- **FR-035**: A estrutura de dados MUST evoluir por migrações versionadas e repetíveis.
- **FR-036**: Segredos MUST ser fornecidos somente por variáveis de ambiente; o sistema MUST NOT abrir portas de entrada e MUST executar sem privilégios de administrador.
- **FR-037**: Ao provedor de pontuação MUST ser enviado somente conteúdo da publicação, rubrica e exemplos aprovados — nunca identificadores do usuário, segredos ou histórico de conversa.
- **FR-038**: O sistema MUST permitir backup e restauração verificada do estado persistente.

### Key Entities *(include if feature involves data)*

- **Feed**: fonte de publicações; atributos: URL, estado (ativo/pausado/removido), contagem de falhas, peso a priori, categoria, validadores de cache, data de ativação.
- **Entrada**: publicação coletada; URL canônica única, hash normalizado, título, texto, indicador de texto parcial, data, fontes de origem; pode estar marcada "já vista" (linha de base).
- **Pontuação**: nota 1–5, categoria, motivo, resumo, versão da rubrica, modelo e resposta bruta do provedor; vinculada a uma Entrada.
- **Entrega**: registro do envio de um item ao usuário: item, destino, modalidade (digest), estado (`pendente`/`confirmado`/`incerto`), identificador da mensagem, data.
- **Feedback**: reação do usuário (útil, não útil, guardar), vinculada a Entrega/Entrada, nota e versão da rubrica; histórico de substituições.
- **Registro de uso pago**: operação, tentativas, tokens, preços, reservas e custo por chamada e por dia.
- **Pulso de ciclo (heartbeat)**: último ciclo concluído e entrega esperada, para a verificação de saúde.
- **Conjunto rotulado**: publicações e casos de repetição com a classificação do usuário, usado exclusivamente na avaliação.
- **Rubrica**: critério versionado de importância (com exemplos aprovados), que origina as pontuações.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: O usuário lê o digest diário em no máximo 5 minutos e recebe no máximo o teto configurado de itens por dia (padrão 7), sem vagas preenchidas com conteúdo abaixo do limiar.
- **SC-002**: Reimportar o mesmo arquivo de feeds resulta em 0 feeds novos, e ativar um feed com histórico resulta em 0 publicações antigas entregues.
- **SC-003**: 0 itens com entrega confirmada são reenviados em qualquer cenário de reinício ou falha testado; 100% das entregas ambíguas ficam visíveis como "incertas" e nenhuma é reenviada sem ação explícita.
- **SC-004**: Nenhuma tentativa paga começa sem reserva suficiente do teto diário: 0 ocorrências de gasto acima do teto nos testes de esgotamento, reinício, concorrência e virada de dia.
- **SC-005**: No conjunto rotulado, o recall nos itens raros e a precisão atendem às metas aprovadas [NEEDS CLARIFICATION: metas numéricas de recall, precisão e erro de agrupamento não definidas — o documento de origem pede medir a linha de base primeiro, sem inventar percentuais; quais valores o usuário aceita como metas de aprovação?].
- **SC-006**: 100% dos itens entregues têm link funcional para o original e 100% das pontuações registram versão de rubrica e modelo.
- **SC-007**: Uma falha em um feed ou em uma chamada isolada não impede que os demais itens do ciclo sejam processados (0 ciclos abortados por falha isolada).
- **SC-008**: Se nenhum ciclo concluir dentro do limite configurado, o estado de saúde reporta falha na verificação seguinte; o usuário é avisado de feeds quebrados e de teto esgotado em até um ciclo.
- **SC-009**: Em ambiente novo, a restauração de um backup recupera 100% dos feeds, entregas e feedbacks, e o sistema inicia em menos de 15 minutos a partir dos arquivos de implantação e das variáveis de ambiente.
- **SC-010**: Mensagens de usuários não autorizados geram 0 respostas e 0 mudanças de estado.
- **SC-011**: A mesma imagem de execução funciona nas arquiteturas amd64 e arm64, verificada por execução.

## Assumptions

**Usuário e plataformas**

- Usuário único (desenvolvedor brasileiro) que lê no Android e quer acesso equivalente em mobile, web e desktop; multiusuário está fora do escopo.
- O canal de entrega é o Telegram, em conversa privada; "web" significa o cliente web do próprio Telegram (Q-21/Q-22 — proposta do documento de origem, ainda não aprovada). Uma interface web própria fica fora desta versão.

**Premissas assumidas a partir das propostas do documento (ainda a aprovar; ajustáveis por configuração)**

- Digest único por dia pela manhã em `America/Sao_Paulo`, teto de 5 a 7 itens (padrão 7) e limiar de nota ≥ 4 (Q-05/Q-06).
- Teto diário de gasto em valor baixo e ajustável, contado pelo dia em `America/Sao_Paulo`, com reserva conservadora (Q-07/Q-27); backlog retido sem rajada compensatória (Q-31).
- Dia vazio: não enviar digest de itens e registrar o motivo (Q-23).
- Entrega incerta: reter e exigir ação explícita para reenviar (Q-24).
- Publicação atualizada após a entrega é ignorada; só publicação nova gera item (Q-13).
- Texto parcial/paywall: pontuar o trecho disponível e marcar como parcial (Q-14).
- Resumos em português do Brasil, com link para o original (Q-20).
- Sem expiração inicial de texto e de itens descartados (Q-15); número de falhas para pausar um feed é configurável (Q-09).
- A importância (rubrica) será redigida com o usuário na etapa de avaliação, com exemplos (Q-03); os ≈50 itens e casos de agrupamento serão rotulados pelo usuário sobre listas geradas pelo sistema (Q-04).
- Os alvos de SC-001 (5 minutos de leitura) e SC-009 (15 minutos para subir em ambiente novo) são padrões propostos por esta spec, não valores do documento de origem; ajustáveis na clarificação.
- Backup do banco e supervisor externo de morte total são necessários, mas seus mecanismos serão definidos no planejamento (Q-17/Q-33).
- Rubrica e lista inicial de feeds ficam versionadas no repositório; o estado operacional de feeds fica no banco, com exportação OPML como instantâneo (Q-34).

**Restrições herdadas da base mantida (para planejamento)**

- Base de comparação do documento: Python, Postgres, Telegram por long polling, imagem única multi-arquitetura e provedor de LLM atrás de uma interface trocável, sem ferramentas e com saída validada. Essas escolhas são decisões já tomadas, registradas aqui para o planejamento; não fazem parte do comportamento especificado.
- Arquivos Python limitados a 300 linhas é convenção do usuário, não requisito funcional.

**Dependências**

- Chave de API do provedor de pontuação (necessária para qualquer avaliação paga) e token do bot do canal de entrega (necessário para interação real). Esta spec não autoriza chamadas pagas nem escrita em banco antes de aprovação explícita do usuário.
- Host/recursos de execução e registry de imagem ainda por decidir (Q-10/Q-11).

**Fora de escopo desta feature (adiados por evidência — §3.2/§3.3 do documento)**

- Alertas imediatos de nota máxima: só depois de recall comprovado e de fechados teto, latência e reaparição (Q-06/Q-12/Q-23).
- Agrupamento de eventos (cluster) e similaridade de títulos: só com limiar e janela avaliados (Q-25/Q-28); nesta versão há deduplicação exata e as repetições temáticas são medidas.
- Prévia pontuada de feed novo, período de observação automático (probation), silenciamento temporário, peso a priori influente, resumo longo sob demanda ("mais detalhes"), busca em itens descartados, estatísticas ricas, exemplos automáticos no prompt.
- Front-end web próprio (inclusive acesso por VPN privada), multiusuário, reenvio de texto integral, ajuste fino de modelo, fontes que não sejam RSS/Atom e tradução integral.
