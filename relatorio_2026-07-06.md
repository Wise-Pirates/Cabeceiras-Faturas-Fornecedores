# Relatório — Projeto 21 — Segunda 2026-07-06

**Cliente:** Cabeceiras.pt (Jorge Brandão)
**Workflow n8n:** `62wyOKnNBy0bnJUw` — 56 nodes — **ATIVO** em produção
**Estado final:** Sistema aceita **PDF + JPEG/PNG**, valida totais, marca `ATENÇÃO VALORES` quando OCR erra.

---

## 1. Decisão do Jorge (manhã)

Sobre Centro de Custo: **"Conforme aparece no ficheiro Excel"**.

**Aplicado (deploy 08:40):**
- `Aplicar Mapeamento Excel` guarda `{codigo, ccusto}` por par NIF+PK
- Expõe `centro_custo_default` (primeiro CCusto do fornecedor)
- `Preparar EC Header`, `Verificar EC Criado`, `Atualizar Datas EC` PATCH usam valor dinâmico do Excel
- Se fornecedor não tem CCusto no Excel → `shortcutDimension1Code=''` (Jorge preenche manualmente)

**Nota:** Fluxo teve de ser **reordenado** para o `Ler Excel Codigos Fornecedores` correr **antes** do POST do header (antes corria em paralelo e o header saía sem CCusto).

---

## 2. Suporte a imagens (JPEG/PNG)

Cabeceiras usa **17 faturas de teste** — algumas em JPEG (fotos), não PDF.

**Alterações:**
1. Nó `Filtrar PDFs` → renomeado `Filtrar Faturas`, aceita `.pdf/.jpg/.jpeg/.png/.gif/.webp`
2. `Preparar Payload Claude` deteta extensão:
   - PDF → `type: 'file'` + `data:application/pdf;base64,...`
   - Imagem → `type: 'image_url'` + `data:image/jpeg;base64,...`
3. `Preparar Payload Claude Guia` (verificar assinatura) — mesma lógica

---

## 3. Descontos + Comentário "ATENÇÃO VALORES"

**Análise das 17 faturas piloto — 3 fornecedores, 3 padrões distintos:**

| Fornecedor | Padrão desconto | OCR extrai |
|-----------|-----------------|-----------|
| Flex2000 | Cabeçalho: "Desconto p. pagamento X%" | `desconto_global_percent` |
| TOPBOIS | Sem desconto (0,00) | 0 |
| FSM Tecidos | Por linha (col %Desc, varia 0/10/35) | `desconto_percent` por linha |

**Regras implementadas:**
1. **Prompt OpenAI** — extrai `desconto_global_percent` (cabeçalho) + `desconto_percent` por linha
2. **`Verificar EC Criado`** — aplica `lineDiscountPercent`:
   - Prioridade: linha > global > 0
3. **Validação total** — calcula `total_previsto` (linhas × desconto × IVA) vs `total_com_iva` OCR
4. **Se divergência > 0,05€** → `postingDescription = 'AUTOMAÇÃO - ATENÇÃO VALORES'` (Jorge vê no header)

**Design de comment corrigido:** primeira tentativa criou linha `type='Comment'` — BC recusou (só aceita G/L Account, Item, Resource, Fixed Asset, Charge, Allocation). Solução: usar campo `postingDescription` do header.

**Nota fluxo:** cálculo da divergência ficou no `Preparar EC Header` (corre antes do POST) para o PATCH `Atualizar Datas EC` já ter o flag disponível.

---

## 4. Fix crítico — vírgulas de milhares no JSON

GPT-4o gerou JSON inválido com números tipo `"total_linha_com_iva": 1,171.65` (vírgula como separador de milhares). Falhava `JSON.parse`.

**Fix no `Parse e Validar`:** regex remove vírgulas entre dígitos:
```js
jsonStr = jsonStr.replace(/(\d),(\d{3}(?:[.,]|\D|$))/g, '$1$2');
```

Aplicado 2× para cobrir casos como `1,234,567.89`.

---

## 5. Prefixo FT — aceita mais prefixos

Antes: forçava `FT ` se não fosse `FT|NC|FR|GT|GS|GR|VD`.

Novo: aceita também `FAT|FRA|FC|FS|FE` (FSM Tecidos usa `FAT`).

---

## 6. Renomear ao mover PDF — DECISÃO REVERTIDA

Implementei rename `EC{numero}_{nome_original}` mas Jorge disse "burrice".

**Decisão final:**
- Não renomear
- Não usar Incoming Documents do BC (complexo)
- Ficheiro fica na pasta `faturas inseridas/YYYY/MM/` com nome original
- Jorge trabalha só com dados da EC no BC — imagem só é consultada se necessário (auditoria)

---

## 7. Testes em BC PROD

| EC | Fornecedor | Nº Fatura | Resultado | Notas |
|----|-----------|-----------|-----------|-------|
| EC2602150 | Flex2000 | FT 31/206437 | Apagada | Duplicada de EC2602181 sem desconto |
| EC2602181 | Flex2000 | FT 31/206437 | ✅ | PATCH aplicou 2% desconto → total 8.711,35€ (bate) |
| EC2602182 | Flex2000 | 31/206787 | ✅ | Antes do fix "FT sempre" — sem prefixo |
| EC2602185 | TOPBOIS | FT 2026/1414 | Apagada | Ficou vazia (erro Aplicar Mapeamento) |
| EC2602186 | TOPBOIS | FT 2026/1414 | ✅ | 1ª JPEG processada com sucesso |
| EC2602187 | TOPBOIS | FT 2026/1445 | Apagada | ATENÇÃO VALORES → Jorge apagou p/ retest |
| EC2602189 | TOPBOIS | FT 2026/1445 | Apagada | 2ª tentativa — apagada p/ retest |
| **EC2602206** | TOPBOIS | FT 2026/1445 | ⚠️ | 3ª tentativa — `ATENÇÃO VALORES` (diff 69,96€, 1 linha errada L4 preço 102,01→72,00) |

**Estado atual das ECs abertas para Jorge validar (4):**
- ✅ EC2602181 Flex2000 8.711,35€ (Jorge já OK)
- ✅ EC2602182 Flex2000 3.071,01€
- ✅ EC2602186 TOPBOIS 1.213,27€
- ⚠️ EC2602206 TOPBOIS 3.517,07€ (deveria ser 3.447,11€ — corrigir L4 preço)

---

## 8. Conclusão OCR de imagens

**GPT-4o com fotos JPEG é irregular** — 3 tentativas da mesma imagem (Image (1).jpeg TOPBOIS) deram 3 resultados diferentes:
1. Anomalia (JSON com vírgula milhares)
2. EC2602187 — 3 linhas erradas
3. EC2602206 — 1 linha errada

**Discussão com Jorge:**
- Confirmado: converter JPEG → PDF **NÃO ajuda** (PDF-de-imagem tem mesma qualidade que JPEG)
- Só ajuda **PDF-nativo** (gerado pelo ERP do fornecedor)
- **Amanhã (2026-07-07) vamos testar com PDFs** — Cabeceiras vai pedir aos fornecedores para enviar PDF

---

## 9. Deploys realizados hoje

| Hora | O que | Resultado |
|------|-------|-----------|
| 08:39 | CCusto dinâmico do Excel | error (Excel corria depois do POST) |
| 08:40 | Correção: Aplicar Mapeamento Excel guarda {codigo, ccusto} | OK |
| 09:17 | Prompt: desconto_global + por linha; regex prefixo FAT/FRA...; validação total → linha Comment | error (BC rejeita type=Comment) |
| 09:25 | Payload OpenAI aceita PDF + imagens (image_url) | OK — mas Filtrar PDFs ainda barrava .jpeg |
| 09:40 | Filtrar PDFs → Filtrar Faturas (aceita .pdf/.jpg/.jpeg/.png/.gif/.webp) | OK — mas fluxo com Excel após POST |
| 09:48 | **Reorder fluxo:** Ler Excel + Aplicar Mapeamento Excel **antes** de Preparar EC Header | OK ✅ EC2602186 sucesso |
| 10:00 | Rename ficheiro ao mover (`EC{num}_original`) | Reverted (Jorge disse desnecessário) |
| 10:20 | Move cálculo divergência para Preparar EC Header + PATCH ajusta postingDescription | OK |
| 11:01 | Sanitizar JSON: remover vírgulas de milhares antes do JSON.parse | OK ✅ |

**9 deploys, workflow desativado no fim (aguardar mais faturas PDF amanhã).**

---

## 10. Estado final do sistema

**Workflow:** DESATIVADO no fim do dia (regra: não deixar PROD ativo sem supervisão)
**Custo OpenAI hoje:** ~$0,20 (aproximado, ~15 execuções úteis)
**Faturas em produção validadas:** 2 (Flex2000 EC2602181 + TOPBOIS EC2602186)
**ATENÇÃO VALORES pendente:** EC2602206 (Jorge revê L4)

---

## 11. Próximos passos (2026-07-07)

1. **Testar com faturas PDF reais** — pedir a fornecedores (TOPBOIS pelo menos)
2. Se PDFs derem 100% precisão consistente → validar sistema em produção contínua
3. **Se ainda houver erros com PDFs**, avaliar:
   - Prompt em 2 passos (extrair tabela + validar linha-a-linha + gerar JSON)
   - Segundo modelo (Claude/Gemini) como "review" quando ATENÇÃO
4. Perguntas ainda em aberto ao Jorge:
   - EC2602182 sem prefixo FT — deixar assim
   - EC2602206 — corrigir L4 e postar
