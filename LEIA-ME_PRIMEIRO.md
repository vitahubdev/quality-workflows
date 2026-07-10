# LEIA-ME PRIMEIRO — O que é cada coisa e para onde vai

Você tem **dois tipos de material** aqui. Eles vivem em lugares diferentes e têm propósitos diferentes. Esta é a separação que importa.

---

## TIPO 1 — DOCUMENTAÇÃO (fica no seu Drive)

São manuais de referência. **Nenhum deles vai para o Git.** Você os guarda no Google Drive e anexa no início das conversas de projeto, como já faz hoje.

| Documento | O que é | Mudou? |
|-----------|---------|--------|
| `REFERENCIA_ECOSSISTEMA.md` | Decisões de stack (email, IA, banco, pagamento, segurança) | ✏️ corrigido |
| `MANUAL_INFRA_BASE.md` | Infra comum a qualquer stack (servidor, deploy, rollback, backup) | 🆕 novo (extraído dos 2 antigos) |
| `INFRA_DELTA_VANILLA.md` | Só o que difere no frontend vanilla | 🆕 novo (delta) |
| `INFRA_DELTA_REACT.md` | Só o que difere no frontend React | 🆕 novo (delta) |
| `MANUAL_QUALIDADE_CI.md` | Arquitetura da verificação de qualidade | 🆕 novo |
| `GUIA_OPERACIONAL_QUALIDADE.md` | Como olhar/receber alerta/corrigir no dia a dia | 🆕 novo |
| `MANUAL_TRAEFIK.md` | Webhook com mTLS | ✏️ ajuste pequeno |
| `PROJECT_KICKOFF_TEMPLATE.md` | Roteiro de kickoff de projeto novo | ✏️ corrigido |

**Os dois manuais de infra antigos** (`MANUAL_INFRA_JS_VANILLA.md` e
`MANUAL_INFRA_REACT.md`) **saem de circulação.** Foram substituídos pelo trio
base + 2 deltas. Pode arquivá-los.

### O que mudou na documentação, em resumo

- **Infra desduplicada:** os 2 manuais com ~500 linhas repetidas viraram base + deltas.
- **Rollback:** seção nova (incluindo o caso de migration já aplicada).
- **Rotação de segredo:** procedimento novo, que não existia.
- **Sentry desde o dia 1:** invertida a recomendação antiga ("só com clientes pagantes").
- **`python-jose` → `pyjwt`:** a biblioteca antiga tem CVEs de bypass de assinatura.
- **Kickoff corrigido:** referência a `STACK_REFERENCE.md` (inexistente) apontava
  para o vazio; Traefik e Qualidade entraram na lista de envio; Sprint 0 ganhou
  entregáveis de infra/qualidade explícitos.

---

## TIPO 2 — CÓDIGO (vai para o Git)

A pasta `quality-workflows/` **é um repositório real.** Vai para o GitHub como
`vitahubdev/quality-workflows`. Os projetos apontam para ele.

Isto NÃO é documentação. São arquivos que EXECUTAM quando você dá push num projeto.

```
quality-workflows/
├── SETUP.md                    ← COMECE AQUI: como subir no GitHub
├── MIGRAR_PROJETO.md           ← depois: como adicionar a um projeto existente
├── README.md                   ← visão geral do repositório
├── .github/workflows/          ← os 5 workflows que executam
│   ├── python-quality.yml
│   ├── node-quality.yml
│   ├── static-quality.yml
│   ├── semgrep.yml
│   └── secrets-scan.yml
├── config/                     ← as regras centrais (ruff, bandit, biome, gitleaks)
└── templates/                  ← o que cada projeto copia
    ├── deploy.yml
    └── .pre-commit-config.yaml
```

---

## A relação entre os dois

O `MANUAL_QUALIDADE_CI.md` (documentação) **descreve** o `quality-workflows` (código).

Um é a planta da casa. O outro é a casa. Você lê a planta para entender; você mora na casa.

---

## Ordem de execução — o caminho correto

```
1. Guardar a documentação (TIPO 1) no Drive, substituindo os manuais antigos
        │
2. Subir o quality-workflows no GitHub          → siga quality-workflows/SETUP.md
   (criar repo + push + tag v1)
        │
3. Migrar UM projeto piloto                     → siga quality-workflows/MIGRAR_PROJETO.md
   (não o mais crítico; ver funcionar primeiro)
        │
4. Repetir a migração nos demais projetos
        │
5. Semanas depois, apertar o gate nos projetos que já estão limpos
```

O passo 2 vem ANTES do 3. Se você migrar um projeto antes do repositório existir,
ele aponta para o vazio e o deploy quebra.

---

## O que NÃO fazer

- **Não suba a documentação (TIPO 1) para o Git.** Ela vive no Drive.
- **Não copie os manuais para dentro de cada projeto.** O `CLAUDE.md` de cada
  projeto os referencia pelo nome; eles são referência externa compartilhada.
- **Não comece a migração pelo projeto mais crítico.** Piloto primeiro.
- **Não aperte o gate no dia 1** num projeto com código legado. Ele avisa primeiro,
  trava depois (só o Gitleaks trava desde o início).
