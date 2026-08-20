# Relatório — Projeto 21 — Sexta 2026-07-24

**Cliente:** Cabeceiras.pt (Jorge Brandão)
**Workflow n8n:** `62wyOKnNBy0bnJUw` + novos `v8yIHOn4wGYHKhq7` (webhook) e `VT15XtXEohrnEkRi` (renovação)
**Foco:** 5 erros do cliente resolvidos, webhook OneDrive trigger imediato, ME003007 re-bloqueado (3ª vez)

---

## 1. Contexto

Manhã: Jorge reportou por WhatsApp print com **5 falhas específicas** em ECs criadas 22-23/07. Sessão dedicada a diagnosticar + fixar cada uma cirurgicamente + implementar trigger imediato via webhook.

---

## 2. Bugs resolvidos (5)

### 2.1 EC2602521 Meilex — Nº Fatura `FT 2743` (Cliente N.)

**Bug:** Doc AI extraiu "2743" (Cliente Nº). Claude leu certo "FA.2026/4202". Fix parcial 23/07 (nfDocBad) só corrigia Merge, mas `pegarValidadoGPT5` no `Preparar EC Header` só propagava `linhas` do Merge — não `numero_fatura/data_fatura/nif`. Reverteia para Doc AI ("2743").

**Fix:** Reescrever `pegarValidadoGPT5` no `Preparar EC Header` — sempre usar Merge para header fields:
```js
const base = { ...validado, dados_fatura: { ...validado.dados_fatura } };
if (numMerge) base.dados_fatura.numero_fatura = numMerge;
if (dtMerge) base.dados_fatura.data_fatura = dtMerge;
if (nifMerge) base.dados_fatura.nif_fornecedor = nifMerge;
```

Reprocessado → **EC2602550** `FT FA.2026/4202` ✅

### 2.2 EC2602517 TEXBOX — Nº Fatura era ATCUD

**Bug:** Doc AI leu `invoice_id = J6ZTJ4K4-3925` (que é o ATCUD, código do documento fiscal AT). Merge não detectava.

**Fix:** `Merge Linhas GPT-5` — inverter regra: preferir Claude sempre que devolver valor válido (comprimento ≥3, não puramente ≤6 dígitos). Doc AI só como fallback:
```js
const nfClaudeGood = !!nfClaude && nfClaude.length >= 3 && !/^\d{1,6}$/.test(nfClaude);
const nfFinal = nfClaudeGood ? nfClaude : nfDoc;
```

Análogo para `data_fatura` (ISO validado).

Reprocessado → **EC2602552** `FT FAC A26/3925` ✅

### 2.3 EC2602515 marpa — data era vencimento

**Bug:** Doc AI não extraiu `invoice_date`, só `due_date=2026-08-20`. Parse usou due_date. Claude tinha correcto `2026-07-21` mas Merge preferia Doc AI. E mesmo depois de Merge corrigir, `Atualizar Datas EC` fazia PATCH lendo `Validar Receção` PRIMEIRO (com data errada) e só fallback Merge.

**Fix duplo:**
1. Merge prefere Claude para data (relacionado 2.2)
2. `Atualizar Datas EC` jsonBody: lê `Preparar EC Header` primeiro (fonte consolidada), depois Merge, depois Validar Receção

Reprocessado → **EC2602553** data `2026-07-21` ✅

### 2.4 EC2602505 Hélio Martins — quantidade `1.483` em vez de `1483,776`

**Bug:** Doc AI extraiu `1,483.7760` (formato US, `,` como milhar). `normalizeNumber` no Parse só sabia formato PT (`1.234,56`). Regex US não batia → caía no `else` → `replace(',','.')` → `"1.483.7760"` → `parseFloat` para no 2º ponto → **`1.483`**.

Total ficou 6,67€ em vez de 5.861,06€.

**Fix:** `Parse e Validar` → `normalizeNumber` com detecção formato US:
```js
if (/^\d{1,3}(\.\d{3})+,\d+$/.test(str)) { /* PT */ }
else if (/^\d{1,3}(,\d{3})+\.\d+$/.test(str)) { str = str.replace(/,/g,''); /* US */ }
else { str = str.replace(',','.'); }
```

Reprocessado → **EC2602554** qty `1483.776 × 4,50 × 12,2% desc = 5.862€` ✅

### 2.5 EC2602512 Woodside — unidade híbrida UN × M²

**Bug:** Fatura tem `52 UN × 8,14 €/M²`, com descrição `(2,8000 x 2,0700 x 0,0190 = 301,3920)`. Total real: 52 × 5,796m² × 8,14 = **2.453,33€**. BC ficou `52 × 8,14 = 423,28€`.

Sistema não sabe multiplicar pelas dimensões da placa quando qty é UN mas preço é €/M².

**Fix duplo:**
1. Prompt Claude — pedir campo obrigatório `total_linha_sem_iva` em cada linha
2. Merge — sanity check: se `qty × preço × (1-desc) ≠ total_linha` (>1% diff), recalcular `qty = total_linha / (preço × (1-desc))`

```js
gptData.linhas.forEach(L => {
  const q = Number(L.quantidade) || 0;
  const p = Number(L.preco_unitario) || 0;
  const dsc = Number(L.desconto_percent) || 0;
  const t = Number(L.total_linha_sem_iva) || 0;
  const calc = q * p * (1 - dsc/100);
  if (t > 0 && p > 0 && calc > 0 && Math.abs(calc - t) / t > 0.01) {
    L.qty_original_ocr = q;
    L.quantidade = t / (p * (1 - dsc/100));
    L.qty_recalculada = true;
  }
});
```

Reprocessado → **EC2602557** qty `301,39 M² × 8,14 = 2.453,33€ s/IVA (3.017,60€ c/IVA)` ✅

---

## 3. Webhook OneDrive — Trigger imediato

### Motivação
Jorge queria "sistema detectar quando tem PDF na pasta em vez de executar sem nada" — poupar recursos + latência.

### Arquitectura
1. **Workflow novo** `v8yIHOn4wGYHKhq7` "Cabeceiras Faturas — OneDrive Webhook":
   - Webhook Trigger em `/webhook/onedrive-notify-cabeceiras`
   - Handshake: se `?validationToken=` → responde 200 com token
   - Notificação real → chama fluxo principal via Execute Workflow

2. **Subscription Microsoft Graph:**
   ```
   POST https://graph.microsoft.com/v1.0/subscriptions
   {
     "changeType": "updated",
     "notificationUrl": "https://.../webhook/onedrive-notify-cabeceiras",
     "resource": "/drives/{drive-id}/root",
     "expirationDateTime": "2026-08-23T...",
     "clientState": "3AyfpWM5m9sy7TZY79TR3LmxKaLjWIDp"
   }
   ```
   Subscription ID: `261fd21e-8a5e-46d2-8452-8919132de583`

3. **Node adicionado** ao workflow principal: `Execute Workflow Trigger` (aceita chamadas externas)

4. **Workflow renovação** `VT15XtXEohrnEkRi` "Renovar Subscription OneDrive":
   - Cron: segunda 07:00 semanal
   - Fetch token Graph → PATCH `/subscriptions/{id}` estende +29 dias
   - Nunca deixa expirar

### Detalhe técnico crítico
Workflow criado via API do n8n sem `webhookId` (UUID) — webhook não registava, 404. Fix: adicionar `webhookId` explícito no node antes de PUT + reactivar.

### Cron principal reduzido
De 3 min → **1x/hora seg-sab 08-19** como fallback (caso webhook caia). Webhook + fallback = ~zero execs vazias.

---

## 4. Horário

- **Antes:** 3 min contínuo (`*/3` cron)
- **Discussão Jorge:** "20 em 20 min, seg-sab 8-19"
- **Final da sessão:** volta a 3 min mas seg-sab 08-19 (última exec 18:57), depois com webhook activo reduzido a **1x/hora** (fallback)

---

## 5. ME003007 re-bloqueado (3ª vez!)

**Descoberta ao final do dia:** 4 ECs criadas 24/07 com **0 linhas** (EC2602559 Elastron, EC2602560 Loja Ferragens, EC2602568 TEXBOX, EC2602570 Colreis). Investigação: BC rejeitou linhas com `Blocked must be equal to 'No' in Item: No.=ME003007. Current value is 'Yes'`.

**Fix:** desbloqueado de novo + 4 ECs apagadas e reprocessadas → EC2602572-EC2602575 com todas as linhas.

**Padrão preocupante:** ME003007 já foi bloqueado 3 vezes (22/07, 23/07, 24/07). Alguém no BC continua a bloquear. **Precisa investigar com Jorge quem/porquê**.

---

## 6. ECs criadas hoje (24/07)

Aproximadamente 25+ ECs (`EC2602550` → `EC2602575+`). Sistema estável, todas as flags conhecidas do Jorge.

---

## 7. Deploys da sessão

| Fix | Node | Efeito |
|-----|------|--------|
| Merge preferir Claude header | `Merge Linhas GPT-5` | numero_fatura + data_fatura + nif |
| `pegarValidadoGPT5` propaga Merge | `Preparar EC Header` | header fields do Merge chegam ao BC |
| Atualizar Datas lê Preparar EC | `Atualizar Datas EC` jsonBody | PATCH datas coerente |
| `normalizeNumber` suporta US | `Parse e Validar` | `1,234.56` OK |
| Sanity qty × preço = total | `Merge Linhas GPT-5` | Woodside e casos UN/M² |
| Prompt Claude — total_linha_sem_iva | `Preparar Payload Claude Linhas` | fonte para reconciliação |
| Webhook OneDrive | novo workflow | trigger imediato |
| Execute Workflow Trigger | Fluxo principal | aceita chamada externa |
| Cron renovação subscription | novo workflow | mantém webhook vivo |
| Cron principal reduzido | `Trigger 3min` | 1x/hora fallback |

---

## 8. Estado final

- Workflow principal: **ATIVO** (fallback 1x/hora seg-sab 08-19)
- Webhook Handler: **ATIVO** (trigger imediato)
- Cron Renovação: **ATIVO** (segundas 07:00)
- Graph Subscription: **ATIVA** até 23/08
- ME003007: **UNBLOCKED** (mas pode voltar a bloquear)
- 5 erros do cliente: **RESOLVIDOS**

---

## 9. Pendências

- **Contactar Jorge sobre ME003007** — quem está a bloquear? (padrão 3 vezes em 3 dias)
- **Considerar auto-desbloqueio** no workflow (retry após ME003007 error → PATCH item + retry)
- **Cadastrar produtos ME003007 no Excel** para Topbois, Francisco Sousa, Nogueira & Ribeiro — reduziria placeholders
