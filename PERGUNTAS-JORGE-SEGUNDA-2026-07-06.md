# Perguntas para Jorge — Segunda-feira 2026-07-06

Contexto: Jorge validou EC2602160 no BC — "já funcionou". Feedback dele: "falta agora adicionar o centro de custo e os descontos correctos no sitio certo".

## 1. Centro de Custo — regra por fornecedor?

**Estado atual (deploy 2026-07-03 18:19):** hardcoded `shortcutDimension1Code = '17'` (Fábrica) em:
- Header EC (POST `workflowPurchaseDocuments`)
- Cada linha (POST `workflowPurchaseDocumentLines`)
- PATCH `Atualizar Datas EC` como fallback

**Referência:** EC recente do Jorge (`TFCS26070044` Flex2000) usa `shortcutDimension1Code = '17'`.

**A confirmar:**
- [ ] Manter fixo `17` ou variar por fornecedor?
- [ ] Se variar: onde definimos a regra?
  - Opção A: Coluna "Centro de Custo" no Excel `Codigos Fornecedores.xlsx` (Jorge já preencheu `17` para Flex2000 mas o Excel local só tem 4 colunas — está diferente entre local e OneDrive?)
  - Opção B: Campo no cadastro do fornecedor no BC (existe?)
  - Opção C: Regra por tipo de produto/família
- [ ] Confirmar que dimensão "CCUSTO" (17 = Fábrica) é sempre a Global Dimension 1 (`shortcutDimension1Code`) — vimos que sim no BC
- [ ] Alguma vez usam Global Dimension 2 (`shortcutDimension2Code`)?

## 2. Descontos — onde e como

**Descoberta OCR EC2602160 (Flex2000 FT 31/206787):**
- 1 linha: 14 × 178,34€ = **2496,76€** bruto
- Total sem IVA na fatura: **2446,82€**
- **Diferença = 49,94€ ≈ 2% desconto**

OpenAI GPT-4o extrai o `total_linha_sem_iva: 2496,76` (bruto) e `total_sem_iva: 2446,82` (líquido) — mas **não extrai o campo "desconto"** explicitamente.

**A confirmar:**
- [ ] Descontos aparecem por linha ou só no total da fatura Flex2000?
- [ ] % de desconto é fixo por fornecedor (contrato) ou varia por linha/produto?
- [ ] Onde Jorge quer no BC?
  - Campo linha: `lineDiscountPercent` ou `lineDiscountAmount`
  - Campo cabeçalho: `invoiceDiscountAmount` ou `paymentDiscountPercent`
- [ ] Pedir screenshot de fatura Flex2000 (ou Emma, madimorais) a mostrar como o desconto aparece
- [ ] Fatura Flex2000 tem desconto sempre 2%? Ou varia?

## 3. Migração OpenAI — validação de qualidade

Workflow migrado de Claude/Anthropic → OpenAI GPT-4o (2026-07-03), ainda **DESATIVADO**.
- Motivo: cliente não tem conta Anthropic, só tem `OpenAi Cabeceiras` (credencial `CogELPgsbfrLHzNg`)
- [ ] Testar 1 fatura piloto com o GPT-4o e comparar qualidade OCR vs Claude
- [ ] Reativar workflow se OK

## 4. Excel `Codigos Fornecedores` — sincronizar local vs OneDrive

Excel local só tem 4 colunas (`Numero | Nome | Codigo Fornecedor | Codigo ERP`), mas o do OneDrive supostamente tem 5 (Jorge adicionou "Centro de Custo" = 17).

- [ ] Confirmar qual é a "fonte da verdade" — OneDrive (Jorge edita) ou local?
- [ ] Se OneDrive, atualizar o local para refletir

---

## Notas de deploy 2026-07-03

Workflow ID: `62wyOKnNBy0bnJUw` — 56 nodes — DESATIVADO
Alterações desta sessão:
1. Migração Claude → OpenAI GPT-4o (9 nós modificados)
2. `shortcutDimension1Code = '17'` fixo em 3 pontos (header + linhas + PATCH)

Ficheiro: `workflow_fase1_ocr.json` (última alteração 2026-07-03 18:19).
