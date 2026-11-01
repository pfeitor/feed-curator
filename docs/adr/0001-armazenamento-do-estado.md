---
title: "ADR-0001: Armazenamento do estado (SQLite × Postgres)"
status: proposed
date: 2026-10-31
revised: 2026-10-31 (após council neutro)
decision-makers: Paulo (a confirmar)
relates-to: specs/001-feed-curation-v1/spec.md, docs/requisitos-feed-curator-v3.md (DEC-02, RNF-002, RNF-016, RF-037)
accept-gates: Q-10 respondida; spike de ~1 h aprovado; caminho de backup confirmado
---

# ADR-0001: Armazenamento do estado (SQLite × Postgres)

## Status

**Proposta.** Nada foi aceito. Há um **padrão provisório** que só vira decisão depois de três condições de aceite (ver *Condições de aceite*). O council recomendou Postgres via `DATABASE_URL` enquanto o host era desconhecido; com as respostas do usuário (VPS com Docker e preferência por zero serviço extra), o padrão provisório passou a **A (SQLite em volume nomeado)**, mantendo o código neutro de motor. Revisado após um council neutro de 3 membros, com votos finais unânimes.

## Entradas do usuário (2026-10-31)

Respostas à Q-10, dadas após o council neutro:

- **Onde roda no dia a dia:** em uma **VPS própria, a mesma do stack de trading (ARPHost)**, com Docker, isolada por projeto Compose, volumes e limites próprios (NFR-014 da spec). Isso reforça a eliminação da opção C: nenhuma instância de banco do trading é reutilizada. A imagem é **gerada localmente (Docker/WSL) e transferida diretamente para a VPS**; o uso de um registro de imagens foi descartado pelo usuário (não é requisito). Precedente: SPEC-077 do projeto de trading (aprovada, fases pendentes); a SPEC-076 construía na VPS.
- **Banco:** a imagem "provisiona" o banco. Interpretação registrada: o **esquema é criado e migrado na subida do container**, mas o **arquivo do banco fica em volume nomeado**, nunca na camada da imagem (usuário concordou).
- **Preferência:** **zero serviço extra** (SQLite).
- **Backup e migração Windows → VPS:** a imagem inclui um **modo de backup** (serviço separado, mesma imagem, credenciais próprias), no padrão da SPEC-076 do projeto de trading (`docs/features/076-migracao-postgres-vps-arphost-backup-2026-10-15`). O backup fica atualizado durante os testes em Docker no Windows e é buscado na VPS. Respostas do usuário: **restore por comando explícito e guiado** (nunca automático por banco vazio) e **destino próprio do feed-curator** (pasta do Drive e repositório cifrado separados dos de trading).

Efeito: o ramo "com disco local e zero serviço extra" da árvore de decisão passa a valer, e o **padrão provisório muda de Postgres para A (SQLite em volume)**. As outras duas condições de aceite continuam abertas.

## Contexto

A spec `001-feed-curation-v1` ficou neutra quanto ao armazenamento. O documento de origem decidiu **Postgres** (DEC-02, RNF-002, RNF-016, RF-037), com o motivo declarado "mudanças feitas pelo Telegram precisam persistir; **imagem sem estado local**". Esse motivo depende do **host**, que ainda não foi decidido (Q-10).

**Fatos verificados nos documentos do projeto**

| Fato | Fonte |
|---|---|
| 21 feeds no OPML inicial | `docs/feeds-dados-ml-ia-curado.opml` |
| 265 posts em 30 dias (~9/dia) ⇒ ≈3,2 mil entradas/ano; com texto completo (estimativa de 20–50 KB por entrada), ≈65–160 MB/ano | `docs/requisitos-feed-curator.md:19` |
| Host, recursos e arquiteturas **não decididos** (Q-10) | spec, Assumptions |
| Sem porta exposta; só conexões de saída | RNF-003 |
| O usuário já opera Postgres em containers, com rotina de backup própria, em outro projeto; não opera SQLite como banco principal | `docker-compose` e `backup/pg_backup` do workspace de trading |

**O que a spec exige do armazenamento (qualquer motor):** reserva de custo atômica antes de cada chamada paga (FR-019); bloqueio contra processadores concorrentes (FR-032); intenção de envio durável gravada antes do envio e estados `pendente/confirmado/incerto` (FR-025/026); idempotência pela chave do digest do dia (FR-012); migrações versionadas (FR-035); cópia periódica do estado (FR-038). Dia de orçamento no fuso `America/Sao_Paulo`: lógica na aplicação.

**Implicação central:** com ~3 mil linhas por ano e um único escritor lógico, **os dois motores satisfazem os requisitos**. O motor não decide correção; decidem a **precondição de implantação** (host, disco, backup, serviço extra).

## Duas decisões separadas

O council apontou que o rascunho anterior misturava duas coisas. Elas são tratadas em separado:

1. **Decisão de motor (interface):** como o código acessa o banco.
2. **Decisão de implantação:** onde e como o banco roda (A, B1 ou B2). Esta é a que depende do host.

### Opções de implantação

| Opção | Descrição |
|---|---|
| **A** | SQLite em arquivo único (WAL, `synchronous=FULL`) em volume local persistente |
| **B1** | Postgres dedicado ao feed-curator, em container ao lado do app, com volume |
| **B2** | Postgres externo gerenciado, via `DATABASE_URL` |
| **C** | Postgres compartilhado com a infraestrutura de trading (mesma instância) |
| **D** | Suportar os dois motores em produção desde a V1 |

## Critérios eliminatórios e de desempate

Sem pontuação numérica: o rascunho anterior somava pesos escolhidos por quem já tendia a uma opção e o resultado mudava de 21 pontos para 5 ao retirar os critérios de inércia. Aqui cada linha é **sim/não** e aponta para algo **verificável**.

| Critério | Como verificar | A | B1 | B2 | C | D |
|---|---|:-:|:-:|:-:|:-:|:-:|
| Funciona em host **sem** disco local persistente? | Documentação de volumes do host (Q-10) | não | não | **sim** | depende | depende |
| Exige serviço adicional além do app? | Inspeção do compose | **não** | sim | sim (externo) | não (reuso) | sim |
| Backup coberto por rotina que o usuário já opera? | Ler o script de backup existente | não | **sim**, se no mesmo host | **não** | sim | parcial |
| Acopla falha, credencial e upgrade ao trading? | Topologia | não | não | não | **sim (elimina)** | não |
| Dobra a matriz de testes e o código por dialeto? | Contagem de suítes | não | não | não | não | **sim (elimina)** |
| Satisfaz os invariantes (reserva atômica, intenção durável)? | **Spike de ~1 h** | previsto sim | previsto sim | previsto sim | previsto sim | — |

**Eliminadas:** **C** (acopla o digest pessoal à plataforma de trading: raio de dano, upgrade e credenciais; se estiver em outro host, ainda expõe o Postgres de trading pela rede, contra RNF-003) e **D** (dobra a matriz de testes e esconde divergências de dialeto sem requisito que peça isso).

**Observações do council**
- A vantagem de **backup e familiaridade do Postgres só existe em B1** (mesmo host, mesmo ciclo de backup). **Em B2 ela desaparece**: o backup passa a ser outro procedimento novo, como seria o do SQLite.
- **`pg_trgm` não pesa na V1.** Similaridade de títulos está adiada (RF-014/015) e uma biblioteca em Python resolve se for aprovada.
- **Familiaridade e alinhamento com DEC-02 são inércia**, não propriedade técnica. Contam como desempate declarado, nunca como mérito.
- Migrar o dado entre motores a ~160 MB leva horas; o custo real está no **código** (dialeto), que o spike mede.

## Decisão (proposta)

**D-1. Interface de acesso (neutra de motor):** acesso via `DATABASE_URL`, SQLAlchemy Core + Alembic, esquema **sem recursos exclusivos de um motor** na V1, fuso e dia de orçamento na aplicação. Em produção roda **um** motor; o código não depende de qual.

**D-2. Implantação, em função do host (Q-10):**

| Situação do host | Escolha |
|---|---|
| Sem disco local persistente | **B2** (Postgres gerenciado) |
| Com disco local, e você quer **zero serviço extra** | **A** (SQLite) |
| Com disco local, e você quer reaproveitar a rotina de backup do Postgres | **B1** |
| Host ainda indefinido (**já não é o caso**) | Postgres como seguro provisório (B1 se houver disco, B2 se não) |

**Situação atual:** VPS com Docker e disco local, preferência por zero serviço extra ⇒ **A (SQLite) é o padrão provisório.** O Postgres permanece como saída documentada (ver *Gatilhos de revisão*); o código neutro de motor mantém essa saída barata.

## Condições de aceite (as três são necessárias)

1. **Q-10 respondida:** ✅ VPS própria com Docker (2026-10-31). Arquitetura da VPS confirmada em 2026-10-31 (`uname -m` = `x86_64`, amd64). Falta confirmar, na VPS, que o volume fica em disco local (não em montagem de rede).
2. **Spike de ~1 h aprovado**, nos dois motores, com SQLAlchemy Core:
   - 2 processos fazem 1000 reservas contra um teto; a soma **nunca** pode ultrapassá-lo.
   - `kill -9` no meio do commit da intenção de envio; o estado **nunca** pode ficar inconsistente (nem `confirmado` sem identificador de mensagem).
   - Medir quantas linhas divergem por dialeto.
3. **Caminho de backup confirmado e testado.** Caminho **definido** (2026-10-31), falta executá-lo:
   - **Geração:** API de backup do SQLite (ou `VACUUM INTO`), nunca `cp` com o processo ativo; arquivo mais manifesto (hash, revisão do esquema, contagens por tabela, rótulo de ambiente `dev`/`prod`).
   - **Transporte:** cifra com `restic` em repositório próprio e envio por `rclone` a uma pasta do Drive própria; na VPS, `fetch` do backup.
   - **Restore guiado:** comando explícito que confere hash, `integrity_check`, compatibilidade da revisão do esquema e contagens; recusa banco populado sem confirmação de sobrescrita; **nunca automático** quando o banco estiver vazio; exibe o rótulo de ambiente.
   - **Isolamento:** serviço separado, com credenciais próprias; o aplicativo não recebe as credenciais do Drive.
   - **Fora da imagem e do repositório:** o repositório é público (e a imagem pode ser compartilhada no futuro), então backup e banco nunca podem ser embutidos na imagem nem commitados.
   - **Acréscimo ao spike (SQLite):** ida e volta do backup, ou seja, gerar, restaurar num banco vazio e obter hash, integridade e contagens iguais.

**Hipótese registrada antes de rodar o spike (para o teste poder refutá-la):** os dois motores passam nos dois testes, e a divergência de dialeto fica restrita à camada de conexão e às configurações do motor. **Refutação:** qualquer estouro do teto, qualquer estado inconsistente após `kill -9`, ou divergência de dialeto que vaze para além da camada de acesso a dados **muda este ADR**.

## Consequências

**Do padrão provisório atual (A, SQLite em volume nomeado)**
- (+) Zero serviço extra, um arquivo, sem credenciais de banco, testes triviais, imagem menor (o driver já vem na biblioteca padrão).
- (−) Exige disco local persistente (não serve em rede nem em montagens do Windows/WSL2); backup é procedimento novo (agora definido: cópia cifrada fora da VPS, no padrão da SPEC-076); reabre DEC-02, RNF-002 e RNF-016 no documento de origem; a restauração verificada, antes adiada, volta à V1 numa versão mínima para viabilizar a migração Windows → VPS.

**Se a escolha passar a Postgres (B1/B2)**
- (+) Funciona em qualquer host; mantém DEC-02; testes contra o mesmo motor do documento de origem (RNF-016); reaproveita a rotina de backup em B1.
- (−) Mais um serviço, segredo e processo de upgrade para ~9 posts/dia; em B2, o backup deixa de ser o já operado.

## Gatilhos de revisão

- **Com data:** decidir quando o host (Q-10) for fixado; o ADR não deve ficar em "proposta" indefinidamente.
- **Para SQLite:** host único com disco local e a manutenção do Postgres se mostrar custo real nas primeiras semanas.
- **Para Postgres (se A for escolhido):** aprovação de RF-014/015 em escala; segundo escritor em outro host; front-end com escrita; erros `database is locked` recorrentes; necessidade de recuperação pontual.

## Parecer do conselho

| Fonte | Posição | Peso |
|---|---|---|
| Council do v3 (9 membros, debate formal) | **Postgres pela maioria**; minoria SQLite (Elon, Lao Tzu) | Alto |
| Council de planejamento desta sessão (8 membros) | Desenhou o núcleo de proteções **assumindo SQLite**, sem avaliar Postgres, a partir de enunciado que descrevia a minoria como "SQLite, cron, lote" | Baixo (não independente) |
| **Council neutro sobre este ADR** (Aristotle, Ada, Feynman; 3 rodadas, enunciado sem recomendação nem histórico) | **Unânime (3/3):** manter *proposta*; Postgres via `DATABASE_URL` como padrão provisório; A legítima se host com disco e preferência por zero serviço; C e D eliminadas; remover o placar numérico; separar motor de implantação; spike obrigatório para aceitar. Confiança 62–65% | Alto para esta pergunta, com limite |

**Limites do council neutro:** só 3 membros, todos da mesma tríade (arquitetura); não houve voto vencido, então não existe relatório de minoria. O contrafactual mais forte foi "**decidir o host primeiro**", incorporado como condição de aceite. A confiança de ~60% reflete que **não há medição**, apenas opinião fundamentada.

## O que não foi verificado

- Nenhum benchmark, spike ou teste de concorrência foi executado; as notas de cada motor são conhecimento geral.
- A estimativa de 65–160 MB/ano supõe 20–50 KB por entrada.
- O host real (Q-10) e a disponibilidade de disco persistente.
- Se o script de backup do Postgres existente é reutilizável para um novo banco.

## Próximos passos

1. Responder Q-10 (onde roda; disco persistente; RAM disponível para um Postgres).
2. Rodar o spike de ~1 h e registrar o resultado neste ADR.
3. Executar o caminho de backup definido (incluindo a ida e volta) e marcar o ADR como *Accepted* (ou *Superseded* se a hipótese for refutada).
4. Ajustar a spec (Assumptions) e, se A vencer, DEC-02/RNF-002/RNF-016 no documento de origem.

## Referências

- Spec `specs/001-feed-curation-v1/spec.md` (FR-012, 019, 025, 026, 032, 035, 038).
- `docs/requisitos-feed-curator-v2.md` e `v3.md` (DEC-02, RNF-002, RNF-003, RNF-016, RF-037, §1.3).
- Formato: skill local `domain-modeling` (`docs/adr/NNNN-slug.md`); estrutura de alternativas, consequências e gatilhos inspirada em `addyosmani/agent-skills@documentation-and-adrs` (MIT) e `github/awesome-copilot@create-architectural-decision-record` (MIT). Nenhuma skill de mercado foi instalada.
