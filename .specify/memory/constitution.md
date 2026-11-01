# Feed Curator Constitution

## Core Principles

### I. Curadoria Guiada por Evidência e Controle Humano (NON-NEGOTIABLE)

- Nenhuma seleção automática de digest por LLM, nova versão de rubrica, conjunto de exemplos
  few-shot ou funcionalidade de complexidade adicional (como agrupamento de eventos, alertas
  imediatos ou interface web própria) MUST entrar em produção sem aprovação humana explícita e
  evidência empírica contra uma regra simples determinística sob critérios congelados em arquivo
  versionado **antes** da execução.
- Quando o recall do LLM nos itens raros for menor ou igual ao da regra simples, quando a amostra
  contiver menos de 10 itens raros (`inconclusivo`) ou quando o custo mensal projetado exceder o
  orçamento definido pelo usuário, a regra simples MUST permanecer ativa.
- Ausência de resposta do usuário MUST NOT ser tratada como aprovação implícita para alterar
  requisitos, promover modelos, iniciar chamadas pagas fora de fluxos autorizados ou mutar estado
  operacional.
- **Racional**: O valor central do produto é reduzir o volume diário de leitura sem perder itens
  raros (falso negativo); inferência paga ou sofisticação que não supere comprovadamente uma regra
  léxica transparente gera custo e opacidade sem benefício.

### II. Proteção Preventiva de Custo, Estado Durável e Falha Segura

- Toda operação paga (pontuação em produção, modo sombra, triagem offline, retentativas de
  validação ou comandos) MUST reservar atomicamente o custo máximo estimado em banco antes de
  iniciar qualquer requisição de rede; sem saldo suficiente no teto diário/mensal persistente,
  nenhuma chamada paga MUST iniciar e o backlog retido MUST NOT gerar rajada compensatória na
  virada do dia.
- Todo envio ao canal de entrega MUST gravar intenção durável antes da chamada externa e registrar
  resultado ambíguo como estado `incerto`, proibindo retentativas automáticas cegas; entregas já
  confirmadas e publicações de linha de base (`"já vistas"` na ativação do feed) MUST NOT ser
  entregues novamente.
- Processadores concorrentes MUST ser bloqueados por exclusão mútua persistente, falhas isoladas
  em um feed ou publicação MUST NOT abortar o processamento dos demais itens do ciclo, e a ausência
  do digest esperado no dia MUST gerar alerta por mecanismo e credencial independentes do processo
  que envia o digest.
- **Racional**: Provedores externos de LLM e mensageria não oferecem transação atômica conjunta com
  o banco local; previsibilidade financeira e ausência de spam dependem de invariantes locais
  persistidos antes de cada efeito externo.

### III. Fronteira de Confiança Zero, Privacidade e Privilégio Mínimo

- Conteúdos externos (feeds RSS/Atom, páginas HTML, anexos OPML) e respostas de modelos MUST ser
  tratados como dados não confiáveis: chamadas ao LLM MUST operar com zero ferramentas/ações
  habilitadas, validação estrita de schema e conferência de identificadores; XML/OPML MUST ser
  processado com bloqueio de DTD/XXE (`defusedxml`); URLs coletadas MUST bloquear esquemas não
  HTTP(S) e endereços privados/loopback (anti-SSRF); e textos enviados ao canal MUST ter saída
  escapada.
- Segredos MUST ser injetados exclusivamente por variáveis de ambiente sem fallbacks literais no
  código (`fail-closed`) e segregados por papel operacional — o serviço principal da aplicação
  MUST NOT receber credenciais do repositório ou destino remoto de backup. Logs e payloads enviados
  a provedores de LLM MUST NOT incluir segredos, tokens, identificadores do usuário no canal ou
  histórico não revisado.
- Em produção, os containers MUST operar sem abrir nem publicar portas de entrada, sob usuário
  não-root e com sistema de arquivos raiz somente leitura (`read_only: true`), restringindo
  escritas ao volume de estado dedicado e a diretório temporário em memória (`/tmp` em `tmpfs`).
- **Racional**: O código reside em repositório público e ingere conteúdo arbitrário da internet em
  uma VPS compartilhada; defesa em profundidade impede escalada de prompt injection, XXE ou SSRF
  para o host ou para o armazenamento externo de backups.

### IV. Simplicidade Proporcional, Modularidade e Portabilidade de Estado

- O desenho do sistema MUST permanecer proporcional à escala real do problema (usuário único,
  operação enxuta em lote/CLI e composição de bibliotecas especializadas), rejeitando serviços
  extras sem necessidade demonstrada ou forks monolíticos difíceis de auditar.
- O acesso ao banco de estado MUST utilizar consultas parametrizadas neutras de motor via
  `DATABASE_URL` (SQLAlchemy Core) e migrações versionadas repetíveis (Alembic), assegurando
  portabilidade contínua por exportação OPML de feeds ativos, backup periódico cifrado fora do
  servidor acompanhado de manifesto (hash, revisão do esquema, contagens e rótulo de ambiente) e
  restauração exclusivamente por comando explícito e guiado (MUST NOT restaurar automaticamente ao
  detectar banco vazio).
- Arquivos de código Python SHOULD manter no máximo 300 linhas com responsabilidade única; desvios
  MUST ser justificados explicitamente quando a divisão prejudicar a clareza ou coesão do módulo.
- **Racional**: Baixo acoplamento de motor, módulos curtos e restauração verificada evitam
  aprisionamento de dados e permitem reconstruir o ambiente completo em menos de 15 minutos.

### V. Verificação Antes de Entrega e Higiene Reprodutível de Artefatos

- Nenhum envio real ou promoção arquitetural MUST ocorrer antes da comprovação prática dos
  invariantes críticos: reserva de orçamento sob concorrência, consistência de estado sob
  interrupção abrupta (`SIGKILL`), ida e volta íntegra de backup/restauração e disparo real do
  alerta independente provocado por falha deliberada do envio.
- A imagem de execução MUST ser gerada em build multi-stage reprodutível a partir de commit limpo
  do tronco e contexto filtrado (`.dockerignore`), contendo apenas o código de runtime, migrações e
  configurações operacionais necessárias — sem ambientes virtuais locais, caches, testes,
  documentação, especificações, segredos, bancos ou caminhos/identificadores da máquina de build.
- Cada build de imagem MUST ser validado por inspeção automatizada do filesystem exportado, teste
  de inicialização em raiz somente leitura, registro de tamanho dentro do teto estipulado e
  varredura de vulnerabilidades (bloqueando envio à produção se houver CVE crítico com correção
  disponível). Servidores simulados em testes locais MUST escutar apenas em `127.0.0.1` (MUST NOT
  usar `0.0.0.0`).
- **Racional**: Regras de segurança, confiabilidade e minimalismo de imagem degradam rapidamente se
  não forem impostas por verificações automatizadas determinísticas a cada build.

## Restrições Operacionais e Isolamento de Implantação

- **Isolamento em host compartilhado**: Ao compartilhar VPS com outros sistemas (como o stack de
  trading), o `feed-curator` MUST manter projeto Compose, rede de saída, volumes, credenciais de
  backup e limites de memória/CPU estritamente isolados; nenhuma rotina de deploy ou operação MUST
  escrever diretamente em `/var/lib/docker`, reutilizar bancos/redes vizinhos ou reiniciar serviços
  de outros projetos, e o carregamento de imagem MUST abortar se a margem livre de disco na VPS for
  insuficiente.
- **Entrega direta e reversível de imagem**: Imagens de produção MUST NOT ser construídas na VPS;
  a entrega direta por pacote comprimido MUST validar integridade em duas camadas (hash SHA-256 do
  arquivo e digest de configuração da imagem na origem e no destino) e manter a imagem anterior
  disponível localmente para rollback imediato.
- **Higiene de repositório público**: Bancos de dados, snapshots, backups, arquivos `.env`, dumps,
  conjuntos rotulados com anotações privadas e logs de execução MUST NOT ser versionados no Git.

## Fluxo de Desenvolvimento e Gates de Qualidade

- **Fluxo Spec Kit e ADRs**: Alterações de comportamento, contratos ou topologia MUST ser
  registradas primeiro nos artefatos da feature (`spec.md`, `plan.md`, `contracts/`) e, quando
  houver decisão arquitetural com alternativas relevantes, em `docs/adr/` antes da codificação.
- **Disciplina de escopo**: Funcionalidades marcadas como fora de escopo ou adiadas por evidência
  na especificação vigente MUST NOT ser introduzidas durante a implementação sem revisão formal da
  spec.
- **Gates de aceitação de código e release**: Toda entrega MUST passar por testes unitários, de
  contrato e de integração (`pytest`), análise estática de tipos e segurança, validação de árvore
  Git limpa e bateria de verificação da imagem (`--read-only`, inspeção de camadas/arquivos e
  varredura CVE) antes de ser transferida ao ambiente de produção.

## Governance

- **Autoridade e Precedência**: Esta Constituição prevalece sobre práticas informais, atalhos de
  implementação e artefatos derivados no repositório. Conflitos identificados em `/speckit-plan`,
  `/speckit-analyze` ou code review são bloqueantes (`CRITICAL`) e exigem adequação do plano,
  especificação ou código — nunca flexibilização silenciosa de um princípio.
- **Procedimento de Emenda**: Emendas a este documento exigem motivação explícita, aprovação direta
  do mantenedor do projeto, atualização da versão/data de emenda e verificação de impacto sobre as
  especificações, planos e ADRs ativos.
- **Política de Versionamento Semântico**:
  - `MAJOR`: Remoção, relaxamento ou redefinição incompatível de princípios ou proibições `MUST` /
    `MUST NOT`.
  - `MINOR`: Inclusão de novo princípio, nova seção normativa ou ampliação material de gates
    obrigatórios.
  - `PATCH`: Ajustes de redação, clareza ou formatação sem mudança semântica nas obrigações.
- **Revisão de Conformidade**: Todo plano de implementação MUST validar explicitamente os gates
  desta Constituição antes da Fase 0 e após o design da Fase 1, justificando em *Complexity
  Tracking* qualquer separação operacional exigida pelos requisitos.

**Version**: 1.0.0 | **Ratified**: 2026-11-01 | **Last Amended**: 2026-11-01
