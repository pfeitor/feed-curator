# Prompt — Avaliação dos requisitos do Feed Curator

Avalie criticamente os dois documentos deste projeto antes de propor os requisitos finais do Feed Curator:

1. [docs/requisitos-feed-curator.md](requisitos-feed-curator.md)
2. [docs/investigacoes/discovery-github-feed-curator.md](investigacoes/discovery-github-feed-curator.md)

Os caminhos de entrada e de entrega indicados neste prompt são relativos à raiz do projeto; os links Markdown são relativos a este arquivo.

O objetivo é decidir o que realmente precisamos construir, o que podemos aproveitar de soluções existentes e quais requisitos precisam ser corrigidos, simplificados ou esclarecidos.

Siga a governança do workspace. Esta tarefa é de análise e documentação: não implemente código, instale dependências, execute projetos de terceiros ou altere os documentos originais.

## Princípios da avaliação

- Os requisitos atuais são um rascunho, não um contrato aprovado.
- As recomendações do discovery são hipóteses fundamentadas, não decisões finais.
- Diferencie necessidade do usuário, requisito de produto, decisão de arquitetura e preferência de implementação.
- Considere o contexto real: usuário único, 21 feeds e aproximadamente nove posts por dia.
- Não acrescente complexidade sem demonstrar qual problema ela resolve.
- Não confunda funcionalidade mencionada em README com capacidade verificada no código.
- Verifique nas fontes primárias as alegações externas que forem determinantes para a decisão.
- Distinga fatos, inferências e pontos ainda não comprovados.

Execute a análise nas etapas abaixo.

## 1. Coerência dos requisitos

Identifique contradições, ambiguidades, redundâncias, dependências e critérios de aceite não verificáveis. Examine especialmente:

- Garantia de não duplicar entregas diante de falhas entre Telegram e Postgres.
- Alerta imediato versus digest: um item alertado pode reaparecer no digest?
- Teto diário de leitura: inclui alertas ou apenas o digest?
- Digest único com botões por item e expansão de resumos.
- Teto de custo abrangendo retries, agrupamento, prévias e comandos.
- Compatibilidade entre operação contínua, run-once e interação por long polling.
- Monitoramento quando o próprio processo ou host estiver indisponível.
- Contagem de 30 dias quando o feed não oferece esse histórico.
- Feed gerenciado no banco versus lista de feeds versionada no repositório.
- Privacidade versus envio de preferências e exemplos de feedback ao LLM.
- Ordem das fases: quais proteções precisam existir antes da primeira entrega real?
- Precisão, recall e qualidade do agrupamento: quais métricas e critérios faltam?

Para cada achado, cite os IDs e as seções envolvidos, explique o impacto e proponha uma redação corrigida.

## 2. Reaproveitamento de soluções

Compare três caminhos:

- A. Adaptar uma aplicação existente.
- B. Compor bibliotecas e desenvolver o domínio específico.
- C. Reduzir o escopo inicial para validar o valor da curadoria.

Avalie aderência funcional, esforço de adaptação, manutenção, licenças, portabilidade e confiabilidade. Não produza estimativas numéricas de esforço sem base.

Indique o que pode ser adotado diretamente, o que serve apenas como referência e o que precisa ser desenvolvido. Explique quais evidências poderiam mudar sua recomendação.

## 3. Escopo mínimo e evolução

Classifique os requisitos existentes como:

- Manter na primeira versão.
- Simplificar.
- Adiar.
- Remover.
- Esclarecer antes da implementação.

Preserve os IDs para rastreabilidade. Para novos requisitos, use identificadores provisórios.

Separe funcionalidades essenciais para validar a curadoria das garantias mínimas necessárias para operar com segurança e confiabilidade. Não adie uma proteção obrigatória para depois da funcionalidade que depende dela.

## 4. Decisões pendentes

Reavalie as questões Q-01 a Q-22. Indique quais:

- Podem ser resolvidas pelos documentos.
- São decisões técnicas reversíveis.
- Exigem um experimento.
- Dependem da preferência do usuário.

Faça apenas perguntas que possam mudar materialmente o escopo ou a arquitetura. Agrupe-as por prioridade e apresente opções, recomendação e consequência de cada escolha. Não trate ausência de resposta como aprovação.

## Entrega

Entregue um único relatório em `docs/investigacoes/avaliacao-requisitos-feed-curator.md` contendo:

- Recomendação principal e sua justificativa.
- Achados priorizados por impacto.
- Matriz de disposição dos requisitos.
- Comparação dos caminhos de construção.
- Escopo recomendado para a primeira versão.
- Propostas de redação para requisitos problemáticos.
- Critérios de aceite mensuráveis e experimentos necessários.
- Decisões que ainda dependem do usuário.

Conclua indicando o que já pode ser consolidado e o que impede fechar os requisitos finais. Não declare requisitos aprovados nem altere o documento original nesta etapa.
