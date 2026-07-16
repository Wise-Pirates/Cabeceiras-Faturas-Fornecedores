# Relatório — Projeto 21 — Sessão 2026-07-03

**Cliente:** Cabeceiras.pt (Jorge Brandão Gonçalves)
**Sessão:** manhã (redesign Excel-first) + tarde/noite (validação PROD)
**Workflow n8n:** `62wyOKnNBy0bnJUw` — 56 nodes — **ATIVO** em produção
**Estado final:** Sistema end-to-end validado em **BC PROD** com 2 faturas Flex2000 processadas com sucesso.

---

## 1. Feedback do Jorge (manhã)

Após validação da **EC2602160** (Flex2000 FT 31/206437), Jorge confirmou:
> "EC2602160 já funcionou... falta agora adicionar o centro de custo e os descontos correctos no sitio certo"

Foco desta sessão: **centro de custo** (implementado) + **descontos** (adiado para segunda com Jorge).

---

## 2. Migração crítica Claude → OpenAI GPT-4o

**Motivo:** Cliente não tem conta Anthropic. Estava a usar credencial pessoal do Robson (incidente evitado).

**Antes desta sessão:** Ficheiro local `/tmp/live_wf.json` já tinha nomes OpenAI, MAS **nunca foi feito push para o n8n**. O workflow em produção continuava a chamar `api.anthropic.com` e a usar credencial `srXSApQJ2OBjvnxL` (Anthropic Robson).

**Descoberta acidental:** Ao processar a fatura Flex2000 FT 31/206437 (execução 67719), o EC2602178 foi criado com a credencial Anthropic pessoal. Foi apagado imediatamente.

**Aplicado:** 9 nós migrados definitivamente:
- `Preparar Payload Claude/Guia/Items` → OpenAI format (`type: "file"` + `file_data` base64)
- `Claude: Extrair Fatura` → `OpenAI: Extrair Fatura`
- `Claude Verificar Assinatura` → `OpenAI Verificar Assinatura`
- `Claude Mapear Items` → `OpenAI Mapear Items`
- `Parse e Validar`, `Consolidar Prova Receção`, `Aplicar Mapeamento` → parser adaptado para `choices[0].message.content`
- Credencial: `CogELPgsbfrLHzNg` (`OpenAi Cabeceiras`)
- Modelo: `gpt-4o`

**Verificação final:** `0 nós com credential Anthropic` no workflow.

---

## 3. Feature — Centro de Custo (shortcutDimension1Code)

**Análise BC:**
- Dimensão `CCUSTO` = "Centro Custo" (Global Dimension 1 no BC)
- Valor `17` = "Fábrica" (usado para Flex2000)
- Confirmado em EC recente do Jorge: `TFCS26070044` (Flex2000) usa `shortcutDimension1Code='17'`

**Implementação (hardcoded `17`):**
- `Preparar EC Header`: `shortcutDimension1Code: '17'` no POST
- `Verificar EC Criado`: `shortcutDimension1Code: '17'` em cada linha
- `Atualizar Datas EC` (PATCH): `shortcutDimension1Code: '17'` como fallback (BC ignora dimensão no POST)

**Nota:** Ficou pendente para segunda com Jorge — decidir se manter fixo `17` ou variar por fornecedor via coluna Excel.

---

## 4. Sequência de bugs corrigidos (7 fixes)

### Fix 1 — Aplicar Mapeamento Excel (crítico)
- **Bug:** Match Excel usava `nif_fornecedor` ("504663232") mas Excel guarda como `bc_vendor_no` ("F00001") → nunca matcha
- **Impacto:** Todos os itens iam para placeholder ME003007
- **Fix:** Trocar `nifFornecedor` por `bcVendorNo = validado.bc_vendor_no`

### Fix 2 — Verificar EC Criado (crítico)
- **Bug:** Lia sempre de `Aplicar Mapeamento Excel` (só matches Excel), ignorando `Aplicar Mapeamento` (fallback OpenAI). Ou crashava quando fallback não corria (Excel 100%).
- **Fix:** `try { data = $('Aplicar Mapeamento').first().json } catch { data = $('Aplicar Mapeamento Excel').first().json }` — robusto para Excel 100% ou com fallback

### Fix 3 — Mover PDF + Criar Pasta Mês + Obter ID Pasta Mês
- **Bug:** Usavam `.item.json` que quebra após loops (`Paired item data unavailable`)
- **Fix:** Trocar para `.first().json`

### Fix 4 — Atualizar Datas EC (PATCH)
- **Bug:** BC sobrescreve `postingDescription='Order EC...'` e `yourReference=''` no POST
- **Fix:** PATCH agora envia `postingDescription: 'AUTOMAÇÃO'`, `yourReference: 'AUTOMAÇÃO'`, `shortcutDimension1Code: '17'` (junto com datas)

### Fix 5 — Prompt OpenAI (crítico)
- **Bug:** GPT-4o gerava JSON inválido com **expressões matemáticas**: `"total_linha_com_iva": 234.20 * 1.23`
- **Impacto:** `Parse e Validar` falhava, fatura ia para anomalia
- **Fix:** Regra explícita no prompt: "todos os valores numericos devem ser NUMEROS LITERAIS, NUNCA expressoes matematicas" + regra vírgula PT → ponto JSON

### Fix 6 — Verificar Duplicado BC (falso positivo)
- **Bug:** Filtrava só por `External_Document_No` → apanhava fatura "FT 31/206437" do DAVIS FABRICS SPAIN (PF26060326) quando estava a processar Flex2000
- **Tentativa 1 (falhou):** Adicionar filtro `Vendor_No` no URL — mas `Verificar Fornecedor BC` corre DEPOIS de `Verificar Duplicado BC`
- **Fix final:** Manter URL original; filtrar `entries` por `Vendor_No === vendor_no` **em memória** no `Avaliar Resultado BC` (que já tem ambos os inputs)

### Fix 7 — Prefixo FT (pedido pelo Jorge)
- **Bug:** Fatura sem prefixo no PDF (`31/206787`) → vendorInvoiceNumber ficou sem `FT `
- **Fix:** Se OCR extrai sem prefixo alfanumérico (FT/NC/FR/GT/GS/GR/VD), força `FT ` no início

---

## 5. Testes end-to-end (BC PROD)

### Teste 1 — EC2602181 (fatura FT 31/206437, 11 Jun 2026)

**Execução 67727** às 19:15:50 UTC — **SUCCESS** (18,1s)

| Campo header | Valor |
|--------------|-------|
| number | EC2602181 |
| buyFromVendorNumber | F00001 (Flex2000) |
| vendorInvoiceNumber | `FT 31/206437` |
| postingDate | 2026-06-11 |
| shortcutDimension1Code | **17** ✅ |
| locationCode | P1 |
| postingDescription | **AUTOMAÇÃO** ✅ |
| yourReference | **AUTOMAÇÃO** ✅ |

**5 linhas — Excel match 5/5 direto (sem fallback OpenAI):**
| Linha | Item BC | Descrição | Qtd × Preço | dim1 |
|-------|---------|-----------|-------------|------|
| 10000 | ME003259 | KAPPITON HYBRID ECO 200X90X22 | 2 × 117,098 | 17 |
| 20000 | ME003260 | KAPPITON HYBRID ECO 200X140X22 | 10 × 156,41 | 17 |
| 30000 | ME003253 | KAPPITON HYBRID ECO 200X160X22 | 16 × 168,245 | 17 |
| 40000 | ME003252 | KAPPITON HYBRID ECO 200X180X22 | 4 × 193,742 | 17 |
| 50000 | ME003253 | KAPPITON HYBRID ECO 200X160X22 | 11 × 178,34 | 17 |

**Pós-processamento:** PDF movido para `/Cabeceiras-PARTILHA/.../faturas inseridas/2026/06/FT_31-206437.pdf`. Linha de log adicionada ao Excel `Log_Faturas`.

### Teste 2 — EC2602182 (fatura 31/206787, 23 Jun 2026)

**Execução 67730** às 19:30:50 UTC — **SUCCESS** (13,1s)

| Campo | Valor |
|-------|-------|
| number | EC2602182 |
| vendorInvoiceNumber | `31/206787` (sem FT — fix 7 aplicado 19:39 DEPOIS deste teste) |
| postingDate | 2026-06-23 |
| shortcutDimension1Code | **17** ✅ |
| postingDescription/yourRef | **AUTOMAÇÃO** ✅ |

**1 linha:**
- L10000: `ME003253` (PK-1079 KAPPITON HYBRID ECO 200X160X22), qty 14 × 178,34€, dim1=17

**Observação:** vendorInvoiceNumber ficou `31/206787` porque o PDF não tinha `FT ` visível. Deploy 19:39 do **Fix 7** garante que EC futuras terão sempre prefixo. Pergunta pendente ao user: fazer PATCH manual na EC2602182 para corrigir?

**Observação 2:** descrição da linha inclui o próprio código PK: `PK-1079 KAPPITON HYBRID ECO 200X160X22 14,000 UN 1...`. OpenAI extraiu a linha inteira do PDF em vez do nome do produto. Possível melhoria futura de prompt.

---

## 6. Estado dos objetos BC do dia

| EC | Estado | Notas |
|----|--------|-------|
| EC2602156 | Apagada | Teste 1 antes das correções |
| EC2602160 | Validada Jorge | "Já funcionou" — pediu centro custo + descontos |
| EC2602163 | Apagada | Teste NC (nota de crédito) |
| **EC2602178** | Apagada | Criada acidentalmente com credencial Anthropic Robson |
| **EC2602180** | Apagada | Ficou vazia (erro try/catch antes do fix) |
| **EC2602181** | ✅ Ativa | **PARA VALIDAR JORGE** — Flex2000 FT 31/206437, 5 linhas, CCusto 17 |
| **EC2602182** | ✅ Ativa | **PARA VALIDAR JORGE** — Flex2000 31/206787, 1 linha, CCusto 17 (falta FT no vendorInvoiceNumber) |

---

## 7. Perguntas pendentes para Jorge (segunda 2026-07-06)

Ver ficheiro dedicado: `PERGUNTAS-JORGE-SEGUNDA-2026-07-06.md`

1. **Centro de custo** — manter fixo `17` ou variar por fornecedor via Excel col.5?
2. **Descontos** — Flex2000 aplica ~2% (diff 49,94€ na EC2602181). Onde no BC? Por linha (`lineDiscountPercent/Amount`) ou global (`invoiceDiscountAmount`)?
3. **Descrição das linhas** — manter como OpenAI extrai (inclui código+qtd) ou forçar só o nome do produto?
4. **EC2602182** com `vendorInvoiceNumber='31/206787'` sem prefixo — corrigir manualmente com PATCH?
5. **Sincronizar Excel local** (4 colunas) com OneDrive (5 colunas com "Centro de Custo") — qual é a fonte da verdade?

---

## 8. Ficheiros modificados esta sessão

| Ficheiro | Alterações |
|----------|------------|
| `workflow_fase1_ocr.json` | 56 nodes — todos os 7 fixes acima + CCusto 17 + prefixo FT |
| `PERGUNTAS-JORGE-SEGUNDA-2026-07-06.md` | novo, tópicos para Jorge |
| `relatorio_2026-07-03.md` | este relatório (substituiu versão de manhã) |
| Memory `project-faturas-fornecedores` | atualizada com feedback Jorge + fixes + validação EC2602181/2182 |

---

## 9. Deploys realizados no n8n

| Timestamp UTC | O que foi | Resultado |
|---------------|-----------|-----------|
| 18:19 | +shortcutDimension1Code=17 (só) | error (Excel bug, sem OpenAI, .item bug) |
| 18:40 | Migração OpenAI + 5 fixes (Excel, Verificar EC, Mover PDF, PATCH desc/yourRef) | error (Aplicar Mapeamento não executado quando Excel 100%) |
| 18:47 | try/catch em Verificar EC Criado | error (JSON expressões matemáticas do GPT-4o) |
| 18:54 | Prompt OpenAI reforçado + filtro Vendor_No (URL) | error (Verificar Fornecedor BC ainda não corrido) |
| 19:13 | Filtro Vendor_No em memória (Avaliar Resultado BC) | **✅ EC2602181 SUCCESS 19:15** |
| — | (nova fatura user, sem deploy) | **✅ EC2602182 SUCCESS 19:30** |
| 19:39 | Fix 7: forçar prefixo FT quando ausente | **ATIVO** — aguardar próxima fatura sem FT no PDF |

---

## 10. Métricas finais

- **Faturas processadas com sucesso end-to-end:** 2/2 (100%)
- **Tempo médio por fatura:** 15,6s (13,1s + 18,1s)
- **Excel matching direto:** 6/6 items (5/5 EC2602181 + 1/1 EC2602182) — 100%, sem cair no fallback OpenAI
- **BC PROD ECs criadas para validação Jorge:** 2 ativas
- **Zero uso de credencial Anthropic** após 18:40 deploy
- **Custo estimado por fatura (OpenAI GPT-4o):** ~$0,02-0,05 (OCR PDF + eventual mapping items)

---

## 11. Próximos passos (segunda 2026-07-06)

1. Jorge validar EC2602181 e EC2602182 no BC PROD
2. Decisão sobre descontos (implementação depende da resposta)
3. Decisão sobre CCusto por fornecedor (Excel col.5)
4. Se OK, reativar processamento contínuo (workflow já ativo — polling 5 min)
5. Monitorar mais faturas de outros fornecedores (Emma, madimorais) para validar robustez
