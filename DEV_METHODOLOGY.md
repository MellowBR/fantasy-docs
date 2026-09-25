# Metodologia de Desenvolvimento — Projetos Fantasy

> Documento para carregar como Project Knowledge no Claude.ai.
> Define o modelo de desenvolvimento padrao para todos os projetos.
> Derivado da experiencia com o Fantasy Optimizer e o Predictor (Mar/2026).
> **Fonte unica e versionamento (25/09/2026):** a copia canonica e `Fantasy/DEV_METHODOLOGY.md`
> (raiz do ecossistema; os projetos apontam para `../DEV_METHODOLOGY.md`). Ela e versionada no
> repositorio da raiz `Fantasy/`, exclusivo de docs transversais (`.gitignore` em whitelist: este
> arquivo, `CONTEXT_FANTASY_PROJECTS.md` e `pff_data/CONTEXT_MYPFF_PROJECT.md`). Nao manter copia
> em nenhum repo de projeto. (Origem: PRED-P19-F1 expôs que este documento e o CONTEXT_MYPFF
> estavam sem historico desde abril.)

---

## Principio Central

Cada projeto segue um ciclo de **plano → implementacao em camadas → validacao**.
O Claude Code implementa, o Claude.ai planeja. Ambos compartilham o mesmo contexto
via documentos vivos.

---

## Principio de Auto-Containment Documental

Os 4 documentos de cada projeto (CLAUDE.md, devplan.md, improvements.md, vision.md)
somados ao codigo devem ser **auto-contidos**: qualquer um (outro Claude sem memoria,
um colaborador novo, o proprio owner daqui a 2 anos) deve conseguir
**retomar, replicar ou auditar o projeto** usando apenas a documentacao + o
repositorio.

**Teste pratico:**

> "Um Claude novo, sem memoria, com acesso apenas ao repositorio e ao
> DEV_METHODOLOGY.md carregado como Project Knowledge, consegue trabalhar
> no projeto no mesmo nivel que um Claude com memoria?"
>
> Se sim → a memoria virou conveniencia, nao dependencia.
> Se nao → algum conhecimento critico esta preso num lugar volatil
> (memoria, conversa Claude.ai, cabeca do owner) e precisa migrar para os docs.

**Consequencias praticas:**
- Log de Decisoes captura o *why*, nao so o *what* — auditor 2 anos depois
  precisa entender por que cada decisao foi tomada
- Resultados reais de validacao vivem no CLAUDE.md, nao so no log — eles sao
  o "estado provado atual", nao historico de decisoes
- Vision.md e o "porque existe para sempre", nao envelhece com a implementacao
- Handoff e ponte temporaria, nunca persistencia — info que nao migrou para o
  devplan antes do descarte esta perdida
- Memoria (auto-memory do Claude) nao e fonte de verdade — guarda so perfil
  do owner, feedback comportamental e referencias externas; estado tecnico
  do projeto vive nos 4 docs

**O que quebra parcialmente o principio:**
Dados sensiveis (.env, senhas, CPFs, dados financeiros reais) nao podem estar
no repo. Nesse caso o CLAUDE.md descreve *o que* deve existir (variaveis,
schemas de dados esperados) sem os valores. O segredo e externo, a estrutura
que o consome e documentada.

---

## Documentos por Projeto

Cada projeto tem exatamente 4 documentos de gestao:

### 1. CLAUDE.md — Referencia tecnica + contrato

**O que contem:**
- Overview do projeto e pergunta central
- Setup & commands (todos os comandos reais, nao planejados)
- Arquitetura (bancos, modulos, responsabilidades)
- Constantes e convencoes
- Resultados reais de validacao (nao expectativas)
- Contrato com outros projetos (quando aplicavel)
- Status de desenvolvimento

**Regras:**
- Sempre reflete o estado ATUAL, nao o planejado
- Atualizado ao final de cada camada
- E o primeiro documento que o Claude Code le ao iniciar uma sessao
- Nao contem planos futuros — so o que existe e funciona

### 2. devplan.md — Plano vivo + Log de Decisoes

**O que contem:**
- Visao geral e filosofia do projeto
- Plano por camadas com status (Not started / In progress / Done)
- Formulas, filtros, regras de negocio detalhadas
- Casos de validacao obrigatorios
- **Log de Decisoes** — secao cronologica no final

**Regras:**
- Documento VIVO — atualizado durante a implementacao
- O plano original pode mudar — o log registra o que mudou e por que
- Nao criar handoffs separados — tudo fica no devplan

**Formato do Log de Decisoes:**
```markdown
## Log de Decisoes

### [DATA] — [CAMADA]
- **[Decisao]:** [justificativa]
- **[Correcao ao plano]:** [o que mudou e por que]
- **[Resultado real]:** [dados concretos, nao expectativas]
```

**Cada entrada precisa capturar o *motivo* da decisao, nao apenas o *que* foi feito.**
Um auditor futuro (humano ou outro Claude sem contexto da conversa) deve conseguir
entender por que aquela escolha foi feita, que alternativas foram consideradas, e
que restricao/contexto motivou o resultado. Sem o motivo, o log vira historico de
mudancas sem utilidade para replicar ou questionar decisoes.

### 3. improvements.md — Backlog vivo

**O que contem:**
- Itens pendentes, parciais e concluidos
- Cada item: problema, proposta, prioridade
- Tabela de status no final

**Regras:**
- Items marcados com 🔲 (pendente), ⚠️ (parcial), ✅ (concluido)
- Adicionar novos itens ao descobrir problemas em producao
- Marcar como ✅ com data ao concluir

**Projetos acoplados:** quando dois projetos formam um sistema unico do ponto
de vista do usuario (ex: Optimizer + Predictor), manter um unico improvements.md
no projeto principal. Itens do projeto secundario ficam com prefixo proprio
(ex: P7, P8) mas no mesmo backlog. Adicionar nota no topo do arquivo indicando
cobertura multi-projeto.

### 4. vision.md — Motivacao e casos de uso

**O que contem:**
- Por que o projeto existe
- Limitacoes de projetos existentes que este resolve
- Casos de uso concretos com jogadores reais

**Regras:**
- Escrito uma vez no inicio, atualizado com resultados reais ao final
- Nao contem detalhes de implementacao — so motivacao e contexto

---

## Fluxo de Trabalho por Camada

### Fase 1: Planejamento (Claude.ai)

1. Ler devplan.md + CLAUDE.md + improvements.md
2. Redigir prompt da camada com:
   - Contexto (o que ja existe)
   - Tarefas detalhadas
   - Formulas e regras
   - Casos de validacao obrigatorios
   - Checklist antes de encerrar
3. Revisar e refinar o prompt antes de enviar ao Claude Code

### Fase 2: Analise (Claude Code)

1. Claude Code recebe o prompt da camada
2. **Antes de implementar:** analisa, critica, e sugere melhorias
3. Verifica dados reais (queries no banco, arquivos existentes)
4. Identifica divergencias entre o prompt e a realidade dos dados
5. Reporta problemas e pede confirmacao antes de implementar

### Fase 3: Implementacao (Claude Code)

1. Implementa apos confirmacao do usuario
2. Testa cada funcao conforme implementa
3. Roda os casos de validacao obrigatorios
4. Documenta decisoes tomadas durante implementacao

### Fase 4: Finalizacao (Claude Code + usuario)

1. Rodar checklist completo
2. Atualizar devplan.md — status da camada + log de decisoes
3. Atualizar CLAUDE.md — estado atual + resultados reais
4. Adicionar itens descobertos ao improvements.md
5. **Handoff de sessao (Code):** ao fim de cada sessao de trabalho, o Code cria
   `handoff_PROJ_DD_MM_YYYY.md` (ex: `handoff_optimizer_03_04_2026.md`) com:
   o que foi feito, commits, estado dos bancos, proximos passos. Este e diferente
   do handoff de camada — nao substitui o devplan, complementa a continuidade
   entre sessoes. O devplan continua sendo a fonte de verdade — o handoff e
   descartavel apos leitura.

### Checklist de fim de sessao (OBRIGATORIO antes de encerrar)

Antes de declarar a sessao encerrada, validar que o estado dos docs canonicos
(devplan + improvements + CLAUDE.md) reflete a realidade end-to-end. Padrao
recorrente de bug a evitar: "info ficou em chat ou em commit message, nao
migrou pra doc canonico → proxima sessao perde inteligencia".

- **Status (✅/⚠️/🔲) em improvements.md reflete realidade end-to-end?**
  Marcar ✅ apenas quando o fix esta validado em produção (não só backend
  unit-tested). Se há réplica frontend/JS não tocada, status correto é ⚠️.
  **Gate de hash deployado (PROC1 — transversal: manager + optimizer deployam no Render):**
  para fechamentos cujo ✅ dependa de **smoke em produção**, "commitado" e "pushado" NÃO
  bastam — confirmar explicitamente que o **hash deployado live em prod é o commit que contém
  a mudança validada** (painel do Render: deploy live = hash esperado) ANTES de confiar no
  resultado do smoke e flipar ✅. Escopo: só fechamentos com gate de prod; itens validáveis
  só em localhost não são afetados. Incidentes que motivaram a regra: **E1** (✅ prematuro
  pós-localhost → falha em prod) e **E4-a/23-06** (o smoke reproduziu o comportamento pré-fix
  porque o deploy live era um commit docs-only anterior, `927831a`, não o `97b90ed` do filtro
  — que nunca tinha sido pushado).
- **Diagnoses subsequentes (F2/F3) que descobriram novos itens viraram entradas no backlog?**
  Per regra ja existente "adicionar itens descobertos ao improvements.md".
  Diagnose F2 que aponta novo bug → novo item 🔲 imediatamente, antes de
  encerrar sessao.
- **Sub-fixes (FIX, FIX2, hotfixes) tem entradas no log do devplan?**
  Commit message nao e fonte de verdade — devplan e. Se uma decisao do log
  original foi revertida/modificada por sub-fix, registrar a correção com
  link para o sub-fix.
- **Meta-mudanças (DEV_METHODOLOGY, tooling, fluxos) tem registro de motivacao?**
  Mudancas em metodologia ou ferramental devem ter um log que responda "por
  que essa regra foi criada, qual incidente motivou". Lugar canonico: log
  do devplan da sessao que originou (referência cruzada nos outros ecossistemas
  se transversal).
- **Pendencias reais (bugs em producao, tarefas inacabadas) estao registradas?**
  Tudo que ficou em "vou fazer depois" no chat precisa virar item 🔲 no
  improvements.md antes de encerrar — senao depende exclusivamente da memoria
  (volatil) ou do owner reabrir.

---

## Fluxo para Bugs e Calibracoes

Diferente do fluxo de features (camadas P0→Pn), bugs e calibracoes seguem:

### Fase 1 — Diagnose (read-only obrigatorio)

- Consultar banco de dados (queries SQL, scripts de leitura)
- Ler codigo relevante para entender causa raiz
- **Replicas e duplicacoes (OBRIGATORIO para logica de calculo/transformacao):**
  perguntar "esta logica/formato existe em mais de um lugar?". Verificar
  explicitamente: outros modulos Python, JS em templates, helpers em outros
  blueprints, geradores de mesma chave/formato. Diagnoses de helpers que
  pulam essa pergunta tendem a gerar fixes pela metade (ex: backend OK,
  frontend continua com bug).
- Documentar achados no improvements.md (sub-item "Fase 1 Diagnose ✅")
- **Read-only** significa nao alterar codigo nem dados — executar scripts de
  leitura (queries via Python, leitura de arquivos) e permitido e esperado

### Fase 2 — Implementacao

- So apos causa raiz confirmada na Fase 1
- Prompt separado com referencia a diagnose (ex: "Ref: OPT-B11-F1")
- Validar que o fix resolve exatamente o problema diagnosticado

**Quando usar:** causa raiz nao e obvia, problema pode ter multiplas causas,
ou fix errado pode causar regressao.

---

## Regras para o Claude Code

### Antes de implementar (OBRIGATORIO)
- Analisar o prompt
- Criticar pontos problematicos
- Verificar dados reais (nao confiar no prompt cegamente)
- **Antes de aceitar escopo de fix em helper de calculo/transformacao:**
  grepar pelo padrao de saida (literal de string, prefixo de chave, formato)
  em TODO o codebase — nao so pelo nome da funcao. Replicas client-side
  raramente importam o helper Python; usam o mesmo formato de saida inline.
  Custo: 3 segundos. Beneficio: pega categoria inteira de "fix backend mas
  esqueci do frontend".
- Sugerir melhorias
- Aguardar confirmacao explicita

### Durante a implementacao
- Antes de escrever qualquer arquivo, apresentar checklist numerado com todas as
  etapas planejadas. Iniciar implementacao apenas apos confirmacao explicita do usuario.
- Usar tasks para tracking de progresso
- Testar cada modulo conforme implementa
- Rodar validacoes antes de declarar a camada concluida
- Documentar decisoes que divergem do plano original
- **Smoke local que dá boot na aplicacao DEVE apontar a env do banco (`DYNASTY_DB` no
  Manager) para uma COPIA TEMPORARIA, nunca para o caminho padrao.** O boot roda
  `db.create_all()` + migracoes; contra o caminho padrao ele muta o **seed versionado**
  (`dynasty.db`), que é consumido tambem pelo **optimizer** e pelo **predictor** — um
  commit acidental propagaria schema de um projeto para os outros dois. (Origem: sessao
  MAN-F13, 31/07/2026 — um boot de smoke caiu no default e rodou migracoes contra o seed;
  detectado no diff antes do commit e revertido. Reforca o [[F12]] do Manager: boot muta
  o DB local.)
- **Toda mudanca de FORMA no MYPFF (`MYPFF_Complete.db`) passa antes por um item `MYPFF-*`,
  qualquer que seja a superficie que a execute (Code, Cowork, Claude.ai, manual).** Forma =
  tabela nova, tipo novo de `source`, mudanca de schema (coluna nova, tipo, indice) ou carga
  de natureza nova (ex.: temporada parcial, dado semanal). **Execucao de rotina de fluxo ja
  registrado nao precisa de item** (ex.: o ciclo anual MYPFF-UN, ou a carga semanal da
  `mypff_weekly` depois que o P19 a formalizar). O item existe para que alguem pergunte, ANTES
  da carga, "quem le isto e como?" — o MYPFF e compartilhado por Predictor e Optimizer, e
  nenhum dos dois tem nocao de "temporada parcial". (Origem: sessao PRED-P19-F1, 25/09/2026 —
  a carga de 22/09 das linhas `*_2026_2027`, intencional e feita via Cowork, entrou como
  temporada comum na tabela `mypff` e e lida como temporada completa por ~100 pontos de
  consumo nos dois projetos; a simulacao mediu 57 jogadores mudando de modelo no Predictor,
  todos os SELL_HIGH do MVO zerados e o overall de 216 de 282 ativos alterado no Optimizer.)

### Apos implementar
- Atualizar devplan.md e CLAUDE.md
- NAO criar arquivos de handoff separados
- Registrar problemas descobertos no improvements.md

---

## Regras para o Claude.ai (Planejamento)

### Antes de redigir qualquer prompt
- Ler DEV_METHODOLOGY.md, CLAUDE.md, devplan.md e improvements.md do projeto
- Consultar o status atual das camadas no devplan.md
- Consultar improvements.md para itens pendentes que afetam a camada
- Nao incluir snippets de codigo no prompt — o Code le o codigo real

### Formato obrigatorio dos prompts
- Seguir o template do tipo aplicavel (Acao ou Registro — ver "Tipos de prompt")
- ID no formato `{PROJ}-{ITEM}-{FASE}` (Acao) ou `{PROJ}-{ITEM}-REG` (Registro)
- Para Acao: sempre as 6 secoes CONTEXTO, TAREFA, DADOS, RESTRICOES, VALIDACAO, AO FINALIZAR
- Nunca incluir codigo nos prompts
- Nunca listar arquivos especificos — indicar modulo ou area se relevante
  (ex: "modulo SBO no Predictor" em vez de "predictor/sbo_model.py linhas 388-395")
- Nunca detalhar sequencia de execucao — o Code decide como
- **RESTRICOES preferem intencao a caminho de arquivo:** descrever o que NAO
  alterar em termos de dominio/contrato (ex: "nao alterar schema, endpoints
  nao relacionados, salary_engine") em vez de "alterar apenas X.py". Restricao
  por arquivo amarra o escopo antes de saber se a logica tem replicas em
  outros arquivos. Use restricao por arquivo so quando o escopo e
  trivialmente unico (ex: typo, rename num arquivo).
- Para Acao: sempre incluir casos de validacao com valores concretos
- Para Acao: sempre pedir ao Code para analisar antes de implementar
- Antes de entregar o prompt, comparar campo a campo contra o template canonico
  (secao Template de prompt, relida na presente sessao). Se alguma secao nao
  corresponder, reescrever antes de entregar.
- Entregar o prompt em bloco unico formatado para copiar e colar

### Durante a analise do Code
- Aguardar a analise critica do Code antes de confirmar
- Responder com bloco de confirmacao estruturado (um item por ponto levantado)
- Registrar alinhamentos relevantes para que o Code inclua no Log de Decisoes
  ao finalizar a camada

### Checklist antes de enviar um prompt
- O tipo esta correto? (Acao para implementar/diagnosticar; Registro para capturar discussao adiada)
- O prompt cobre exatamente o escopo intendido (escopo da camada para Acao; conteudo da discussao para Registro)?
- Para Acao: os casos de validacao tem valores concretos e verificaveis?
- Para Acao: o prompt pede checklist numerado de etapas ao Code antes de implementar?
- Para Registro: secoes Discussao, Decisoes, Alternativas, Questoes em Aberto preenchidas?
- Nao ha codigo nem listagem de arquivos especificos no prompt?
- Se cross-repo: TAREFA cobre os 4 lados (producer, consumer, contract_test, ordem de deploy)?

### Tipos de prompt

Existem dois tipos por proposito:

- **Acao** — produz codigo ou diagnose (implementar feature, fixar bug, calibrar modelo, investigar causa raiz). Template canonico abaixo.
- **Registro** — produz uma entrada de backlog que serve como registro historico de uma discussao substantiva ainda nao implementada nesta sessao. Captura o porque, as decisoes ja tomadas e as alternativas descartadas, para que quem pegar o item depois (talvez 3 semanas) entenda o contexto sem reconstruir.

Itens triviais (typo, rename, registro pos-decisao obvia) nao precisam de prompt — sao edicao direta de 2 linhas em improvements.md.

### Template de prompt — Acao (Claude.ai → Code)

Todo prompt de acao deve seguir este formato:

> **Fonte unica.** Este bloco e a unica definicao do template de prompt. Nao resumir,
> parafrasear ou reproduzir parcialmente em CLAUDE.md, CONTEXT_*.md, ROADMAP.md,
> handoffs ou qualquer outro doc (incluindo artefatos do Claude.ai Project) —
> copias driftam e Claude.ai passa a consultar o resumo errado em vez desta fonte.
> Outros docs podem apenas referenciar esta secao por ancora (ex: "ver
> DEV_METHODOLOGY.md secao Template de prompt").
>
> **Antes de redigir qualquer prompt {PROJ}-{ITEM}-{FASE}:** reler esta secao
> literalmente. Nao confiar em memoria da conversa, em resumo de outro doc, nem
> em cache do Claude.ai Project.

```
# {PROJ}-{ITEM}-{FASE} | {Projeto} | {Item e descricao}

## CONTEXTO
[2-4 frases: o que existe, o que o diagnose encontrou]
Ref: [ID do prompt anterior, se aplicavel]

## TAREFA
[Intencao clara do que implementar — sem codigo]

## DADOS
[Valores concretos, IDs, regras de negocio relevantes]

## RESTRICOES
[O que NAO fazer, o que NAO alterar]

## VALIDACAO
[Casos concretos com valores esperados]

## AO FINALIZAR
[Atualizar improvements.md + commits + o que reportar]
```

**Cross-repo (item toca producer + consumer):** TAREFA deve cobrir explicitamente as 4 dimensoes — (1) mudanca producer, (2) mudanca consumer, (3) contract_test, (4) ordem de deploy. RESTRICOES preserva o contrato dos demais consumidores (ex: versoes de schema anteriores aceitas em paralelo). VALIDACAO inclui caso producer-side + caso consumer-side + backwards-compat se aplicavel.

### Template de prompt — Registro

Quando a discussao desta sessao gerou um item de backlog que **nao** sera implementado agora (adiado por dependencia, prazo, escopo), use este template no lugar do Acao. O objetivo e que improvements.md fique com registro suficiente para retomar 3 semanas depois sem perder contexto.

```
# {PROJ}-{ITEM}-REG | {Projeto} | {Descricao curta}

## CONTEXTO
[O que motivou levantar isso nesta sessao]

## PROBLEMA / OPORTUNIDADE
[O que precisa mudar e por que importa — 2-4 frases]

## DISCUSSAO
[Pontos-chave do debate — o que mudou de opiniao, o que ficou claro]

## DECISOES JA TOMADAS
- [X em vez de Y porque Z]
- [Limite de escopo: ate W]

## ALTERNATIVAS DESCARTADAS
- [Opcao A: rejeitada porque...]
- [Opcao B: rejeitada porque...]

## QUESTOES EM ABERTO
[Coisas que F1 (diagnose) deve responder antes de F2]

## DEPENDENCIAS
- Depende de: [outros itens]
- Bloqueia: [outros itens]

## AO FINALIZAR
Registrar em improvements.md com 🔲 + ID + secoes acima preservadas. Adicionar linha curta no Log de Decisoes do devplan referenciando o item.
```

A acao subsequente (F1/F2) referencia o registro: `Ref: {PROJ}-{ITEM}-REG`. O prompt de acao fica mais curto porque rationale ja esta no backlog.

### ID de prompt

Cada prompt recebe um ID no formato `{PROJ}-{ITEM}-{FASE}`:
- `OPT-B11-F1` → Optimizer, item B11, Fase 1 (Acao)
- `OPT-A28` → Optimizer, item A28 sem fases (Acao)
- `PRED-P7` → Predictor, item P7 (Acao)
- `MAN-T1` → Manager, item T1 (Acao)
- `OPT-B25-REG` → Registro de discussao substantiva do item B25 (Acao subsequente vira `OPT-B25-F1`/`OPT-B25-F2`)

O ID aparece no cabecalho do prompt e permite referenciar conversas
anteriores entre Claude.ai e Code, especialmente quando o owner segue
outras discussoes com o Claude.ai enquanto o Code implementa.

### Ao revisar resultados
- Comparar resultados reais com expectativas
- Identificar surpresas (Ferguson HIGH, Burden resgate)
- Decidir se sao bugs ou insights do modelo
- Registrar correcoes ao devplan quando necessario

---

## Integracao entre Projetos

Quando dois projetos se comunicam:

1. **Contrato formal** no CLAUDE.md de ambos os lados
   - Produtor: o que garante (schema, formato, valores validos)
   - Consumidor: o que espera (colunas, versoes, regras de manutencao)

2. **Contract test** no lado do produtor
   - Script executavel que valida a integracao
   - Roda antes de publicar (`--contract-test`)

3. **Schema version** nos dados publicados
   - Versao no JSON/tabela para o consumidor verificar compatibilidade
   - Versao desconhecida = warning (nao erro) — degradacao elegante

4. **Comunicacao via banco, nao via codigo**
   - Projetos nao importam modulos entre si
   - Toda comunicacao e via tabelas publicadas em bancos compartilhados
   - Cada projeto funciona independentemente dos outros

---

## Propagacao da Metodologia

A metodologia e **transversal** aos ecossistemas (fantasy, energy, finance),
nao especifica de projeto. Qualquer alteracao a um DEV_METHODOLOGY.md deve
ser propagada aos outros na **mesma sessao**, com as adaptacoes minimas
necessarias (exemplos de IDs de prompt, nomes de projetos).

**Caminhos canonicos:**
- `~/fantasy/DEV_METHODOLOGY.md` — cobre fantasy_manager, fantasy_optimizer, predictor
- `~/energy/DEV_METHODOLOGY.md` — cobre ccee_simulator, battery_evaluator
- `~/finance/gestor-financeiro/DEV_METHODOLOGY.md` — cobre gestor_financeiro (unico projeto)

**Regra:** antes de concluir uma sessao que alterou qualquer DEV_METHODOLOGY.md,
confirmar que os 3 caminhos refletem a mesma metodologia base.

**Projetos novos:** podem reservar uma categoria dedicada de IDs no backlog para
itens meta/metodologia desde o inicio (ex: prefixo proprio, distinto das
categorias de features do projeto). Projetos existentes mantem suas convencoes
locais — itens meta usam os IDs sequenciais proprios de cada backlog.

---

## Versionamento (Git)

### Regras
- Cada projeto tem seu proprio repositorio git local
- `.gitignore` obrigatorio: excluir `.env`, `*.db`, `__pycache__/`, `.claude/`, `data/`
- Nunca commitar bancos de dados ou arquivos de ambiente
- Tags semanticas ao concluir marcos: `projeto-v1.0`, `projeto-post-feature`

### Quando commitar
- Ao concluir uma camada completa do devplan
- Ao concluir uma feature do improvements.md
- Antes de mudancas estruturais arriscadas (para poder reverter)

### Formato de commit
```
Titulo curto (< 72 chars)

Descricao do que foi feito, por que, e resultados relevantes.

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>
```

---

## Anti-patterns (o que NAO fazer)

- ❌ Criar um handoff por camada — fragmenta informacao
- ❌ Manter devplan congelado e desatualizado — gera divergencia
- ❌ Implementar sem analisar o prompt antes — erro mais caro de corrigir
- ❌ Confiar em valores esperados do prompt sem verificar dados reais
- ❌ Importar codigo entre projetos — cria acoplamento fragil
- ❌ Alterar bancos read-only "so desta vez" — viola contrato
- ❌ Hardcodar caminhos de banco — usar .env
- ❌ Criar arquivos de documentacao que ninguem vai ler — manter 4 documentos vivos
- ❌ Incluir codigo nos prompts Claude.ai → Code — o Code le o codigo real, snippets no prompt ficam desatualizados e gastam tokens
- ❌ Listar arquivos especificos a modificar no prompt — indicar modulo ou area, o Code descobre os arquivos
- ❌ Detalhar sequencia de execucao no prompt — e papel do Code decidir como implementar
