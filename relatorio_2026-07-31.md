# Relatório — Projeto 21 — Sexta 2026-07-31

**Cliente:** Cabeceiras.pt (Jorge Brandão)
**Workflow n8n:** `62wyOKnNBy0bnJUw` (principal) + `v8yIHOn4wGYHKhq7` (webhook) + `VT15XtXEohrnEkRi` (renovação)
**Foco:** Email routing Fase 4.5, tracking de custo real por fatura, PL000001 confirmado

---

## 1. Contexto

Manhã: Jorge questionou porquê Anthropic gastou €18 em 24h e se era do projeto de faturas. Investigação mostrou que este projeto gasta apenas ~€3.31/dia — restantes ~€14/dia vêm do workflow `Cabeceiras_Orcamentos` que partilha a mesma credencial Anthropic.

Adicionalmente:
- Confirmámos que ME003007 já não é usado (substituição feita 27/07 permanece efectiva)
- Sistema não processou nada 31/07 até às 10h47 porque Jorge não digitalizou (não é bug)
- Uma fatura Francisco Sousa processada às 10h46 → EC2602662 sem intervenção humana

---

## 2. Email routing (Fase 4.5 do planeamento) — IMPLEMENTADO

### Motivação
`planeamento-projeto.md` secção 4.5 previa routing de emails por tipo de anomalia. Estava por fazer.

### Fix
Novo node `Determinar Destinatários` (Code) analisa `anomalias[]` e retorna `email_to`, `email_cc`, `email_subject`, `email_tipos`. Node `Email Anomalia` (Gmail) agora usa expressões dinâmicas.

### Regras
```
Duplicado                        → compras1@cabeceiras.pt  (CC: Jorge)
Sem guia/receção                 → compras2@cabeceiras.pt
Fornecedor desconhecido          → compras1@cabeceiras.pt
Discrepância preço/valores       → comprasdetecidos@cabeceiras.pt
Produto sem match (PL000001)     → compras1@cabeceiras.pt
Default (anomalia genérica)      → wp@cabeceiras.pt  (CC: Jorge)
```

Múltiplas anomalias na mesma fatura → agrega destinatários únicos, tipos aparecem no assunto.

---

## 3. Tracking de custo real por fatura — IMPLEMENTADO

### Motivação
Jorge queria saber quanto gasta o projeto em APIs. Precisamos ter valor real por processamento no workflow.

### Custos por stack (medido em exec real 85872)
| Stack | Calls/fatura | Custo (fatura 1 pág) |
|-------|-------------|----------------------|
| Google Doc AI | 1 | **$0.065** (~€0.06) |
| Claude Sonnet 4.5 | 1 | **$0.013** (~€0.012) |
| OpenAI GPT (assinatura) | 1 | ~$0.001 |
| BC OData | 10 | $0 (plano BC Essentials) |
| MS Graph OneDrive | 9 | $0 (plano M365) |
| **TOTAL variável** | | **~$0.08 ≈ €0.07** |

### Fix
1. Adicionadas 4 colunas ao log Excel `Tabela1`:
   - `custo_total_eur`
   - `custo_docai_eur`
   - `custo_claude_eur`
   - `custo_gpt_eur`
2. Node `Preparar Log` reescrito — calcula custo real usando `usage.input_tokens`/`output_tokens` de Claude e GPT + $0.065 fixo Doc AI, converte USD→EUR (0.92).
3. Cada fatura processada regista custo real no log — permite Jorge filtrar/somar por dia, mês, fornecedor.

### Diferenças custo:
- **Doc AI** escala com **páginas** ($0.065/pág)
- **Claude** escala com **tokens** (fatura 1 linha ~€0.008; fatura 14 linhas ~€0.05)
- **GPT** escala com tokens (leve, ~$0.001/call)

### Projecção
46 faturas/dia × €0.07 = **~€3.31/dia** (fica claro que Anthropic dashboard mostra outros workflows)

---

## 4. ECs criadas ontem (30/07) — 46 ECs

Sistema processou 46 faturas entre 15h e 22h30 sem intervenção. Todas via webhook OneDrive (trigger imediato). Fornecedores: Francisco Sousa Magalhães, Cadeiras Pinto, Ernesto Romano, EUROPEAN SKY, Lucia Oliveira, LOJA DAS FERRAGENS PAULO, HELDER ROBERTO, ALBERTO SANTOS, Helio Martins & Santos, Ferragens Carreiras, Madimorais, Flex2000, etc.

Range: EC2602616 → EC2602661.

---

## 5. ECs criadas hoje (31/07)

Até agora: **1 EC** — EC2602662 Francisco Sousa (FAT 385126/3439, 440.59€) às 10h46 UTC (11h47 Portugal). Latência do PDF na pasta até EC no BC: ~27 segundos via webhook.

---

## 6. Estado final

- **Workflow principal**: ATIVO (fallback cron 1x/hora seg-sab 08-19)
- **Webhook**: ATIVO (trigger imediato via Graph subscription até 25/08)
- **Renovação subscription**: ATIVO (segundas 07h)
- **PL000001**: 15 ocorrências no workflow (substituiu ME003007 desde 27/07)
- **ME003007**: 0 ocorrências no workflow, item permanece no BC mas não é usado
- **Email routing Fase 4.5**: IMPLEMENTADO
- **Custo tracking**: IMPLEMENTADO

---

## 7. Fase 4.5 do planeamento — cobertura

| Item | Estado |
|------|--------|
| Templates HTML anomalia | ✅ (existente reforçado com tipos) |
| Templates HTML sucesso | ❌ (Jorge indicou que não quer, log Excel basta) |
| Templates HTML atraso EV | ❌ (Fase 4 não crítica) |
| Routing por tipo → destinatários | ✅ (feito hoje) |
| Multi-anomalia agrega destinos | ✅ |
| CC Jorge em duplicados | ✅ |

---

## 8. Fase 5 do planeamento — cobertura

| Item | Estado |
|------|--------|
| Log Excel auditoria | ✅ (existente) |
| Log Excel custos por stack | ✅ (feito hoje) |
| Feedback aprendizagem produtos | 🟡 (Jorge cadastra manualmente Excel Codigos Fornecedores) |

---

## 9. Pendências

- Confirmar com Jorge se emails `compras1@`, `compras2@`, `comprasdetecidos@` existem realmente (se não, ajustar routing)
- Investigar workflow `Cabeceiras_Orcamentos` (mesmo credencial Anthropic) para explicar os restantes ~€14/dia
- Cadastrar produtos ME/PL para Topbois, Francisco Sousa Magalhães (reduzir placeholders)
