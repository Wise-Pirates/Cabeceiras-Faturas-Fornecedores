# Relatório — Projeto 21 — Terça 2026-07-15

**Cliente:** Cabeceiras.pt (Jorge Brandão)
**Workflow n8n:** `62wyOKnNBy0bnJUw` — **69 nodes** — DESATIVADO no fim da sessão
**Sessão:** longa (manhã → fim tarde)
**Resultado:** 13 ECs entregues ao Jorge (EC2602356-364 + EC2602374-376 + EC2602378), 5 bugs corrigidos, sistema migrado para env vars + publicado no GitHub

---

## 1. Ponto de partida (manhã 15/07)

- Sistema deixado ativo em 14/07 com Claude Sonnet 4.5 como OCR primário de linhas
- 10 ECs entregues ao Jorge para validação (EC2602209/232 pré-existentes + EC2602274/278/293/294/337/338/339/340)
- Feedback do Jorge sobre problemas específicos → decisão de reprocessar

### Feedback do Jorge (síntese)
- **EC2602209 Flex FT 31/206437** — não identificou produtos (apesar de existirem no ficheiro)
- **EC2602232 FSM FAT 385126/2705** — acertou códigos mas errou preços e desconto de linha
- **EC2602274 Topbois FT 2026/1414** — reprocessar (já com produtos no Excel)
- **EC2602278 Topbois FT 2026/1505** — reprocessar (códigos já no Excel)
- **EC2602294 Eurofins** — fornecedor errado, era PLASPUR (F00080 NIF 504871226) — sistema atribuiu Eurofins (F01291 NIF 510718531)

Jorge apagou as 4 no BC. Reprocessamento iniciado.

---

## 2. Bug 1 — NIF do CLIENTE Jorge extraído pelo Document AI

### Sintoma
Documento AI extraía consistentemente `510718531` (NIF do próprio Jorge Brandão Gonçalves) em vez do NIF do FORNECEDOR nas faturas Ferragens Carreiras (real: 503315745), Würth (real: 500302030) e PLASPUR (real: 504871226).

### Causa raiz
Layout típico das faturas PT tem **duas caixas** com NIFs:
- Cabeçalho: `Contribuinte: XXXXXXXX` (fornecedor)
- Tabela cliente: `Nº de Contribuinte: XXXXXXXX` (cliente Jorge)

Document AI (entity `supplier_tax_id`) apanhava a segunda caixa por erro.

### Impacto histórico
A EC2602294 Eurofins (08/07) — a que Jorge reportou — foi criada por este bug. Naquela altura o BC ainda tinha F01291 (Eurofins) com NIF `510718531` cadastrado (entretanto corrigido para `513564543`), permitindo match acidental.

### Fix cirúrgico no `Avaliar Resultado BC`
Duas alterações:
1. **Splitting de nome inclui hífen:** `/[\s,.\-;:/()]+/` (antes só `/[\s,.]+/`)
   - Efeito: "PLASPUR-SOCIEDADE MAT.PLASTICOS," → primeira palavra passa de "PLASPUR-SOCIEDADE" (contains 0 no BC) para "PLASPUR" (contains F00080)
2. **Aceitar match único com termo distintivo:** se `nameVendors.length === 1 && searchTerm.length >= 5` → match direto
   - Cobre: WÜRTH↔"Wurth Portugal" (BC tem `contains(name,'WURTH')` = 1), PLASPUR↔"Plaspur-Soc.Mat.Plásticos" (1 candidato)
   - FERRAGENS mantém exact match (15 candidatos "Ferragens" no BC → força exact) — F00044 "Ferragens Carreiras, Lda." bate

### Auditoria de segurança
Antes de aplicar single-match: query `contains(name,'X')` para todos os fornecedores em risco:
- `PLASPUR` → 1 match (seguro)
- `WURTH` → 1 match (seguro)
- `FERRAGENS` → 15 matches (não usa single-match, exige exact)
- Threshold `>= 5 chars` mitiga colisões

---

## 3. Bug 2 — Data PT `DD.MM.YYYY` quebra BC OData

### Sintoma
```
"Cannot convert the literal '02.07.2026' to the expected type 'Edm.Date'"
```
Würth escreve data com pontos (`02.07.2026`). BC OData v4 exige ISO `2026-07-02`. Document AI devolveu literal.

### Fix em `Parse e Validar`
Nova função `normalizeDate()` que converte:
- `2026-07-09` → `2026-07-09` (ID)
- `02.07.2026` → `2026-07-02`
- `02/07/2026` → `2026-07-02`
- `02-07-2026` → `2026-07-02`
- `2026-07-09T00:00:00Z` → `2026-07-09`

Devolve tuple `{value, converted}` para sinalizar quando houve conversão (usado depois em flags).

### Retrocompatibilidade
Datas já ISO passam iguais (testado com todas as 9 ECs anteriores). Zero regressão.

---

## 4. Bug 3 — Flags no `postingDescription` não sinalizavam correções

### Sintoma
EC2602367 Würth passou sem qualquer flag apesar de:
- Vendor identificado por **nome** (não NIF — plano B do bug 1)
- Data **convertida** de formato PT

Jorge não sabia que precisava rever nada.

### Fix em cascata (3 nodes)

**a) `Avaliar Resultado BC` — propagar `matchType`:**
Já calculava internamente `matchType = 'NIF' | 'nome exato' | 'nome unico distintivo'` mas não o expunha no output. Adicionado ao `return`.

**b) `Preparar EC Header` — construir postingDescription:**
```js
const fornecedorPlanoB = matchTypeUsado && matchTypeUsado !== 'NIF';
const dataConvertidaPT = dados.data_convertida_pt === true;
if (fornecedorPlanoB) marcasFlag += ' - VERIFICAR FORNECEDOR';
if (dataConvertidaPT) marcasFlag += ' - VERIFICAR DATA';
```

**c) `Atualizar Datas EC` — refazer no PATCH final:**
Descoberta importante: o PATCH final refaz `postingDescription` (não usa o do POST). A expressão IIFE inline tinha que ser atualizada igualmente. Sem esta terceira alteração, as flags eram sobrescritas.

### Flags finais possíveis (combinam)
- `AUTOMAÇÃO` — tudo OK, sem intervenção
- `- ATENÇÃO VALORES` — Σ linhas ≠ total OCR
- `- ITEM SEM MATCH` — usou placeholder `ME003007`
- `- VERIFICAR FORNECEDOR` — match por nome (não NIF)
- `- VERIFICAR DATA` — data convertida de formato PT

---

## 5. Bug 4 — Claude leu quantidades com separador PT como valores milhões

### Sintoma
**EC2602373 PLASPUR** — total no BC saiu como **4.399.402,50€** (4 milhões!) quando fatura real é **4.399,40€** (fator 1000x).

### Investigação
Fatura PLASPUR tinha:
```
Linha 1: SACO NATURAL — Quant: 1 255,000 KG — Preço 2,85 — Total 3.576,75€
Linha 2: PALETES DE MADEIRA — Quant: 1,000 UN — sem preço
```

Formato PT: espaço = separador milhares, vírgula = decimal → `1 255,000` = **1255 kg**, `1,000` = **1 palete**.

Claude interpretou como:
- `1 255,000` → `1255000` (1.255 milhões)
- `1,000` → `1000` (mil paletes)

### Fix 1 (não funcionou sozinho) — reforço do prompt Claude
Adicionadas regras explícitas:
```
5. Números PT: vírgula = decimal, ponto/espaço = separador milhares.
   - "1.255,00" ou "1 255,00" = 1255.00 — NUNCA 1255000
   - "1,000" = 1.000 — NUNCA mil
```

Testado: Claude continuou a interpretar erradamente. Reforço de prompt insuficiente para este layout.

### Fix 2 (definitivo) — sanity check no `Merge Linhas GPT-5`
```js
if (totalOcr > 0 && soma > 0) {
  const razao = soma / totalOcr;
  if (razao > 5 || razao < 0.2) {
    return [{ json: {
      ...parse,
      gpt5_fallback: `claude_rejeitado_magnitude(claude=${soma} vs ocr=${totalOcr}, ${razao}x)`,
      dados_fatura: dados
    }}];
  }
}
```

Quando Claude diverge por ordem de magnitude (>5x ou <0.2x do total OCR), rejeita as linhas → sistema cai em placeholder ME003007 com total OCR (comportamento pré-existente).

### Simulação retrospetiva (13 ECs)
| EC | Fornecedor | Razão Σ/OCR | Ação |
|----|-----------|-------------|------|
| 356 | Topbois | 1.000 | ✓ aceita |
| 357 | Flex (desc 2%) | 1.020 | ✓ aceita |
| 358-360, 362 | Topbois | 1.000 | ✓ aceita |
| 361 | FSM (0.87€ diff) | 1.004 | ✓ aceita |
| 363 | Meilex | 1.000 | ✓ aceita |
| 364 | Ernesto Romano | 1.000 | ✓ aceita |
| 374 | Ferragens 28€ | 1.000 | ✓ aceita |
| 375 | Ferragens 119€ | 1.000 | ✓ aceita |
| 376 | Würth | 1.000 | ✓ aceita |
| **378** | **PLASPUR (bug)** | **1000.001** | **❌ REJEITA** |

Zero regressão nas 12 boas. Threshold seguro.

---

## 6. Bug 5 (menor) — timing de deploy vs poll

Cronologia:
- 12:09 — poll pega Würth
- 12:11 — deploy do fix `Atualizar Datas EC`
- Würth processado com código antigo (sem flags)

Aprendizagem: sempre confirmar `updatedAt` do workflow vs `startedAt` da exec antes de investigar bugs. Reprocessado após — ficou correto.

---

## 7. Ciclo de deploys da sessão

~10 iterações apply → test → adjust:

1. Adicionar splitting hífen + single-match distintivo (`Avaliar Resultado BC`)
2. `normalizeDate` (`Parse e Validar`)
3. Flags VERIFICAR FORNECEDOR/DATA — 1ª tentativa (bug variável `validadoDoc` inexistente)
4. Fix variável (`data`/`dados`)
5. Descoberta: `Atualizar Datas EC` refaz postingDescription
6. Fix expressão IIFE no `Atualizar Datas EC`
7. Reprocessamento 4 ECs → flags aparecem
8. Descoberta PLASPUR 4M€
9. Reforço prompt Claude (não bastou)
10. Sanity check magnitude no `Merge Linhas GPT-5`
11. Reprocessamento PLASPUR → correto

**Total ECs no fim:** 13 (EC2602356-364, EC2602374-376, EC2602378).

---

## 8. 13 ECs finais entregues (relação para Jorge)

| EC | Fornecedor | BC | Fatura | Data | Total c/IVA | Flags |
|---|---|---|---|---|---:|---|
| EC2602356 | Topbois | F00896 | FT 2026/1414 | 01/06 | 1.213,27€ | — |
| EC2602357 | Flex2000 | F00001 | FT 31/206437 | 11/06 | 8.711,35€ | ATENÇÃO VALORES (desc 2%) |
| EC2602358 | Topbois | F00896 | FT 2026/1445 | 03/06 | 3.447,11€ | ITEM SEM MATCH |
| EC2602359 | Topbois | F00896 | FT 2026/1461 | 05/06 | 2.362,07€ | ITEM SEM MATCH |
| EC2602360 | Topbois | F00896 | FT 2026/1480 | 08/06 | 1.834,85€ | ITEM SEM MATCH |
| EC2602361 | FSM | F00041 | FT 385126/2705 | 16/06 | 247,45€ | ATENÇÃO VALORES (Claude leu 0.588 vs 0.58) |
| EC2602362 | Topbois | F00896 | FT 2026/1505 | 11/06 | 2.750,22€ | ITEM SEM MATCH |
| EC2602363 | Meilex | F00136 | FT FA.2026/3897 | 09/07 | 2.686,76€ | ITEM SEM MATCH |
| EC2602364 | Ernesto Romano | F00094 | FT 2601/000099 | 07/07 | 2.558,40€ | — |
| EC2602374 | Ferragens Carreiras | F00044 | FT 356126/2248 | 09/07 | 28,19€ | ATENÇÃO VALORES · ITEM SEM MATCH |
| EC2602375 | Ferragens Carreiras | F00044 | FT 356126/2261 | 09/07 | 119,25€ | ATENÇÃO VALORES · ITEM SEM MATCH |
| EC2602376 | Würth Portugal | F00648 | FT 912080041 | 02/07 | 94,71€ | ITEM SEM MATCH · VERIFICAR DATA |
| EC2602378 | Plaspur | F00080 | FT 1445 | 08/07 | 4.399,40€ | ITEM SEM MATCH · VERIFICAR FORNECEDOR |
| **TOTAL c/IVA** | | | | | **30.453,03€** | |

---

## 9. Migração de secrets para env vars (fim de sessão)

### Motivação
Ao criar repo GitHub, Secret Scanning bloqueou push 3× — 3 `client_secret` Azure hard-coded no workflow.

### Feedback crítico do user
> "todo sistema já tem que ser criado nessa linha" — sistemas nascem sem secrets expostos, não são "limpos" retroativamente

Guardado em memória permanente: `feedback_secrets_desde_inicio.md`.

### Solução aplicada
- 3 HTTP nodes (`Obter Token Graph`, `Obter Token BC`, `Obter Token BC DEV`) tinham `client_secret` literal no bodyParameters
- Substituído por `{{ $env.GRAPH_CLIENT_SECRET }}` / `{{ $env.BC_PROD_CLIENT_SECRET }}` / `{{ $env.BC_DEV_CLIENT_SECRET }}`
- Env vars adicionadas ao Railway serviço "Primary" via `railway variables --set`
- Redeploy automático pelo Railway
- Testado: 3 OAuth passam via env vars

### Resultado
| Onde | Contém secret? |
|---|---|
| n8n workflow | Só `{{ $env.X }}` |
| Ficheiro local | Só `{{ $env.X }}` |
| GitHub repo | Só `{{ $env.X }}` |
| Railway Variables | Sim (encriptado, apenas fonte) |

Rotação recomendada dos 3 secrets no Azure (não urgente, estiveram no filesystem local durante ~semanas).

---

## 10. Publicação no GitHub

- **Repo:** https://github.com/robsonadvincula-svg/sistema-automacao-faturas (privado)
- Conta pessoal `robsonadvincula-svg` — não tinha permissões admin em `Wise-Pirates` org
- 2 commits: `init:` (baseline com `<REDACTED>`) → `refactor:` (env vars limpos)
- `.gitignore` bloqueia: chave Google SA, Excel cliente, faturas reais, screenshots BC, mp3, `.bak`
- 21 ficheiros commitados (docs `.md` + 2 workflows + README + `.gitignore`)

---

## 11. Memórias criadas/atualizadas

- **`feedback_parar_workflow_antes_corrigir.md`** — SEMPRE desativar workflow antes de qualquer PUT/alteração
- **`feedback_secrets_desde_inicio.md`** — sistemas nascem sem secrets expostos; grep de padrões antes de qualquer push
- **`project_faturas_fornecedores.md`** — atualizada com sessão 15/07 (5 bugs + 13 ECs + estado final)

---

## 12. Pendências

### Aguarda Jorge
- Validação das 13 ECs (EC2602356-364 + EC2602374-376 + EC2602378)
- Se OK → sistema continua em produção
- Se pedir mudanças → aplicar

### Recomendação tua (não urgente)
- Rotar 3 client secrets Azure no Portal → atualizar env vars Railway → workflow continua sem tocar em código

### Bug conhecido adiado
- FSM: Claude lê preço `0.588` quando fatura mostra `0,5880` (zeros à direita = 0.58). Diff 0.87€ na EC2602361. Adiado por Jorge ("deixa assim vamos ver").

---

## 13. Regras críticas reforçadas nesta sessão

1. **SEMPRE parar workflow antes de qualquer alteração** — user chamou atenção após reactivação por reflexo
2. **SEMPRE criar sistemas sem secrets expostos** — não é aceitável "limpar depois"
3. **Fix cirúrgico > fix abrangente** — cada mudança avaliada em relação às 12 ECs boas para garantir zero regressão
4. **Sempre confirmar timing deploy vs poll** — evita investigar "bugs" que são apenas execuções pré-deploy

---

## 14. Chaves e endpoints (referência)

- **n8n workflow:** `62wyOKnNBy0bnJUw` (69 nodes, desativado)
- **Anthropic credential:** `srXSApQJ2OBjvnxL` (Cabeceiras_Orcamentos)
- **OpenAI credential:** `CogELPgsbfrLHzNg`
- **Google Document AI:** projeto `cabeceiras-ocr`, processor `d9230429c35852ce` (EU)
- **BC PROD OData:** `.../PROD/ODataV4/Company('Jorge%20Brand%C3%A3o%20Gon%C3%A7alves')`
- **Log_Faturas Excel:** `01JSSR6ZOQEYD67CHDWZDYCTBQAOQPLZKA` / Tabela1
- **Codigos Fornecedores Excel:** `01JSSR6ZOQ4BV4R2JA4JEJJD4BPSVMLY2K` / MapeamentoItems
- **Pasta faturas por inserir:** `01JSSR6ZOWI4JQ7ADEZFHZT24VNIWIU42R`
- **Pasta faturas inseridas:** `01JSSR6ZLDQWWQPDGYP5CZJAQFOZUTJGI2`
- **Pasta faturas_com_erro:** `01JSSR6ZMVOBS354FCNVDZG2CAXFQFJUR6`

Env vars Railway (Primary): `GRAPH_CLIENT_SECRET`, `BC_PROD_CLIENT_SECRET`, `BC_DEV_CLIENT_SECRET` (também existe `BC_CLIENT_SECRET` legado com mesmo valor de PROD).

---

## 15. Resumo executivo

Dia intensivo com **5 bugs corrigidos** em ciclo apply→test→adjust, resultando em **13 ECs consistentes** entregues ao Jorge para validação (total 30.453€ c/IVA). Sistema **migrado para env vars** — zero secrets no repo GitHub agora publicado (`robsonadvincula-svg/sistema-automacao-faturas`, privado). Workflow **desativado** aguardando resposta do Jorge.

Regras críticas guardadas em memória permanente: parar workflow antes de qualquer alteração; sistemas nascem sem secrets expostos.
