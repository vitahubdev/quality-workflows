# ARQUITETURA — O que é cada arquivo e por que existe

Referência de consulta. Se você abriu este repositório e quer entender o que é
cada peça sem ler os 12 arquivos, está no lugar certo.

Para **como usar** (subir, migrar projeto), veja `SETUP.md` e `MIGRAR_PROJETO.md`.
Este documento é o **mapa**, não o manual de operação.

---

## A ideia central em uma frase

Em vez de copiar as mesmas verificações de código dentro de cada projeto, elas
ficam aqui, num lugar só, e os projetos apontam para cá com uma linha.

---

## O mapa

```
quality-workflows/
│
├── .github/workflows/     ← as RECEITAS que executam (o coração)
├── config/                ← as REGRAS que as receitas seguem
├── templates/             ← o que cada PROJETO copia para si
│
├── README.md              ← visão geral
├── ARQUITETURA.md         ← este arquivo (o mapa)
├── SETUP.md               ← como subir no GitHub (uma vez)
└── MIGRAR_PROJETO.md      ← como plugar num projeto existente
```

Três pastas com papéis distintos: **receitas** executam, **regras** definem o rigor,
**templates** são o que se copia. O resto é documentação.

---

## 1. `.github/workflows/` — as receitas que executam

Cada arquivo aqui é um "workflow reutilizável" do GitHub. Não roda sozinho —
roda quando um projeto o chama. É o mecanismo `on: workflow_call`.

| Arquivo | O que faz | Roda quando |
|---------|-----------|-------------|
| `python-quality.yml` | Ruff (formatação/lint), mypy (tipagem), Bandit (segurança), pip-audit (CVE em dependência) | O projeto tem backend Python |
| `node-quality.yml` | Biome (formatação/lint), tsc (tipagem), npm audit (CVE) | O projeto tem frontend React/Vite |
| `static-quality.yml` | Biome apenas | O projeto tem frontend vanilla (sem package.json) |
| `semgrep.yml` | Semgrep (busca vulnerabilidade rastreando o caminho do dado) | Sempre — funciona em qualquer linguagem |
| `secrets-scan.yml` | Gitleaks (procura senha/token vazado no código e no histórico) | Sempre — é o mais importante |

**Por que separados e não um só:** cada projeto ativa só os que precisa. Uma landing
page vanilla chama `static-quality` + `secrets-scan` e ignora o resto. Uma API Python
chama `python-quality` + `semgrep` + `secrets-scan`. Você monta como Lego.

**O detalhe técnico que vale saber:** cada workflow faz DOIS checkouts — o código do
projeto (para verificar) e este repositório (para pegar as regras da pasta `config/`).
É isso que torna a regra verdadeiramente central: o projeto não guarda cópia das regras,
ele as baixa daqui a cada execução.

---

## 2. `config/` — as regras que as receitas seguem

As receitas dizem "rode o Ruff". Estes arquivos dizem "com QUAIS regras".

| Arquivo | Controla | Exemplo do que define |
|---------|----------|----------------------|
| `ruff.toml` | Ruff (Python) | Linha de 100 caracteres, quais checagens ativar, o que ignorar em testes |
| `bandit.yaml` | Bandit (Python) | Quais pastas pular, quais alertas silenciar |
| `biome.json` | Biome (JS/TS/CSS) | Aspas simples, ponto-e-vírgula só quando necessário, avisar sobre innerHTML perigoso |
| `gitleaks.toml` | Gitleaks (segredos) | Herdar as ~150 regras padrão + allowlist de falsos positivos do ecossistema (ex.: `.env.example`) |

**Por que ficam aqui e não em cada projeto:** para mudar uma regra uma vez e valer
para todos. Se amanhã você decidir que a linha deve ter 120 caracteres em vez de 100,
edita `ruff.toml` aqui, e todos os projetos passam a usar 120 no próximo push. Sem
tocar em nenhum projeto.

---

## 3. `templates/` — o que cada projeto copia

Diferente das duas pastas acima (que ficam só aqui), estes arquivos são feitos para
serem **copiados para dentro de cada projeto**. Eles são o ponto de contato.

| Arquivo | Vai para | O que é |
|---------|----------|---------|
| `deploy.yml` | `.github/workflows/` do projeto | O arquivo que chama as receitas e, se passarem, faz o deploy. É onde mora o "gate". |
| `.pre-commit-config.yaml` | Raiz do projeto | Roda Ruff/Biome/Gitleaks na SUA máquina, antes do commit, em ~2 segundos. |

**Por que são template e não referência:** o `deploy.yml` precisa ser ajustado por
projeto (nome da pasta no servidor, qual frontend, etc.). Então ele é um ponto de
partida que você copia e edita — não algo que se aponta remotamente.

**A diferença entre os dois pontos de verificação:**
- `.pre-commit-config.yaml` → na sua máquina, rápido, pega o óbvio antes do commit
- `deploy.yml` → no GitHub, completo, pega tudo antes do deploy

Os dois existem de propósito: o primeiro te poupa de esperar o CI para erros bobos;
o segundo é a rede de proteção real que impede código ruim de chegar ao servidor.

---

## 4. Os documentos (README, SETUP, MIGRAR, este)

| Arquivo | Quando ler |
|---------|-----------|
| `README.md` | Primeira olhada, visão geral |
| `ARQUITETURA.md` | Quando quer entender o que é cada coisa (você está aqui) |
| `SETUP.md` | Uma vez, ao subir o repositório no GitHub |
| `MIGRAR_PROJETO.md` | Cada vez que for plugar um projeto novo |

O manual completo da operação do dia a dia (como olhar alertas, como corrigir) NÃO
está aqui — está no `GUIA_OPERACIONAL_QUALIDADE.md`, que vive junto da documentação
do ecossistema, porque é sobre usar, não sobre a estrutura interna deste repositório.

---

## Como as peças se conectam — o fluxo completo

```
VOCÊ escreve código no projeto
        │
        ▼
git commit
        │
        ├──► .pre-commit-config.yaml (copiado para o projeto)
        │       roda Ruff + Biome + Gitleaks NA SUA MÁQUINA
        │       └─ falhou? commit barrado, conserta e refaz
        ▼
git push
        │
        ▼
deploy.yml (copiado para o projeto) é acionado no GitHub
        │
        ├──► chama python-quality.yml ─┐
        ├──► chama node-quality.yml    │  (as receitas, AQUI neste repo)
        ├──► chama semgrep.yml         │
        ├──► chama secrets-scan.yml ───┘
        │         │
        │         └─ cada receita baixa as regras de config/ e executa
        │
        ├─ tudo passou? ──► deploy roda, servidor atualiza
        └─ algo falhou? ──► deploy NÃO roda; alerta na aba Security do projeto
```

O projeto tem só dois arquivos (o `deploy.yml` e o `.pre-commit-config.yaml`, copiados
dos templates). Todo o resto — as receitas e as regras — vive aqui e é referenciado
remotamente. Muda aqui, muda para todos.
