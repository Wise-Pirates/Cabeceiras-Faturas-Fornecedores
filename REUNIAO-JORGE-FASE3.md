# Reunião com Jorge — Decisão sobre Fase 3 (Inserção no BC)

> Documento de apoio para reunião · Robson + Jorge Brandão

---

## Estado actual do projeto

Já está em produção e a funcionar:

| Fase | O que faz | Estado |
|------|-----------|--------|
| 1 | OCR Claude lê PDFs e extrai NIF/nº/data/totais/linhas | ✅ Operacional |
| 2 | Valida fornecedor por NIF no BC + deteta duplicado | ✅ Operacional |
| 4 | Verifica prova de receção (assinatura ou guia) | ✅ Operacional |
| 5 | Deteta duplicado local no log Excel | ✅ Operacional |
| 6 | Alerta diário de POs atrasadas | ✅ Operacional |

**Tudo escreve para o Excel `Log_Faturas`** — onde a Tesouraria já vê tudo validado.

---

## A decisão pendente: Fase 3 (Inserção automática no BC)

### O que está documentado (palavras tuas, Jorge — transcript 25/06)

> *"E que as insira no ERP."* (linha 11)
> *"Ele insere só; depois um responsável valida humanamente."* (linha 53)
> *"Sim, ele só encerra [insere]. Isto já poupa 90% do trabalho."* (linha 66)

E em `product-discovery.md` (secção 2.4):
> *"Inserção automática: se todas as validações passarem, o Agente insere os dados no ERP, transformando a Nota de Encomenda em Encomenda de Venda. O Agente apenas insere ("grava"); não efetua o registo contabilístico final."*

---

## A proteção que tu próprio desenhaste

O sistema **não regista contabilisticamente**. Apenas insere em estado **"Open" / "Gravado"**.

```
Fluxo proposto:
  Sistema lê fatura → Valida tudo → Insere no BC em "Gravado"
       ↓
  Tesouraria abre o BC, vê a fatura pronta
       ↓
  Tesouraria clica "Receção" ou "Registar" — só ela é que decide
```

**Se o Claude ler errado:**
- Erro entra como fatura "Open" no BC
- Tesouraria revê (já fazia antes)
- Anula com 1 clique
- Zero impacto contabilístico

---

## O que precisamos do Jorge confirmar

| Pergunta | Por quê |
|----------|---------|
| **A** Avançamos com a Fase 3 como originalmente desenhada? | É o coração do projeto |
| **B** Limita ao Flex2000 nas primeiras 2 semanas? | Reduzir risco de aprendizagem |
| **C** Confirmas que o BC tem `ME003007` configurado para gerar erro ao registar? | Para produtos não identificados |
| **D** Permissões BC API: a app `fabec729-...` pode criar `purchaseInvoice`? | Tem que ter permissão de escrita |
| **E** Centro de custo `P1` por defeito é OK? | Mudas manualmente depois quando for VNG |
| **F** Pasta `Guia ou Fatura Assinadas` criada — o scanner do estofo já está pronto a digitalizar guias? | Para Fase 4 funcionar 100% |

---

## Custos e tempo se avançarmos

| Item | Estimativa |
|------|-----------|
| Implementação Fase 3 | 2-3 dias de desenvolvimento |
| Testes com 5-10 faturas piloto | 3 dias |
| Validação Tesouraria (2 semanas) | Em paralelo |
| **Go-live com 3 fornecedores piloto** | ~3 semanas a partir do GO |

---

## Alternativa (se Jorge não quiser avançar)

Mantemos como está. Tesouraria abre o Excel `Log_Faturas`, copia os dados (já validados) para o BC manualmente. Poupa ~80% do trabalho (a parte de digitar/validar). Mas perde os 20% finais (o copy-paste).

---

## Resumo executivo (1 frase)

> **O sistema já lê, valida e prepara tudo. Só falta o último passo — escrever no BC. O Jorge desenhou o projeto exatamente assim e a proteção contra erros é o estado "Gravado" sem registo contabilístico. Avançamos ou paramos aqui?**
