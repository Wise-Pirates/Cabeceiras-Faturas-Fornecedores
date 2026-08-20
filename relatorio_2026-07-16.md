# Relatório — Projeto 21 — Quinta 2026-07-16

**Cliente:** Cabeceiras.pt
**Foco:** Publicação do projeto no GitHub + migração de secrets para env vars Railway

---

## 1. Contexto

Sessão dedicada a colocar o projeto sob controlo de versões e resolver secrets hard-coded no workflow.

---

## 2. Criação repo GitHub

- **Repo:** `robsonadvincula-svg/sistema-automacao-faturas` (privado)
- 21 ficheiros commitados: README, workflows, docs, relatórios, `.gitignore`

Push inicial bloqueado pelo GitHub Secret Scanning — encontrou 3 `client_secret` Azure em `workflow_fase1_ocr.json`:
- `[REDACTED]` (Graph)
- `[REDACTED]` (BC PROD)
- `[REDACTED]` (BC DEV)

Também 3 secrets em `.md` (planeamento, ESTADO-ATUAL, relatorio_2026-06-29).

### Fix
1. Substituir todos os secrets por placeholders `<REDACTED_*>` nos ficheiros (JSON + MD)
2. `rm -rf .git` + reinit para limpar histórico local com secrets
3. Push forçado → aceito

---

## 3. Migração para env vars Railway (aliás Jorge)

**Novo padrão:** secrets NUNCA no ficheiro exportado. n8n usa `{{ $env.NOME_SECRET }}`, valores só no Railway.

### Alterações no workflow
- `Obter Token Graph`: `client_secret={{ $env.GRAPH_CLIENT_SECRET }}`
- `Obter Token BC`: `client_secret={{ $env.BC_PROD_CLIENT_SECRET }}`
- `Obter Token BC DEV`: `client_secret={{ $env.BC_DEV_CLIENT_SECRET }}`

### Railway config
- Env vars adicionadas em Primary + Worker 1/2/4
- Também setado `N8N_BLOCK_ENV_ACCESS_IN_NODE=false` (n8n bloqueia `$env` por defeito)

### Testado
3 OAuth tokens (Graph + BC PROD + BC DEV) passam via env vars. Workflow funciona igual.

---

## 4. Documentação criada

- `ONBOARDING.md` (359 linhas) — guia para novos colaboradores:
  - Contexto de negócio
  - Setup de acessos
  - Arquitetura + fluxo 22 passos
  - Playbook operacional (8 tarefas comuns com comandos prontos)
  - Gotchas críticos
  - Ordem de leitura dos ficheiros do repo
  - Passos concretos dia 1
  - O que o projeto NÃO faz

---

## 5. Feedback do dia — regra `secrets-desde-inicio`

Robson deu regra explícita: **sistemas nascem sem secrets expostos, não são "limpos" retroativamente**.

Guardado em `feedback_secrets_desde_inicio.md`:
- n8n → usar credentials nativas
- Scripts → env vars
- `.env` no `.gitignore` antes do 1º commit
- Grep de padrões antes de qualquer push

---

## 6. Estado final

- Workflow: DESATIVADO (fim sessão)
- Repo GitHub: publicado privado
- Secrets: só no Railway env vars — zero exposição
- ~10 ECs entregues ao Jorge (das sessões anteriores) aguardavam validação
