# quality-workflows

Workflows reutilizáveis de análise e verificação de código, compartilhados por todos os projetos.

**A ideia:** em vez de copiar as mesmas ~60 linhas de "instale o Ruff, rode o Bandit, rode o Gitleaks" em cada projeto, essas instruções vivem aqui. Cada projeto referencia com uma linha. Regra muda num lugar, propaga para todos.

Documentação completa de como funciona, como olhar os alertas e como corrigir: `MANUAL_QUALIDADE_CI.md` (no repositório de documentação do ecossistema).

---

## O que roda

| Workflow | Ferramentas | Linguagem |
|----------|-------------|-----------|
| `python-quality.yml` | Ruff, mypy, Bandit, pip-audit | Python |
| `node-quality.yml` | Biome, tsc, npm audit | React/Vite |
| `static-quality.yml` | Biome | HTML/CSS/JS vanilla |
| `semgrep.yml` | Semgrep (taint analysis) | cross-language |
| `secrets-scan.yml` | Gitleaks | qualquer |

---

## Como um projeto usa

No `.github/workflows/deploy.yml` do projeto:

```yaml
jobs:
  backend:
    uses: alrm1909/quality-workflows/.github/workflows/python-quality.yml@v1
    with:
      working-directory: backend
      package-dir: backend/app
    permissions:
      contents: read
      security-events: write

  secrets:
    uses: alrm1909/quality-workflows/.github/workflows/secrets-scan.yml@v1
    permissions:
      contents: read
      security-events: write

  deploy:
    needs: [backend, secrets]      # o gate: deploy só roda se passou
    # ...
```

Templates prontos em `templates/`.

---

## Versionamento

Os projetos apontam para uma **tag** (`@v1`), não para `@main`. Assim, um commit
que quebre um workflow aqui não derruba o deploy de todos os projetos ao mesmo tempo.

Para publicar uma versão:

```bash
git tag -f v1        # move a tag v1 para o commit atual
git push -f origin v1
```

Mudanças que quebram compatibilidade (renomear input, remover workflow) → nova tag maior (`v2`),
e os projetos migram um a um.

---

## Estrutura

```
quality-workflows/
├── .github/workflows/      os 5 workflows reutilizáveis
├── config/                 configs compartilhados (ruff, bandit, biome, gitleaks)
├── templates/              o que se copia para cada projeto
└── README.md
```

---

## Perfil de risco

Nem todo projeto precisa de tudo. Resumo (detalhe em `MANUAL_QUALIDADE_CI.md` §7):

- **Gitleaks:** todos, sempre, bloqueante.
- **Ruff / Biome:** todo projeto da linguagem.
- **Bandit / Semgrep / pip-audit:** backend com input externo.
- **Gate bloqueante:** Monetarie e APIs de saúde desde o dia 1. Demais, avisa primeiro
  (`strict-*: false` nos inputs).
