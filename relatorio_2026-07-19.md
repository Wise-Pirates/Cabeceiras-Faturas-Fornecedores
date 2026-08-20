# Relatório — Projeto 21 — Domingo 2026-07-19

**Cliente:** Cabeceiras.pt (Jorge Brandão)
**Foco:** Teste isolado da Elastron para validar match por nome no Excel + EC2602446 criada

---

## 1. Contexto

Domingo — mini-sessão focada em resolver bug da Elastron (Doc AI extraiu NIF do cliente Jorge por engano; fatura tinha nomes "Paris/London" cadastrados por Jorge no Excel mas sistema não achava por PK).

---

## 2. Descoberta chave — Jorge cadastra por 2 métodos no mesmo Excel

Ao investigar Elastron F00052 no Excel `Codigos Fornecedores`, achei 19 linhas com **2 formatos misturados**:

**Método 1 — PK exacto:**
```
E0437069001  → MP000048
E0413059101  → MP000123
5376-10-340MY → MP000246
```

**Método 2 — Nome de família (novo padrão do Jorge):**
```
Paris    → MP000480
London   → MP000480
Portland → MP000479
Brooklyn → MP000480
Lotus    → MP000480
Babel    → MP000480
```

Sistema atual só fazia match por PK exacto → todas as linhas Elastron da fatura ficavam `ME003007` porque Doc AI extrai PKs numéricos (E035..., E094..., etc) que não batem com "Paris"/"London".

---

## 3. Fix — match por nome contido na descrição

`Aplicar Mapeamento Excel`:

1. **`normPK` melhorada** — remove tudo excepto alfanumérico
2. **`mapaExcel` deduplicado** — se múltiplas linhas com mesmo PK, pega a que tem Cod ERP preenchido
3. **`mapaNomesPorVendor` (novo)** — para linhas Excel onde "PK" tem letras e ≥3 chars (ex: "Paris"), guarda para match por nome
4. **Match por nome** — se PK exacto não bate, procura `descricao.toLowerCase().includes(pk_lower)`. Preferir match mais longo (mais específico)

Ordem final: PK exacto → nome contido → placeholder ME003007 → ERP fallback

---

## 4. Também melhorado — Claude extrai NIF quando Doc AI falha

Adicionado ao prompt Claude Linhas: extrair também `nif_fornecedor` do PDF (procura em toda a fatura, incluindo rodapé).

Merge Linhas GPT-5 propaga NIF Claude para `dados_fatura` quando Doc AI extraiu vazio ou inválido.

Verificar Fornecedor BC + Avaliar Resultado BC lêem NIF do Merge.

---

## 5. Teste isolado — EC2602446 Elastron

**Fatura:** `Digitalização de 2026-07-09 02_34_21 PM.pdf` (reutilizada, já processada anteriormente)

**Resultado:**
- Vendor: F00052 Elastron (via nome único distintivo)
- Nº Fatura: FT CABECEIRAS (Doc AI ainda extraiu errado; problema resolvido no dia seguinte)
- purchaserCode: JB ✅

**Linhas — 18 total:**
- 6 via Excel PK exacto (MP000048 LONDON BEIGE, MP000123 LIVERPOOL, MP000538 BROOKLYN PEARL, MP000133 MUNICH x2, MP000246 BABEL)
- 6 via Excel nome (MP000480 PARIS, LONDON, LOTUS x2, BROOKLYN; MP000479 PORTLAND x2)
- 3 duplicados de linhas idênticas (mapaDescricao)
- 3 placeholder ME003007 (Tubo de Cartão x2 + Outros Serv. — não cadastrados no Excel)

**Antes deste fix:** 6 excel + 12 ME003007
**Depois:** 12 excel + 3 duplicados + 3 ME003007

Reduziu 12→3 placeholders. Só ficam legítimos (embalagem + genérico).

---

## 6. Estado final

- Workflow: DESATIVADO (aguarda validação Jorge)
- EC2602446 criada com resultado esperado — Jorge só precisa aprovar
- Fix match por nome no Excel funcional
- Fix Claude extrai NIF parcial (Elastron ainda extrai "CABECEIRAS" como nº fatura — resolvido no dia seguinte)

---

## 7. Nota

Elastron EC2602446 acabou por ser apagada no 20/07 pelo Jorge (queria reprocessar com fixes mais recentes).
