# HarnessQwen v10.1 (template generico)

Harness generico e stack-agnostic para desenvolver projetos inteiros com o cline CLI guiado por um LLM local (Qwen via LM Studio). Esta pasta e o TEMPLATE: copie para um repo novo, escreva uma SPEC, gere as sprints e rode. Qualquer agente que receba esta pasta + uma SPEC tem toda a informacao para tocar o projeto do zero ao fim.

## O que e

E um esqueleto de projeto que voce copia para a raiz de um repo novo. Voce descreve o produto numa SPEC unica e quebra o trabalho em sprints; o harness entao roda o cline CLI feature a feature, com o LLM servido localmente pelo LM Studio.

O modelo central e contexto-zero por feature: um loop externo (`orchestrate.sh`) abre um cline NOVO para cada feature pendente, ele implementa SOMENTE aquela feature, marca como done e encerra. A proxima feature comeca noutra task limpa. O reset de contexto vem da task nova do cline, nao de reiniciar o LM Studio (que nunca para). Isso mantem o contexto plano feature a feature e evita estouro de janela em projetos grandes.

Cada feature passa por gates mecanicos antes de fechar. Os gates de parse, unicode, paths e grep (positivo/negativo) rodam num comando so via `gates.py <sprint> <feat>`; o typecheck roda a parte. So depois da self-review com evidencia por criterio a feature vira done.

> Nao usamos o workflow `/develop` do cline. O loop e o `orchestrate.sh` (bash, externo) e as regras sao o `.clinerules/clinerules.md` (auto-carregado pelo cline a cada turno). Por isso nao existe `.clinerules/workflows/` neste harness.

## O que e generico (pronto) vs o que VOCE escreve (por projeto)

Esta e a divisao mais importante do harness. O motor inteiro ja vem pronto e nunca e editado; o que muda por projeto e SO conteudo (SPEC, sprints, tokens do clinerules):

| Camada | Generico - JA PRONTO, nao edite | Voce escreve por projeto |
|--------|----------------------------------|--------------------------|
| Definicao do trabalho | - | `SPEC.md` (via `templates/SPEC-TEMPLATE.md`); sprints `.harness/sprints/NN-*.json` (via `templates/SPRINT-TEMPLATE.md`); `00-index.json` |
| Regras do cline | regras 0-24 de qualidade em `.clinerules/clinerules.md` | os placeholders `<...>` do topo + a secao 8 (Invariantes do projeto), via `templates/CLINERULES-TEMPLATE.md` |
| Gates | os scripts em `.harness/scripts/` (`gates.py`, `gate-*.py`, `gate-*.sh`) | o bloco `verification` de CADA feature no sprint JSON (e o que os scripts checam) |
| Motor / loop | `orchestrate.sh` + `feat-*.py` + `sprint-*.py` + `audit-final.py` | nada (so variaveis de ambiente: `THINKING`, `FEAT_TIMEOUT`) |
| Observabilidade | `render.py`, `tokens.py`, `monitor.py` | nada |

Em uma frase: voce nunca escreve codigo de motor nem de gate. Voce escreve a SPEC, as sprints (com o bloco `verification` de cada feature) e preenche os tokens do clinerules. O resto ja roda.

## Regra de ouro: features PEQUENAS

O reset de contexto e ENTRE features. DENTRO de uma feature o contexto cresce a cada turno, e o cline NAO compacta de forma confiavel no modo CLI. Entao cada feature TEM que ser pequena o bastante pra nunca chegar perto da janela do modelo:

- 1-3 arquivos por feature (idealmente 1).
- Alvo: ~20 turnos do modelo, contexto abaixo de ~40k tokens.
- Se uma feature toca muitos arquivos OU tem logica complexa (um driver inteiro, uma rota inteira), QUEBRE em features menores extraindo um helper/modulo. Ex: um driver -> mapeador puro + driver; uma rota SSE -> helper de stream + handler.
- O alarme de 70k do `render.py` (modo `CTX=1`) sinaliza feature grande demais. Feature que estoura a janela e truncada pelo servidor (perde contexto) e falha; o orchestrator re-tenta, mas a solucao certa e quebrar a feature.

## O motor: orchestrate.sh (generico)

`orchestrate.sh` e o loop mestre. E 100% generico: identico em todo projeto, voce NAO edita. Roda num terminal (zsh/bash) FORA do cline e, para cada feature pendente:

1. le `.harness/current.txt` (qual sprint esta em execucao).
2. acha a proxima feature com `status: "pending"` no sprint JSON.
3. abre um cline NOVO (contexto zero) so pra ela, com um prompt fixo: pega a feature via `feat-context.py`, implementa seguindo o `.clinerules`, roda `gates.py`, faz a self-review, marca done via `feat-status.py` e encerra.
4. espera o cline encerrar e le o `status` da feature no JSON.
5. `done` -> proxima feature | nao-`done` -> re-tenta (ate 3x; o cline ja tem `--retries 4` interno por feature) | sprint sem pendentes -> `sprint-close.py` fecha a sprint e avanca o `current.txt`.
6. repete ate `current.txt == "DONE"`.

Toda a configuracao e por variavel de ambiente (nunca editando o script):

| Var | Default | O que e |
|-----|---------|---------|
| `THINKING` | `high` | esforco de raciocinio: none, low, medium, high, xhigh |
| `FEAT_TIMEOUT` | `2400` | timeout em segundos por feature |
| `CTX` | (desligado) | `CTX=1` liga o dashboard de tokens (`render.py` + `monitor.py`/`tokens.py`) |

O modelo NAO e variavel de ambiente: e o que voce define em `cline auth lmstudio -m <modelo>` (qualquer modelo carregado no LM Studio, veja `lms ls` - nao precisa ser Qwen).

Rodar: `bash .harness/scripts/orchestrate.sh` (ou `CTX=1 bash .harness/scripts/orchestrate.sh` pra ver os tokens por feature). O script descobre a raiz do repo sozinho; rode da raiz do projeto.

## Gates: como funcionam e como criar os seus

Os gates sao a verificacao mecanica que cada feature passa antes de fechar. Eles NAO confiam na palavra do modelo. Tem duas camadas:

1. O motor (generico, nao mexa). Os scripts em `.harness/scripts/` sao iguais em todo projeto:
   - `gates.py <sprint> <feat>` roda num comando so: parse dos arquivos editados, unicode (proibe em-dash e smart quotes), paths de `files[]` existem, e grep positivo/negativo.
   - `gate-*.py` / `gate-*.sh` sao checagens estruturais reusaveis: `gate-anti-empty` (nao esvaziou campos do JSON), `gate-lifecycle`/`gate-idempotency`/`gate-sprint-closed` (estado das sprints), `gate-paths`, `gate-import-resolve`, `gate-unused`, `gate-consistency`.
   - Voce NUNCA edita esses scripts. Eles leem o que checar do sprint JSON.

2. O que VOCE escreve, por feature: o bloco `verification`. "Criar um gate" para a sua feature = preencher o `verification` daquela feature no sprint JSON. Os scripts genericos consomem esse bloco. Campos principais:
   - `grepMustMatch`: regex que TEM que existir nos arquivos da feature (prova que implementou). Semantica OR entre alternativas.
   - `grepMustNotMatch`: regex que NAO pode existir (proibe atalho, placeholder, anti-pattern).
   - `grepFiles`: em quais arquivos rodar os greps.
   - `smoke` (opcional): comando executavel + `executedBy: workflow` + `timeoutSeconds`, para um teste de fumaca.

   Ou seja: o motor e generico e ja vem pronto; o "gate" de cada feature e DADO que voce escreve no JSON, nao codigo. O detalhe campo-a-campo do bloco `verification` (e de toda a sprint) esta em `templates/SPRINT-TEMPLATE.md`, secoes 4 e 5.

O typecheck roda a parte, como gate dinamico pos-feature (o `<TYPECHECK_CMD>` que voce define no clinerules), nao via `gates.py`.

## Sprint Review Final: audit por categoria contexto-zero + boot-smoke real

A ultima sprint do pipeline e a Sprint Review Final (template generico em `.harness/sprints/REVIEW-TEMPLATE.json`). Ela roda DEPOIS de todas as sprints de implementacao e ANTES do `DONE`. Existe porque um LLM medio nao consegue revisar 60+ arquivos de uma vez: a janela estoura e a revisao vira superficial. A solucao tem duas camadas complementares.

**Camada 1 - audit por categoria, contexto-zero.** `audit-final.py` varre o codigo ja implementado e gera um relatorio categorizado em `/tmp/harness/audit-final.json` (categorias: CONSISTENCY, DEAD_CODE, SECURITY, ANTI_PATTERN, DUPLICATION, TODO). Cada feature da sprint consome UMA categoria e so fecha quando aquela categoria nao tem mais findings HIGH (algumas exigem HIGH+MED zerados). Como cada feature roda num cline fresco (mesmo modelo contexto-zero do resto do harness), o revisor olha um conjunto pequeno e focado de findings por vez, em vez de tentar segurar o projeto inteiro na cabeca. A revisao fica incremental e mecanicamente verificavel: o proprio `audit-final.py` re-roda e prova que a categoria zerou.

**Camada 2 - boot-smoke real.** Audit e typecheck sao estaticos: leem o codigo parado e checam tipos. Eles nao veem a classe de bug que so aparece quando o app SOBE. A feature de boot-smoke sobe o entrypoint REAL do app montado (o mesmo wiring de producao - servidor/CLI/worker), espera readiness e da health-check nos endpoints/superficies principais (HTTP via `curl -fsS` no `/health` e nas rotas-chave; ou, se nao for servico de rede, invoca o entrypoint montado e checa exit code + saida esperada), depois derruba o processo limpo. Ela e stack-agnostic: o comando concreto (build, start, porta, paths de endpoint) vem do bloco `verification.smoke` da feature, preenchido por projeto a partir dos tokens do clinerules - nada de framework hardcoded no template. O smoke roda via `executedBy: workflow`, entao o workflow re-executa por conta propria pos-done: o modelo nao pode mentir que subiu.

**Por que isso pega bugs de integracao.** Typecheck garante que os tipos casam; o audit pega inconsistencia, dead code, security e anti-patterns no codigo parado. Nenhum dos dois exercita o grafo de montagem em runtime. So o boot-smoke pega o "compila, tipa e passa no audit, mas explode ao subir": wiring quebrado entre modulos, ordem de inicializacao errada, provider/dependencia que so falha quando instanciado, import circular que so quebra em runtime, rota registrada com handler errado, variavel de ambiente obrigatoria nao lida no boot, schema/migracao que so diverge ao conectar nos dados. Esses bugs vivem nas costuras entre componentes que cada sprint construiu isoladamente; subir o app montado e a unica prova de que as pecas realmente se encaixam. Quando o boot revela um bug, a feature conserta o wiring no codigo de app e re-roda verde - nunca relaxa o health-check para passar.

> **Stack do audit:** o `audit-final.py` e os gates `gate-import-resolve.py`/`gate-consistency.py`/`gate-unused.sh` sao focados em projetos **TypeScript/Node** (extensoes `.ts/.tsx`, diretorios convencionais, `tsc`). Para outra stack (Python, Go, Rust, ...), peca a um agente LLM para adaptar esses arquivos a sua linguagem (extensoes, diretorios de codigo, comando de typecheck) antes da Sprint Review Final. O boot-smoke ja e stack-agnostico (o comando vem do `verification.smoke` da feature).

## Estrutura de pastas

```
HarnessQwen-v10.1-template/
|- README.md                  <- este arquivo
|- START-HERE.md              <- passo a passo para iniciar um projeto novo
|- .clinerules/
|  '- clinerules.md           <- regras/invariantes carregadas todo turno do cline (auto-load)
|- .harness/
|  |- scripts/                <- orchestrate.sh + gates.py + gate-*.py/sh + feat-*/sprint-* + render.py/tokens.py/monitor.py + audit-final.py
|  '- sprints/                <- ja vem 00-index.json.example + REVIEW-TEMPLATE.json (voce cria as sprints reais aqui)
|- docs/
|  '- CLINE-SETUP.md          <- setup do cline CLI + LM Studio
'- templates/                 <- SPEC, SPRINT, CLINERULES (modelos para preencher)
```

> **Pastas ocultas:** `.clinerules/` e `.harness/` comecam com ponto, entao ficam OCULTAS no gerenciador de arquivos (use **Ctrl+H** para ve-las) e no `ls` comum (use `ls -a`). Elas SAO incluidas normalmente no zip e sao obrigatorias para o harness rodar.

Voce cria, por projeto: `SPEC.md` na raiz, as sprints `NN-*.json` + `00-index.json` em `.harness/sprints/`, e `.harness/current.txt` apontando a primeira sprint. Opcional: uma pasta `design/` com o design lock, se o projeto tiver, referenciada pela SPEC.

## Como rodar (3 passos)

1. Preencha o conteudo do projeto: escreva `SPEC.md` (use `templates/SPEC-TEMPLATE.md`), crie as sprints em `.harness/sprints/` (`00-index.json` + uma `NN-<nome>.json` por sprint, via `templates/SPRINT-TEMPLATE.md`), preencha os tokens do `.clinerules/clinerules.md` (via `templates/CLINERULES-TEMPLATE.md`) e aponte `.harness/current.txt` para a primeira sprint.

2. Autentique o cline no LM Studio:

   ```bash
   cline auth lmstudio -k lm-studio -m <modelo>
   ```

3. Rode o loop mestre na raiz do projeto:

   ```bash
   bash .harness/scripts/orchestrate.sh
   ```

   Ele itera as sprints feature a feature, abrindo um cline fresco por feature, ate `.harness/current.txt == "DONE"`. Por padrao a saida e o cline nativo streamando na tela; `CTX=1 bash .harness/scripts/orchestrate.sh` ativa o dashboard de tokens (`render.py` + `monitor.py`) para conferir o reset de contexto.

O passo a passo completo (pre-requisitos, criar sprints, preencher tokens, validar antes de rodar) esta em `START-HERE.md`. Setup do cline CLI e do LM Studio em `docs/CLINE-SETUP.md`.
