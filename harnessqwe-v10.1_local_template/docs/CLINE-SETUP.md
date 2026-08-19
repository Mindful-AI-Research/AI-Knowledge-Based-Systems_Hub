# cline CLI + LM Studio - setup

Setup do cline CLI e do LM Studio para rodar o harness v10.1. Ambiente: Linux com zsh.

<br><br>

## 1. Instalar o cline CLI

```bash
npm i -g cline
which cline
cline --version
```

Requer Node 20+.

<br><br>

## 2. Autenticar no LM Studio

O LM Studio expoe um endpoint compativel com OpenAI em `http://127.0.0.1:1234/v1`. Aponte o cline para ele:

```bash
cline auth lmstudio -k lm-studio -m <modelo>
```

- `-k lm-studio` e a chave aceita pelo servidor local do LM Studio (qualquer string nao-vazia serve; `lm-studio` e a convencao).
- `-m <modelo>` e a key do modelo carregado. Liste com `lms ls`.

<br><br>

## 3. Configurar o LM Studio

### Modelo

Use um modelo de codigo Qwen com quantizacao Q5_K_M ou maior. Carregue-o:

```bash
lms load <modelo>
```

### Janela de contexto

| Configuracao | Valor |
|--------------|-------|
| Context Length | a maior janela que o modelo + GPU suportarem |
| GPU Offload | maximo possivel (idealmente 100%) |
| KV Cache | Q8_0 (economiza VRAM com perda minima) |
| Flash Attention | ON |

O modelo contexto-zero por feature mantem o contexto plano feature a feature, entao a janela nao precisa ser gigante; mas folga ajuda em features que tocam varios arquivos.

### Inferencia

| Setting | Valor |
|---------|-------|
| Temperature | 0.2 - 0.3 (codigo precisa de determinismo) |
| Top-K | 40 |
| Top-P | 0.9 |
| Repeat Penalty | 1.05 |
| Min P | 0.05 |

<br><br>

## 4. O loop orchestrate.sh

O `orchestrate.sh` e o loop mestre externo. Para cada feature pendente da sprint apontada por `.harness/current.txt`, abre um cline NOVO (contexto zero), que implementa SOMENTE aquela feature e encerra; o loop fecha a sprint quando ela termina e avanca o `current.txt`, ate `DONE`.

```bash
bash .harness/scripts/orchestrate.sh
```

Por padrao, a saida e o cline nativo streamando na tela (e salva em `.log` para auditoria). Para acompanhar tokens por feature, use o modo dashboard:

```bash
CTX=1 bash .harness/scripts/orchestrate.sh
```

Com `CTX=1`, a saida passa por `render.py` e alimenta `monitor.py`, que mostra contexto por request por feature (a coluna deve ficar plana, provando o reset). Rode `python3 .harness/scripts/monitor.py` num segundo terminal para ver o dashboard.

Variaveis de ambiente do loop:

| Var | Default | O que faz |
|-----|---------|-----------|
| `THINKING` | `high` | nivel de reasoning (none, low, medium, high, xhigh) |
| `FEAT_TIMEOUT` | `2400` | timeout em segundos por feature |

O modelo usado pelo loop e o que voce definiu em `cline auth lmstudio -m <modelo>` (secao 2), qualquer modelo de `lms ls` (nao precisa ser Qwen); nao e variavel de ambiente. Para trocar de modelo, rode o auth de novo com a nova key.

<br><br>

## 5. Validacao pos-config

Antes de rodar o pipeline, confirme no terminal (zsh):

```bash
# 1. Ferramentas no PATH
which cline node python3
node --version    # >= 20
python3 --version # >= 3.11

# 2. LM Studio respondendo com o modelo carregado
curl -s http://127.0.0.1:1234/v1/models | head -1
# deve retornar JSON com os modelos
```

Se algum check falhar, NAO comece. Resolva primeiro.

<br><br>

## Troubleshooting

### `cline: command not found`

O pacote global nao esta no PATH. Confirme `npm i -g cline` e que o bin global do npm esta no `$PATH` (`npm bin -g`).

<br><br>

### `curl` no /v1/models nao retorna nada

O LM Studio nao esta servindo. Abra o LM Studio, ative o servidor local e rode `lms load <modelo>`.

<br><br>

### LM Studio com CPU 100% e sem progresso

Modelo nao foi para a GPU. Cheque `nvidia-smi`: se GPU em 0% e RAM cheia, ajuste o GPU Offload para o maximo.

<br><br>

### Out of memory ao carregar o modelo

Modelo grande demais para a VRAM. Tente quantizacao menor (Q5_K_M para Q4_K_M), janela de contexto menor, ou KV cache Q4_0 em vez de Q8_0.

### Uma feature nao fecha em 3 tentativas

O `orchestrate.sh` para e pede investigacao manual. Veja o `.log` da feature em `/tmp/harness/run/` para o output real do cline.

<br><br>


### Inspecionar o que o cline fez numa feature (replay)

No modo `CTX=1`, o stream cru de cada feature fica em `/tmp/harness/run/<sprint>.<feat>.jsonl`. Para um replay legivel (thinking, texto, tool calls com args, erros, tokens por turno):

<br><br>

```bash
python3 .harness/scripts/audit.py <sprint> <feat>          # ex: audit.py 01-fundacao feat-001
python3 .harness/scripts/audit.py <sprint> <feat> --tools  # so tool calls + erros
```
