# Análise da Estratégia de Automação — Comparação com o Sistema Atual

**Data:** 17/07/2026
**Base:** Documento *"Estratégia Automação Facturas"* enviado pelo Jorge Brandão
**Objetivo:** confirmar o que já está alinhado e clarificar os pontos que precisam de decisão do Jorge antes de avançarmos.

---

## 1. Resumo

Lemos com atenção o documento do Jorge. **A maior parte do que o Jorge descreve já está implementada** no sistema atual. Identificámos **4 pontos** que precisam de decisão ou clarificação do Jorge antes de fazer alterações — para evitar retrabalho.

---

## 2. O que já está a funcionar (alinhado ✅)

| Requisito do Jorge | Estado no sistema |
|---|---|
| Criar automaticamente Encomendas de Compra no ERP | ✅ Sim — cada fatura vira uma EC no BC |
| Identificar Nº Fornecedor (código + contribuinte + nome) | ✅ Sim — via OCR + validação no BC |
| Preencher Data de Registo (data da fatura) | ✅ Sim — extraído automaticamente |
| Preencher Nº Factura Fornecedor | ✅ Sim |
| Preencher "Sua Referência" = *AUTOMAÇÃO* | ✅ Sim — todas as ECs criadas pelo sistema têm esta marca |
| Inserir produtos usando Excel `Codigos Fornecedores` | ✅ Sim — corrigido ontem (agora identifica mesmo com formatação inconsistente) |
| Verificar duplicados de fatura antes de criar | ✅ Sim — mas ver ponto 5 abaixo |

**Conclusão:** o sistema faz o essencial do que o Jorge descreveu.

---

## 3. Pontos que precisam de decisão do Jorge

### 🟡 Ponto 1 — Campo "Cód.Comprador (JB)"

**O que o Jorge escreve:** *"o agente deverá reconhecer e inserir sem erros o Cód.Comprador (JB)"*

**Situação atual:** o sistema **não preenche este campo**. As ECs criadas ficam com `Cód.Comprador` vazio.

**Pergunta ao Jorge:**
> Confirmas que queres que TODAS as ECs criadas pela automação fiquem com `Cód.Comprador = JB`? (é sempre este valor, ou depende de algo?)

**Se sim:** implementação simples (10 min). Passa a preencher sempre.

---

### 🟡 Ponto 2 — Nome das pastas OneDrive

**O que o Jorge escreve:** *"Quando inserir-mos facturas na pasta 'faturas inseridas' automaticamente e imediatamente o agente irá processar"*

**Situação atual:** temos duas pastas com nomes que podem ser confusos:
- **`faturas por inserir`** = onde o Jorge coloca as faturas novas para o agente processar (INPUT)
- **`faturas inseridas`** = onde o agente ARQUIVA as faturas depois de processadas com sucesso (OUTPUT / histórico)

**No documento do Jorge, ele chama "faturas inseridas" ao que hoje é "faturas por inserir"** — provavelmente é só uma questão de vocabulário.

**Pergunta ao Jorge:**
> Estás confortável com os nomes atuais ("faturas por inserir" = onde colocas para processar; "faturas inseridas" = arquivo depois de processadas)? Se preferires trocar nomes, dizes quais.

---

### 🔴 Ponto 3 — O que fazer quando há fatura DUPLICADA

**Este é o ponto mais importante — mudança de comportamento significativa.**

**O que o Jorge escreve:** *"Caso identifique alguma duplicação deverá escrever na 'Linha' de produtos um 'Comentário' com a Descrição 'DOCUMENTO DUPLICADO'"*

**Situação atual:** quando o sistema deteta que uma fatura já existe no BC:
- **NÃO cria** nova EC (para evitar 2 registos da mesma fatura)
- Regista no ficheiro de log com marca `Anomalia`
- PDF fica em `faturas_com_erro`
- Jorge fica com a fatura original intacta

**O que o Jorge quer:** criar uma EC mesmo assim, com uma linha "Comentário" a dizer `DOCUMENTO DUPLICADO`.

**Trade-off importante para o Jorge decidir:**

| Comportamento atual | Comportamento pedido |
|---|---|
| Nunca cria 2 ECs para a mesma fatura no BC | Vai criar EC nova em cima da original — passa a haver 2 ECs no BC para a mesma fatura |
| Jorge tem que ir ao log ver os duplicados | Jorge vê a EC nova aparecer com marca `DOCUMENTO DUPLICADO` |
| BC fica limpo | BC tem que ser limpo manualmente (Jorge apaga a EC duplicada) |

**Dificuldade técnica adicional:**
O BC não aceita linhas do tipo "Comentário" via API — só aceita Item, Conta, Recurso, etc. Vamos ter que usar um item placeholder (ex: `ME003007`) com a descrição `DOCUMENTO DUPLICADO`. Fica visível na EC mas não é literalmente uma "linha de comentário" no sentido BC.

**Perguntas ao Jorge:**
> 1. Queres mesmo que se crie EC duplicada com marca "DOCUMENTO DUPLICADO"? (Alternativa: manter comportamento atual — não criar, avisar por email)
> 2. Se sim, aceitas que a "linha comentário" seja implementada como um item placeholder com a descrição `DOCUMENTO DUPLICADO` (visualmente igual)?

---

### 🟡 Ponto 4 — Cobertura da verificação de duplicados

**O que o Jorge escreve:** *"deverá pesquisar no ERP em 'Encomendas de Compra' e 'Facturas de Compra Registadas' se já existe alguma factura com o mesmo numero"*

**Situação atual:** o sistema procura em:
- ✅ **Encomendas de Compra em aberto** (`workflowPurchaseDocuments`)
- ✅ **Movimentos do fornecedor** (`VendorLedgerEntries` — normalmente inclui facturas registadas)

**Dúvida técnica:** é preciso confirmar se `VendorLedgerEntries` cobre EXATAMENTE o que Jorge chama "Facturas de Compra Registadas". Provavelmente sim, mas vale a pena Jorge confirmar com um caso concreto.

**Pergunta ao Jorge:**
> Podes indicar uma fatura já registada no BC (Nº) para eu testar se o sistema a deteta como duplicado se ela fosse reenviada?

---

## 4. Resumo — o que precisamos do Jorge

Se possível responder a estas 4 perguntas:

1. **Cód.Comprador (JB)** — confirmas preencher sempre `JB`? (SIM/NÃO)
2. **Nomes das pastas** — atuais estão OK ou queres trocar? (OK / renomear para X)
3. **Duplicados** — criar EC com marca "DOCUMENTO DUPLICADO" ou manter comportamento atual (não cria, avisa por log)?
4. **Teste duplicados** — indicar Nº de uma fatura já registada no BC para testarmos.

Assim que tivermos as respostas, avançamos com as alterações rapidamente.

---

*Documento preparado por Robson (equipa técnica) — 17/07/2026.*
