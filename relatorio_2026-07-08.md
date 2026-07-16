# Relatório — Projeto 21 — Quarta 2026-07-08

**Cliente:** Cabeceiras.pt (Jorge Brandão)
**Workflow n8n:** `62wyOKnNBy0bnJUw` — 63 nodes — ATIVO no fim da sessão
**Sessão:** 09h → 18h30 local (Portugal WEST = UTC+1)
**Resultado:** Sistema **operacional em produção** com fallback robusto para divergências OCR — Jorge só precisa rever linhas placeholder no BC.

---

## 1. Objetivo do dia

Continuar validação do sistema com faturas reais (PDF-scan digitalizados iPhone) do Jorge. Ontem (07/07) resolvido: GPT-5 + Responses API + Document AI. Hoje: apanhar bugs residuais em produção com múltiplas faturas TOPBOIS e FSM.

---

## 2. Bugs descobertos hoje

### Bug 1 — Loop SplitInBatches v3 processa mal múltiplas faturas
**Sintoma:** quando pasta OneDrive tinha 2+ ficheiros, `Loop Faturas` (SplitInBatches v3) corria mas não emitia items para o resto do fluxo. Só a 1ª ia até ao final; as outras ficavam presas.

**Solução (opção A):** modificar `Filtrar Faturas` para devolver APENAS 1 ficheiro por poll (o mais antigo). Cada poll processa 1 fatura. Trade-off: 3 faturas na pasta = 9 min para acabar (poll 3min).

```js
const ordenados = validos.slice().sort((a,b) => (a.createdDateTime||'').localeCompare(b.createdDateTime||''));
const primeiro = ordenados[0];
return [{ json: { file_id: primeiro.id, file_name: primeiro.name, ... } }];
```

### Bug 2 — Document AI troca comprador ↔ vendedor no cabeçalho
**Sintoma:** faturas FSM (fatura 385126/2705) chegavam ao BC com `nome_fornecedor = "JORGE BRANDÃO GONÇALVES UNIPESSOAL, LDA"` (comprador Cabeceiras). O NIF salvou o dia (505205440 = F00041 correto), mas o nome saía trocado.

**Causa:** Document AI ocasionalmente confunde `supplier_name` com `receiver_name` em PDFs onde o cabeçalho tem layout ambíguo.

**Solução:** `Avaliar Resultado BC` passou a sobrescrever `dados_fatura.nome_fornecedor` com o nome do vendor do BC (obtido via lookup do NIF). Original preservado em `nome_fornecedor_ocr` para auditoria.

```js
if (match && match.name) {
  dadosFatOverride.nome_fornecedor_ocr = dadosFatOverride.nome_fornecedor;
  dadosFatOverride.nome_fornecedor = String(match.name).trim();
}
```

### Bug 3 — Document AI erra qty/preço das linhas mas acerta o total
**Sintoma:** OCR extrai total correto (rodapé da fatura) mas nas linhas coloca quantidades e preços que não somam ao total. Ex: FSM total 246,58€ mas 3 linhas somaram 351,90€ (+105,32€).

**Descoberto por Robson:** o Excel Log_Faturas tinha o valor CERTO (246,58€) — sistema devia usar isso como âncora.

**Solução (final):** `Verificar EC Criado` deteta divergência (>0,05€) e **substitui todas as linhas OCR** por 1 única linha placeholder:
- `number: 'ME003007'`
- `quantity: 1`
- `directUnitCost: total_com_iva / (1 + IVA/100)`
- `description: 'AJUSTE - fatura com divergencia OCR - rever manualmente'`
- `taxa_iva: 23` (ou maior IVA das linhas)

EC entra no BC com **total correto**. Jorge desmembra as linhas ao abrir a EC.

`postingDescription` fica `AUTOMAÇÃO - ATENÇÃO VALORES - ITEM SEM MATCH` para sinalizar.

### Bug 4 — Log_Faturas Excel spamava linhas Anomalia
**Sintoma:** cada poll criava nova linha "Anomalia" no Log — 2, 3, 4 linhas idênticas da mesma fatura.

**Solução:** `Preparar Log Erro` procura linha existente Anomalia (NIF+número). Se encontrar → devolve `existing_row_index` + marca `is_update:true`. Novo IF `Log Existe?` roteia:
- true → novo HTTP `Log Erro Update` (PATCH `/rows/itemAt(index=X)`)
- false → `Log Erro` (POST add)

### Bug 5 — Anomalia "Divergencia totais" em `Parse e Validar` bloqueava tudo
**Sintoma:** mesmo após implementar ajuste em `Verificar EC Criado`, TOPBOIS FT 2026/1445 continuava presa. Investigação: `Parse e Validar` empurrava anomalia "Divergencia totais: linhas=X vs total_sem_iva=Y" ANTES do IF `Extração OK?`, que ratificava para `Preparar Log Erro`.

**Solução:** comentei o push da anomalia em `Parse e Validar`. Divergência já não é bloqueadora — o ajuste em `Verificar EC Criado` trata.

```js
// if (!totaisOk && dados.total_sem_iva > 0) {
//   anomalias.push(`Divergencia totais: linhas=... vs total_sem_iva=...`);
// }
```

---

## 3. Timeline de deploys (hora local Portugal WEST = UTC+1)

| # | Local | O que | Ficheiro/Nó | Resultado |
|---|-------|-------|-------------|-----------|
| 1 | 16:49 | Opção A: `Filtrar Faturas` retorna só 1 ficheiro/poll | Filtrar Faturas | ✅ resolve bug Loop |
| 2 | 17:16 | Novo IF `Divergência OK?` — bloqueia POST BC quando divergência | Preparar EC Header, novo IF | Blocking funcionou mas era demasiado restritivo |
| 3 | 17:34 | Deploy anti-spam Log — IF `Log Existe?` + `Log Erro Update` (PATCH) | Preparar Log Erro + 2 novos nós | ✅ Log deixa de spamar |
| 4 | 17:42 | Nome fornecedor override do BC | Avaliar Resultado BC | ✅ nome correto no Log/EC |
| 5 | 17:54 | Remover bloqueio + `Verificar EC Criado` faz ajuste (1 placeholder) | Preparar EC Header, Verificar EC Criado | ✅ EC criada com total OCR |
| 6 | 18:16 | Remover anomalia "Divergencia totais" do Parse | Parse e Validar | ✅ TOPBOIS 1445+ processam |

**6 deploys, todos incrementais e testados.**

---

## 4. ECs criadas hoje em PROD

| EC | Fornecedor (BC) | Nº fatura | Total c/IVA | Nº linhas BC | postingDescription | Notas |
|----|-----------------|-----------|-------------|-------------|-------------------|-------|
| **EC2602251** | F00896 Topbois, Lda | FT 2026/1414 | 1.213,27€ | 1× ME003007 @ 986,40€ | AUTOMAÇÃO - ITEM SEM MATCH | Sem catálogo TOPBOIS no BC — placeholder esperado. Valores OCR corretos, sem divergência. |
| **EC2602254** | F00041 Francisco Sousa Magalhães, Lda. | 385126/2705 | 246,58€ | 1× ME003007 @ 200,47€ (ajuste) | AUTOMAÇÃO - ATENÇÃO VALORES - ITEM SEM MATCH | OCR das linhas errou (351,90€) → sistema ajustou para total OCR 246,58€ |
| **EC2602255** | F00896 Topbois, Lda | FT 2026/1445 | 3.447,11€ | 1× ME003007 @ 2.802,53€ (ajuste) | AUTOMAÇÃO - ATENÇÃO VALORES - ITEM SEM MATCH | OCR linhas 2.547,55€ ≠ total 2.802,53€ → ajuste |
| **EC2602256** | F00896 Topbois, Lda | FT 2026/1461 | 2.362,07€ | 1× ME003007 @ 1.920,38€ (ajuste) | AUTOMAÇÃO - ATENÇÃO VALORES - ITEM SEM MATCH | idem |
| **EC2602257** | F00896 Topbois, Lda | 2026/1480 | 1.834,85€ | 1× ME003007 @ 1.491,75€ (ajuste) | AUTOMAÇÃO - ATENÇÃO VALORES - ITEM SEM MATCH | idem |

**Também apagadas durante testes:**
- EC2602249 (FSM valores errados — deploy antigo, apagada por Robson)
- EC2602250 (TOPBOIS 1414 vazia — apagada)
- EC2602252 (FSM linhas erradas — apagada, era caso de teste do bug 3 antes do ajuste)

---

## 5. Estado final (18:30 local)

**Workflow n8n `62wyOKnNBy0bnJUw`:**
- 63 nodes
- `active: True` (Robson não pediu para desativar hoje)
- Trigger 3min

**Pasta OneDrive `faturas por inserir`:** vazia ✅

**Log Excel `Log_Faturas` (5 linhas):**
1. EC2602251 — FT 2026/1414 — 1.213,27€ — Inserido BC
2. EC2602254 — 385126/2705 — 246,58€ — Inserido BC
3. EC2602255 — FT 2026/1445 — 3.447,11€ — Inserido BC
4. EC2602256 — FT 2026/1461 — 2.362,07€ — Inserido BC
5. EC2602257 — 2026/1480 — 1.834,85€ — Inserido BC

**BC PROD:**
- 5 ECs com yourReference=AUTOMAÇÃO (as acima)
- Fornecedores: TOPBOIS (4), FSM (1)
- Todos com CCusto vazio (nem TOPBOIS nem FSM têm registo no Excel Codigos Fornecedores)

---

## 6. Arquitetura do fluxo de linhas (final)

Para não perder o desenho:

```
Listar OneDrive → Filtrar Faturas (1 file/poll) → Tem PDFs?
  → Download PDF → Preparar Body DocAI → Document AI Extrair
  → Parse e Validar (SEM anomalia divergência)
  → Verificar Duplicado Local → É Duplicado Local?
  → Preparar Payload GPT Assinatura → GPT Verificar Assinatura → Merge Assinatura
  → Ler Log Excel → Obter Token BC → Verificar Duplicado BC
  → Ler Excel Codigos Fornecedores → Verificar Fornecedor BC → Avaliar Resultado BC (override nome_fornecedor)
  → Validar Receção → Verificar EC Existente BC → Avaliar EC Existente
  → Extração OK? (só bloqueia NIF inválido / sem nº fatura / sem linhas)
    ├── OK → Obter Token BC DEV → Aplicar Mapeamento Excel
    │   → Preparar EC Header (marca postingDescription mas NÃO bloqueia)
    │   → Criar EC Header BC PROD (POST)
    │   → Tem Linhas Sem Match?
    │      ├── sim → OpenAI Mapear Items → Aplicar Mapeamento → Adicionar Excel → Verificar EC Criado
    │      └── não → Verificar EC Criado
    │   → [Verificar EC Criado FAZ AJUSTE: se Σlinhas ≠ total, substitui por 1 ME003007 @ total OCR]
    │   → Loop Linhas EC → Criar Linha EC BC PROD
    │   → Atualizar Datas EC (postingDescription dinâmico)
    │   → Preparar Destino → Criar Pasta Ano → Criar Pasta Mês → Obter ID Pasta Mês
    │   → Mover PDF → Preparar Log → Log Sucesso
    └── Anomalia → Preparar Log Erro (com dedup) → Log Existe?
        ├── existe → Log Erro Update (PATCH)
        └── novo → Log Erro (POST)
        → Mover para Erro? → Mover PDF para Erro (após 3 tentativas) → Email
```

---

## 7. Regras críticas aprendidas hoje

1. **Excel Log_Faturas é fonte de verdade para totais.** OCR do rodapé é fiável; OCR das linhas é irregular. Se conflitam, confiar no total.

2. **1 alteração por deploy.** Cada bug isolado num deploy. 6 deploys hoje, cada um verificável isoladamente. Voltámos a validar a regra `um-deploy-uma-alteracao`.

3. **Não bloquear fluxo quando podemos ajustar.** Bloquear = trabalho manual para Jorge. Ajustar com placeholder + flag = sistema continua a produzir valor.

4. **Anomalias devem ser deduplicadas no Log.** Cada poll não deve criar linha nova para a mesma fatura pendente.

5. **Sobrescrever campos "não confiáveis" do OCR pelo BC** quando temos ID confiável (NIF). Nome, morada, etc.

---

## 8. Pendências para amanhã (2026-07-09)

### Prioritárias
1. **Cadastro Excel Codigos Fornecedores para TOPBOIS + FSM.** Sem catálogo, toda EC destes vai com `ME003007` e Jorge tem de desmembrar manualmente. Se Robson tiver access ao catálogo de items destes fornecedores, adicionar ao Excel para reduzir trabalho do Jorge.

2. **Modelo custom Document AI Studio treinado com layout TOPBOIS.** O `prebuilt-invoice` do Google erra sistematicamente as linhas TOPBOIS. Treinar com 10-20 faturas anotadas pode subir para ~99% precisão.

3. **Investigar variação Document AI entre execuções.** Mesma fatura → valores diferentes em polls diferentes (não-determinístico). Não crítico mas incomodativo em retries.

### Melhorias
4. Bug residual: prefixo duplicado `ECEC` em alguns nomes de ficheiro movido (raro).
5. `Preparar Log Erro` — considerar aumentar tentativas antes de mover para `faturas_com_erro` (agora é 3).
6. Adicionar campo `total_ocr` vs `total_bc` no Log para auditoria fácil.

### Alertas
7. **Workflow ficou ATIVO no fim da sessão** (18:30). Se Robson colocar mais PDFs à noite, sistema vai processar. Se não for esperado, desativar.

---

## 9. Ficheiros modificados

| Ficheiro | Alteração |
|----------|-----------|
| `workflow_fase1_ocr.json` (via n8n API) | Filtrar Faturas, Parse e Validar, Avaliar Resultado BC, Preparar EC Header, Verificar EC Criado, Preparar Log Erro (+ 3 novos nós: Divergência OK? removido do fluxo, Log Existe?, Log Erro Update) |
| `relatorio_2026-07-08.md` | este relatório |
| `MEMORY.md` (auto-memory) | possivelmente atualizar `project-faturas-fornecedores` com sessão de hoje |

---

## 10. Chaves e endpoints úteis

- **n8n URL:** `https://primary-production-0fe7d.up.railway.app`
- **n8n workflow:** `62wyOKnNBy0bnJUw` — 63 nodes, active
- **BC PROD OData:** `https://api.businesscentral.dynamics.com/v2.0/f41c8222-df66-449c-93b5-c1879e641cb2/PROD/ODataV4/Company('Jorge%20Brand%C3%A3o%20Gon%C3%A7alves')`
- **BC App ID:** `fabec729-bb7e-48f8-a3cc-4a649ed4ab45`
- **Graph App ID:** `b8e798a0-6988-47aa-a4a5-0d4d1fe125eb`
- **Log_Faturas Excel:** `01JSSR6ZOQEYD67CHDWZDYCTBQAOQPLZKA` / Tabela1 / Log_Faturas
- **Codigos Fornecedores Excel:** `01JSSR6ZOQ4BV4R2JA4JEJJD4BPSVMLY2K` / MapeamentoItems
- **Pasta faturas por inserir:** `01JSSR6ZOWI4JQ7ADEZFHZT24VNIWIU42R`
- **Pasta faturas inseridas:** `01JSSR6ZLDQWWQPDGYP5CZJAQFOZUTJGI2`
- **Pasta faturas_com_erro:** `01JSSR6ZMVOBS354FCNVDZG2CAXFQFJUR6`
- **Google Document AI:** projeto `cabeceiras-ocr` (id `164548594521`), processor `d9230429c35852ce`, region `eu`
- **OpenAI credencial n8n:** `CogELPgsbfrLHzNg` (usada para GPT-5 verificar assinatura + GPT-4o mapear items)
- **BC vendor cache Ontem:** F00001 Flex2000, F00041 FSM, F00896 TOPBOIS

---

## 11. Como testar amanhã

Para validar que tudo está OK após restart:

```bash
# 1) Verificar workflow ativo
curl -s -H "X-N8N-API-KEY: $N8N_KEY" \
  "$N8N_URL/api/v1/workflows/62wyOKnNBy0bnJUw" \
  | python3 -c "import sys,json;d=json.load(sys.stdin);print('active:',d.get('active'),'nodes:',len(d['nodes']))"

# 2) Colocar 1 PDF na pasta OneDrive faturas por inserir
# 3) Esperar até 3 min → verificar exec recente
curl -s -H "X-N8N-API-KEY: $N8N_KEY" \
  "$N8N_URL/api/v1/executions?workflowId=62wyOKnNBy0bnJUw&limit=1" | python3 -c "
import sys,json
d=json.load(sys.stdin)
e=d['data'][0]
print(f'{e[\"id\"]} {e[\"status\"]} start={e[\"startedAt\"]}')"

# 4) Confirmar EC criada + Log Excel populado + PDF movido
```

---

## 12. Resumo executivo (para amanhã de manhã)

> **Sistema funciona em produção.** 5 faturas processadas hoje com sucesso, todas com valor certo no BC. Faturas com OCR de linhas irregular (TOPBOIS, FSM sem catálogo) ficam com 1 linha placeholder `ME003007` no valor total OCR — Jorge desmembra manualmente ao abrir cada EC no BC. Nome do fornecedor vem sempre do BC (nunca do OCR, evita trocas comprador↔vendedor). Log Excel é deduplicado. Não bloqueia — sempre entrega valor a Jorge.
>
> **Falta principal:** cadastrar items de TOPBOIS e FSM no Excel Codigos Fornecedores para reduzir placeholders `ME003007`. Sem isso, todas as EC destes fornecedores caem sempre no ajuste.

---

## 13. Bug 6 (final) + Teste de regressão

### Bug 6 — Duplicado por prefixo diferente (FT vs FAT)
**Descoberto:** ao auditar as ECs no BC, apanhei 2 ECs para a mesma fatura FSM:
- EC2602232 (`FAT 385126/2705`) — pré-existente
- EC2602254 (`FT 385126/2705`) — criada hoje

**Causa:** `Verificar Duplicado BC` e `Verificar EC Existente BC` só removiam `FT` e `NC` antes de comparar. `Preparar EC Header` força prefixo `FT ` quando OCR vem sem prefixo reconhecido — mas EC2602232 tinha `FAT ` (FSM). Comparação `385126/2705 (após tirar FT)` vs `FAT 385126/2705` → diferentes → não apanhou duplicado.

**Fix (18:36 local):** alarguei o regex nos 2 nós dedup para todos os prefixos reconhecidos:
```
.replace(/^(FT|FAT|NC|FR|FRA|FC|FS|FE|GT|GS|GR|VD)\s*/i, '')
```

Mesmo padrão que `Preparar EC Header` já usa para detectar prefixos existentes.

**EC2602254 apagada** do BC + Log Excel (era duplicado, não era EC nova legítima).

### Teste de regressão (18:39 local — EC2602258)

Robson pediu para colocar mais uma fatura na pasta para validar que o sistema continua a funcionar após os fixes. Colocou 1 PDF TOPBOIS.

**Execução 69133 (18:39 local, 20,4s):**

| Campo | Valor | Status |
|---|---|---|
| File | Digitalização 10_23_17 (TOPBOIS FT 2026/1505) | |
| NIF | 510501885 | ✅ |
| Fornecedor BC | F00896 Topbois, Lda | ✅ (do BC, não OCR) |
| Total OCR | 2.750,22€ | |
| Linhas OCR | 6 | |
| Σ linhas OCR | 2.750,22€ | ✅ **BATE com total** |
| Ajuste divergência | NÃO aplicado (sem divergência) | ✅ funciona certo |
| Verificar Duplicado BC | 0 encontrados | ✅ regex novo OK |
| Verificar EC Existente BC | 0 encontrados | ✅ regex novo OK |
| EC criada | EC2602258 | ✅ |
| postingDescription | `AUTOMAÇÃO - ITEM SEM MATCH` | ✅ (sem ATENÇÃO VALORES porque não há divergência) |
| Pasta OneDrive | 0 ficheiros | ✅ PDF movido |

**6 linhas preservadas no BC (Σ = 2.750,22€):**

| # | Descrição | Qty | P.Unit | Total c/IVA |
|---|---|---|---|---|
| 1 | AGLOMERITE CARVALHO HERA ATLAS 285x210x19 | 5 | 46,20€ | 284,13€ |
| 2 | MDF HIDROFUGO 280x207x19 (104) | 5 | 40,75€ | 250,61€ |
| 3 | AGL. FOLH. FAIA ROSADA A/B 275x183x19 | 5 | 47,50€ | 292,13€ |
| 4 | AGLOMERITE CINZA HID TF 280x207x16 | 30 | 35,35€ | 1.304,41€ |
| 5 | AGLOMERITE CINZA HID. TF 280x207x19 | 4 | 38,40€ | 188,93€ |
| 6 | MDF. FOLH. CARVALHO A/B 275x185x19 | 4 | 87,40€ | 430,01€ |

Todas com `number=ME003007` (TOPBOIS sem catálogo BC) mas **qty/preços/descrições reais** preservados. Jorge só substitui o código do item ao abrir a EC — não precisa reintroduzir qty nem preço.

**Confirmação do desenho final:**
- Se OCR das linhas soma o total → mantém todas as linhas OCR (Jorge só substitui códigos ME003007)
- Se OCR das linhas ≠ total → substitui por 1 placeholder no valor total (Jorge desmembra tudo)

### Estado final da sessão (18:44 local)

**BC AUTOMAÇÃO — 7 ECs (0 duplicados):**
- EC2602209 Flex2000 FT 31/206437
- EC2602232 FSM FAT 385126/2705 (pré-existente)
- EC2602251 Topbois FT 2026/1414 — 1 linha, valor OCR OK
- EC2602255 Topbois FT 2026/1445 — 1 linha ajuste
- EC2602256 Topbois FT 2026/1461 — 1 linha ajuste
- EC2602257 Topbois FT 2026/1480 — 1 linha ajuste
- **EC2602258 Topbois FT 2026/1505 — 6 linhas OCR mantidas**

**Log Excel:** 5 linhas Inserido BC (todas de hoje)
**Pasta OneDrive:** vazia
**Workflow:** ATIVO (63 nodes)

**7 deploys hoje** (não 6 como reportado antes): deploy 7 = fix regex prefixos dedup.
