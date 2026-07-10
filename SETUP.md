# SETUP — Subir o quality-workflows no GitHub

Passo a passo para transformar esta pasta num repositório real e funcional.
Faça isto **uma vez**. Depois os projetos passam a consumi-lo.

---

## Pré-requisito

Uma conta GitHub (`vitahubdev`) e Git configurado na máquina.

---

## 1. Criar o repositório no GitHub

🌐 Em `https://github.com/new`:

- **Owner:** vitahubdev
- **Repository name:** `quality-workflows`
- **Visibilidade:** Private (recomendado)
- **NÃO** inicialize com README (esta pasta já tem um)

Clique em *Create repository*.

> ⚠️ **Repositório privado + workflow reutilizável privado:** funciona, mas só para
> repositórios da MESMA conta/organização. Como todos os seus projetos estão sob
> `vitahubdev`, está tudo certo. Se algum projeto estiver em outra org, ou este repo
> precisa ser público, ou o projeto precisa migrar para a mesma org.

---

## 2. Subir os arquivos

💻 Na sua máquina, dentro desta pasta (`quality-workflows/`):

```bash
git init
git add .
git commit -m "chore: workflows de qualidade reutilizáveis"
git branch -M main
git remote add origin git@github.com:vitahubdev/quality-workflows.git
git push -u origin main
```

---

## 3. Criar a tag v1

Os projetos apontam para `@v1`, não para `@main`. Isso protege: um commit que
quebre um workflow aqui não derruba o deploy de todos os projetos ao mesmo tempo.

💻 Ainda nesta pasta:

```bash
git tag v1
git push origin v1
```

Pronto. O repositório está publicado e a tag `v1` existe.

---

## 4. Permitir que os projetos usem workflows privados

Se o `quality-workflows` é **privado**, cada projeto que for consumi-lo precisa ter
permissão de leitura. Uma vez, por organização:

🌐 `https://github.com/settings/actions` (ou, se for organização,
`https://github.com/organizations/SUA-ORG/settings/actions`)

Em *Actions permissions* → seção sobre acesso entre repositórios, permita que
workflows de `quality-workflows` sejam usados por outros repositórios da conta.

> Se todos os repositórios são da conta pessoal `vitahubdev`, geralmente já funciona
> sem ajuste. Se der erro `workflow was not found` num projeto, é aqui que se resolve.

---

## 5. Validar

Antes de migrar um projeto real, confirme que o repositório está saudável:

🌐 Na aba *Actions* do `quality-workflows`, não deve haver erro (os workflows
reutilizáveis não rodam sozinhos — só quando chamados — então a aba fica vazia,
o que é o esperado).

Confirme que a tag existe: `https://github.com/vitahubdev/quality-workflows/tags`
deve listar `v1`.

---

## Depois

Com o repositório no ar e a tag `v1` criada, siga para a migração de um projeto
piloto. O procedimento está em `MIGRAR_PROJETO.md`.

---

## Manutenção futura

Quando quiser mudar uma regra (ex.: apertar o Ruff, atualizar versão de uma action):

```bash
# edite o arquivo aqui, depois:
git add .
git commit -m "feat: nova regra X"
git push origin main

# mova a tag v1 para o commit novo (todos os projetos passam a usar):
git tag -f v1
git push -f origin v1
```

Mudança que quebra compatibilidade (renomear input, remover workflow) → crie `v2`
em vez de mover `v1`, e migre os projetos um a um trocando `@v1` por `@v2`.
