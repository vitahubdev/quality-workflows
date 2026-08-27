# MIGRAR_PROJETO — Adicionar verificação de qualidade a um projeto existente

Procedimento para um projeto que **já está em produção** e ainda não tem verificação.
Faça primeiro num projeto **piloto** (não o mais crítico), veja funcionar, depois repita nos demais.

Pré-requisito: o `quality-workflows` já está no GitHub com a tag `v1` (ver `SETUP.md`).

---

## Antes de começar — escolha do piloto

Bom piloto: um projeto que você conhece bem, que não é crítico se o CI ficar
barulhento nos primeiros dias. Evite começar por Monetarie ou pelas APIs de saúde —
esses são os que MAIS precisam, mas são os que você quer migrar depois de já ter
o procedimento afiado.

---

## Passo 1 — pre-commit na sua máquina

💻 Na raiz do projeto:

```bash
# Copie o template (do quality-workflows/templates/)
cp caminho/para/quality-workflows/templates/.pre-commit-config.yaml .

# Ajuste: se o projeto é só backend, remova o bloco Biome.
#         se é só frontend, remova o bloco Ruff.
#         o bloco Gitleaks NUNCA sai.

# Instale
pipx install pre-commit     # se ainda não tiver
pre-commit install
```

Teste sem commitar nada:

```bash
pre-commit run --all-files
```

> ⚠️ **Este primeiro run vai acusar MUITA coisa** — meses de formatação acumulada.
> É esperado. Ruff e Biome corrigem a maioria sozinhos. Veja o Passo 4.

---

## Passo 2 — o deploy.yml com gate

O projeto já tem um `.github/workflows/deploy.yml` que só faz deploy. Você vai
substituí-lo por um que roda quality ANTES do deploy.

💻:

```bash
# Faça backup do atual, para comparar
cp .github/workflows/deploy.yml .github/workflows/deploy.yml.bak

# Copie o template
cp caminho/para/quality-workflows/templates/deploy.yml .github/workflows/deploy.yml
```

Agora edite o novo `deploy.yml` e ajuste:

1. **`NOME_DO_PROJETO`** na última linha (`cd /opt/apps/NOME_DO_PROJETO`) — o nome
   da pasta no servidor.
2. **`package-dir`** do backend, se o código não está em `backend/app`.
3. **Remova o job que não se aplica:** se o projeto não tem frontend, apague o job
   `frontend` e tire-o do `needs:` do deploy. Se o frontend é vanilla, troque
   `node-quality.yml` por `static-quality.yml`.

Compare com o `.bak` para garantir que o passo de deploy (host, script SSH) ficou igual.

### 🔴 Se você NÃO usar o template — o `permissions` não é opcional

Nem todo projeto quer entregar o `deploy.yml` (o do `apis`, por exemplo, sobe stack
Compose, roda migrations e confere o boot do worker — o template não faz nada disso).
Nesse caso você escreve o chamador à mão, e **tem de copiar os blocos `permissions`
que o template já traz**:

```yaml
jobs:
  backend:
    uses: vitahubdev/quality-workflows/.github/workflows/python-quality.yml@v1
    permissions:
      contents: read
      security-events: write     # <- sem isto, o run MORRE antes de comecar
    with:
      working-directory: backend
      package-dir: backend/app
```

**Por quê:** workflow chamado **não escala permissão** — ele só recebe o que o
chamador concede. Se o repositório está em `Settings → Actions → Workflow permissions:
Read repository contents` (que é o default de muitos), o `python-quality` pede
`security-events: write` para subir o SARIF do Bandit, não recebe, e o Actions
**mata o run em `startup_failure`**.

⚠️ **E o sintoma é traiçoeiro: não há job, não há log, não há anotação.** É idêntico
ao de uma **queda do GitHub Actions** — foi exatamente essa confusão que custou uma
investigação inteira no `apis` em 26–27/08/2026.

📌 **O sinal que separa os dois:** numa queda do Actions **nenhum** run nasce. Aqui,
os outros workflows do mesmo push nascem normalmente e **só o chamador morre**.
**Run vizinho saudável ⇒ o problema é do seu arquivo, não do GitHub.**

Conferir o default do repositório:

```bash
gh api repos/<owner>/<repo>/actions/permissions/workflow
```

---

## Passo 3 — preparar o repositório para os alertas

🌐 Na aba **Settings → Code security and analysis** do projeto:

- Ligue **Dependabot alerts** e **Dependabot security updates**
- Confirme que **Code scanning** está disponível (é onde o SARIF aparece)

Sem isso, os alertas de Bandit/Semgrep/Gitleaks não têm onde aparecer.

---

## Passo 4 — o primeiro run, com cuidado

Não faça push de tudo de uma vez com o gate apertado — você travaria o próprio deploy.

**Ordem recomendada:**

**4a. Formatação primeiro, num commit isolado.**

```bash
# Deixe o Ruff e o Biome arrumarem tudo
ruff check --fix backend/ 2>/dev/null || true
ruff format backend/ 2>/dev/null || true
npx @biomejs/biome check --write frontend/ 2>/dev/null || true

git add -u
git commit -m "style: formatação automática (ruff + biome)"
```

Esse commit é ruído puro (só formatação), fácil de revisar. Nenhuma lógica muda.

**4b. Segredos — resolva antes de qualquer push.**

```bash
# Rode o gitleaks localmente antes de subir
pre-commit run gitleaks --all-files
```

Se acusar segredo real: **rotacione a credencial** (ver `MANUAL_INFRA_BASE.md` §5),
tire do código, ponha no `.env`. Se for falso positivo, adicione à allowlist.

> ⚠️ **`pre-commit run gitleaks` NÃO é o mesmo que o CI.** O hook varre o *working
> tree*; o workflow roda `gitleaks detect`, que varre **todo o histórico do Git**
> (`fetch-depth: 0`). Local verde não prova CI verde — e a recíproca do erro é pior:
> segredo que você já **apagou** do código continua no histórico, e o CI continua
> acusando até a credencial ser rotacionada e o achado tratado.

#### 🔴 Falso positivo: fingerprint, e NUNCA transcreva o valor

Para os falsos positivos do próprio repositório (fixture de teste, quase sempre),
prefira um **`.gitleaksignore` na raiz, por fingerprint**, à allowlist de path ou
regex na config central:

- **fingerprint falha FECHADO** — fixture nova acende vermelho, alguém confere e
  acrescenta a linha. Incômodo pequeno, direção certa.
- **allowlist de `tests/` (ou regex `"teste"`) falha ABERTO** — segredo de verdade
  que vazasse para um arquivo de teste passaria calado, e essa é a única falha
  irreversível que este scanner existe para pegar.

⚠️ **E não escreva os valores no cabeçalho do arquivo.** O gitleaks varre o
`.gitleaksignore` como qualquer outro arquivo: um cabeçalho que cita as três linhas
"para provar que eram fixture" vira **três achados novos**. Aconteceu no `apis` em
27/08/2026 — o CI ficou vermelho por causa do comentário que explicava por que
estava verde. **Descreva a forma do segredo; nunca reproduza o valor.**

📌 **E corrigir o texto não basta:** o commit fica no histórico, e o gitleaks varre
histórico. Em branch de PR que vai por **squash**, o histórico é descartável —
colapse os commits (`git reset --soft <base>` + `git push --force-with-lease`) e o
achado some na raiz, em vez de virar mais fingerprints apontando para um commit que
nunca chega à `main`.

**4c. Só então, o push.**

```bash
git add .
git commit -m "ci: adiciona verificação de qualidade com gate"
git push origin main
```

🌐 Vá à aba **Actions**. Acompanhe o run. O gate está configurado com
`strict-bandit: false` e `strict-mypy: false` por padrão — então Bandit e mypy
**avisam sem travar**. Só Gitleaks e erros de sintaxe bloqueiam de início.

> ⚠️ **Se você PULOU o 4a** (formatação), passe também **`strict-format: false`** no
> `deploy.yml` — senão o `ruff format --check` trava o job sozinho, e você fica sem
> lint, sem tipagem e sem segurança por causa de espaço em branco. Ele é o único
> `strict-*` que começa `true`, de propósito (README, "Perfil de risco").

---

## Passo 5 — olhar os alertas e corrigir

🌐 Aba **Security → Code scanning**. Aqui aparecem os achados.

Prioridade:
1. **Gitleaks** (se sobrou algo) — resolva hoje
2. **Semgrep severidade "error"** — SQL injection, XSS
3. **O resto** — ao longo das semanas, ou "Dismiss" se for aceitável

O fluxo de correção de cada alerta está no `GUIA_OPERACIONAL_QUALIDADE.md`.

---

## Passo 6 — apertar o gate (semanas depois)

Quando a aba Security estiver limpa, torne o gate rígido. No `deploy.yml` do projeto:

```yaml
  backend:
    uses: vitahubdev/quality-workflows/.github/workflows/python-quality.yml@v1
    with:
      working-directory: backend
      package-dir: backend/app
      strict-bandit: true      # agora Bandit trava o deploy
      strict-mypy: true        # agora mypy trava o deploy
```

Para Monetarie e APIs de saúde, aperte antes — idealmente já no Passo 2.

---

## Checklist da migração

- [ ] `.pre-commit-config.yaml` copiado e ajustado à stack
- [ ] `pre-commit install` rodado
- [ ] `deploy.yml` substituído, `NOME_DO_PROJETO` e `package-dir` ajustados
- [ ] **Se escreveu o chamador à mão: `permissions` em cada job** (Passo 2) —
      sem isso o run morre em `startup_failure`, sem log
- [ ] Job do frontend correto (node vs static) ou removido
- [ ] Passo de deploy conferido contra o `.bak`
- [ ] Dependabot + Code scanning ligados
- [ ] Commit de formatação isolado — ou **`strict-format: false`** se pulou
- [ ] Gitleaks limpo (segredos rotacionados se necessário)
- [ ] **`.gitleaksignore` sem valores transcritos** no cabeçalho (Passo 4b)
- [ ] Push feito, quality rodou ANTES de deploy na aba Actions
- [ ] Aba Security mostra alertas com link para a linha
- [ ] `.bak` removido depois de confirmar que o deploy funciona
- [ ] (semanas depois) gate apertado com strict-*: true
