# Relatório — Projeto 21 — Quinta 2026-07-23

**Cliente:** Cabeceiras.pt (Jorge Brandão)
**Workflow n8n:** `62wyOKnNBy0bnJUw`
**Foco:** Bug reincidente ME003007 blocked, fixes header (número/data), reprocessamento 35 ECs

---

## 1. Contexto matinal

Sessão começou 09h com pasta cheia (26 PDFs do lote 22/07 do Jorge). Objectivo: processar e resolver os erros que ficaram do dia anterior.

Ao ativar workflow, todas as tentativas ficavam ~19-25s (curtas) — indício de falha antes de criar EC.

---

## 2. Bug bloqueio silencioso — ME003007 re-bloqueado

### Sintoma
Doc AI + Claude extraiam bem, Merge produzia bem, mas EC no BC ficava com **0 linhas**. Investigação mostrou:
```
Blocked must be equal to 'No' in Item: No.=ME003007. Current value is 'Yes'
```

Item `ME003007` (placeholder "Mercadoria P.P a alterar") tinha sido re-bloqueado no BC entre 22/07 e 23/07.

### Fix
Desbloqueado via API v2.0 (`PATCH /items({id})` com `{"blocked": false}`).

**Nota:** este bug já tinha sido resolvido 22/07. Alguém no BC voltou a bloquear. Precisa ser monitorizado.

---

## 3. Bug data — Doc AI extrai "20/07/2.026" (ponto milhar no ano)

### Sintoma
BC rejeitava PATCH orderDate com erro:
```
Cannot convert the literal '20/07/2.026' to the expected type 'Edm.Date'
```

Doc AI extraiu ano com formato milhar (`2.026`), Parse não tratava.

### Fix
`Parse e Validar` → `normalizeDate` — limpa ponto entre dígitos únicos e 3 dígitos antes do regex:
```js
const str = String(s).trim().replace(/(\d)\.(\d{3})(?!\d)/g, '$1$2');
```

---

## 4. Bug match vendor por nome — Nogueira & Ribeiro

### Sintoma
Fatura da "ACO Nogueira & Ribeiro" NIF 502884138 travou em loop (15+ execuções falhadas mesmo PDF). Doc AI extraiu NIF errado por 2 dígitos (real 502684135 = F00771 no BC). Fallback por nome não achou porque `&` não estava no split e primeira palavra ficou "Nogueira&Ribeiro" (não bate "NOGUEIRA E RIBEIRO LDA").

### Fix
`Avaliar Resultado BC` → `findByName`:
1. Split também por `&`
2. Refinar com tokens seguintes: se primeira palavra dá múltiplos vendors, filtrar por 2º token (`RIBEIRO`)

Resultado: NOGUEIRA (4 vendors no BC) → filtra RIBEIRO → único F00771 ✅

---

## 5. Bug número fatura — Meilex "FT 2743"

### Sintoma
Meilex EC2602521 ficou com `Nº Fatura = FT 2743` (era Cliente Nº). Real: `FT FA.2026/4202`.

Doc AI leu "2743" (código do cliente Cabeceiras na Meilex) como invoice_id. Claude leu correcto "FA.2026/4202" mas Merge preferia Doc AI se >3 chars.

### Fix inicial
`Merge Linhas GPT-5` — regra `nfDocBad` rejeita números puramente dígitos ≤6 chars:
```js
const nfDocBad = !nfDoc || nfDoc.toLowerCase()==='cabeceiras' || nfDoc.length < 3 || /^\d{1,6}$/.test(nfDoc);
```

Depois descoberto que fix não bastava — ver relatório 24/07 (`pegarValidadoGPT5` também precisava propagar Merge).

---

## 6. ECs criadas hoje (23/07)

**35 ECs via AUTOMAÇÃO — EC2602514 → EC2602548:**

| Vendor | Nº ECs | Notas |
|--------|--------|-------|
| Topbois, Lda | 10 | ~26.700€ — todas ME003007, precisa cadastrar Excel |
| Francisco Sousa Magalhães | 11 | ~5.500€ |
| Paulo Oliveira & Ribeiro (marpa) | 2 | |
| Meilex | 2 | 1 com erro Nº Fatura (resolvido dia 24) |
| TEXBOX | 2 | 1 com erro ATCUD (resolvido dia 24) |
| Nogueira & Ribeiro | 1 | Fix nome funcionou |
| Outros | 7 | Ferragens, Ernesto Romano, Vanja, etc. |

**Estado das flags:**
- 11 completamente limpas
- 24 com "ITEM SEM MATCH" (produto ME003007 — Jorge cadastra Excel)
- 1 com "ATENÇÃO VALORES" (Nogueira, header divergência)
- 1 com "VERIFICAR FORNECEDOR" (Nogueira, match por nome)

---

## 7. Correcções finais do dia

- **EC2602521 Meilex** — Nº Fatura errado `FT 2743` → apagada, PDF devolvido, aguarda fix mais profundo no dia seguinte

---

## 8. Deploys da sessão

- Parse e Validar: `normalizeDate` limpa `2.026` → `2026`
- Avaliar Resultado BC: `findByName` com split & + refino tokens
- Merge Linhas GPT-5: `nfDocBad` rejeita numero puro ≤6 dígitos
- BC: ME003007 desbloqueado (2ª vez em 2 dias)

---

## 9. Estado final

- Workflow: **ATIVO** (cron 3min seg-sex 07-18)
- Timezone: Europe/Lisbon
- ME003007: unblocked
- ECs entregues: 35 (dia)
- 1 EC pendente reprocessamento (Meilex → dia 24)

---

## 10. Lições

- **ME003007 pode ser re-bloqueado por outros utilizadores do BC** — necessário check periódico ou trigger de erro-then-fix
- Doc AI erra frequentemente em: ATCUD, código cliente, due_date (em vez de invoice_date), formatos numéricos regionais
- Claude é fonte mais fiável para header fields (num_fatura, data_fatura, nif_fornecedor)
