# Planeamento Completo — Sistema de Automação de Faturas de Fornecedores
> Projeto 21 · Cliente: Cabeceiras.pt · WisePirates  
> Data: 2026-06-25 · Autor: André Silva + Robson Advincula  

---

## Resumo Executivo

Automatizar o ciclo completo de receção de faturas de fornecedores: desde a digitalização com scanner até à inserção validada no ERP (Business Central), com cruzamento contra notas de encomenda, verificação de prova de receção e notificação de anomalias. A intervenção humana fica restrita a validação final e correção de exceções.

**Stack:** n8n (Railway) · Microsoft Graph API · Business Central API · **Claude Sonnet 4.6 Vision** · OneDrive (brandao@cabeceiras.pt)

> **⚠️ Para o estado atual e capacidades já implementadas, consulta `ESTADO-ATUAL.md` (snapshot vivo).**
> **Última atualização do snapshot: 2026-06-29 — Fase 1 (OCR + Log Excel) está concluída e em produção limitada.**

---

## Infraestrutura — Estado Atual (Fase 0 — CONCLUÍDA)

| Item | Valor | Estado |
|---|---|---|
| Tenant ID Azure | `f41c8222-df66-449c-93b5-c1879e641cb2` | ✅ |
| Client ID | `b8e798a0-6988-47aa-a4a5-0d4d1fe125eb` | ✅ |
| Client Secret | `<REDACTED_GRAPH_CLIENT_SECRET>` (válido até 26/02/2028) | ✅ |
| BC Environment | `PROD` | ✅ |
| BC Company | `Jorge Brandão Gonçalves` | ✅ |
| OneDrive user | `brandao@cabeceiras.pt` | ✅ |
| Pasta `Faturas/` | ID `01JSSR6ZPSPUMKRL27XBGZQNZBKTLDJIBW` | ✅ |
| Pasta `faturas por inserir/` | ID `01JSSR6ZOWI4JQ7ADEZFHZT24VNIWIU42R` | ✅ |
| Pasta `faturas inseridas/` | ID `01JSSR6ZLDQWWQPDGYP5CZJAQFOZUTJGI2` | ✅ |
| n8n Railway | `https://primary-production-0fe7d.up.railway.app` | ✅ |
| Permissões Graph | `Files.Read.All`, `Files.ReadWrite.All`, `Sites.Read.All` | ✅ |

---

## Arquitetura do Sistema

```
SCANNER
  │  PDF
  ▼
OneDrive: faturas por inserir/
  │  (polling a cada 5 min via n8n)
  ▼
[ETAPA 1] Deteção de novo ficheiro
  │
  ▼
[ETAPA 2] Extração OCR + IA (GPT-4o Vision)
  │  NIF, nome, data, nº fatura, nº guia, linhas (código, qtd, unidade, preço, IVA)
  ▼
[ETAPA 3] Verificação de duplicado
  │  Hash PDF + nº fatura + NIF → rejeitar se já processado
  ▼
[ETAPA 4] Validação do Fornecedor (BC API)
  │  Busca por NIF no BC → confirma nome (dupla validação)
  ▼
[ETAPA 5] Cruzamento com Nota de Encomenda (BC API)
  │  Busca EV pelo fornecedor → compara quantidade e preço por produto
  ▼
[ETAPA 6] Verificação de Prova de Receção
  │  Guia de remessa assinada no OneDrive OU receção no BC
  ▼
[ETAPA 7] Decisão
  ├── Sem anomalias → inserir no BC (estado "Inserir")
  └── Com anomalias → notificação email + sinalizar
  ▼
[ETAPA 8] Inserção no BC
  │  purchaseInvoices POST + linhas com códigos ERP (ou SSS/ME/PL)
  ▼
[ETAPA 9] Mover PDF
  │  faturas por inserir/ → faturas inseridas/YYYY/MM/
  ▼
[ETAPA 10] Validação Humana (fora do agente)
  │  Responsável no BC: Receção / Receção+Registo
  └── Produtos SSS → criar/associar código real → registar
```

---

## Fase 1 — Motor OCR e Extração de Dados
**Objetivo:** Ler qualquer fatura em PDF e extrair campos estruturados com alta precisão.  
**Duração estimada:** 1 semana  
**Entregável:** Node n8n "Extrair Fatura" funcional com 2-3 fornecedores piloto

### 1.1 Configuração do modelo de visão

- **Modelo:** GPT-4o (via OpenAI API) — melhor custo/benefício para documentos fiscais
- **Input:** PDF convertido para imagem base64 (cada página)
- **Prompt sistema:** instrução específica para faturas portuguesas com campos fiscais (NIF, ATCUD, IVA, etc.)

### 1.2 Campos a extrair por fatura

```json
{
  "nif_fornecedor": "string",
  "nome_fornecedor": "string",
  "morada_fornecedor": "string",
  "data_fatura": "YYYY-MM-DD",
  "numero_fatura": "string",
  "numero_guia_remessa": "string | null",
  "linhas": [
    {
      "descricao": "string",
      "codigo_erp": "string | null",
      "quantidade": "number",
      "unidade_original": "string",
      "unidade_erp": "string",
      "preco_unitario": "number",
      "taxa_iva": "number",
      "total_linha_sem_iva": "number",
      "total_linha_com_iva": "number",
      "centro_custo": "string"
    }
  ],
  "total_sem_iva": "number",
  "total_iva": "number",
  "total_com_iva": "number",
  "confianca": "high | medium | low"
}
```

### 1.3 Dicionário de layouts por fornecedor

Na **1ª fatura de cada fornecedor novo**, criar entrada em Google Sheets "Fornecedores_Config":

| NIF | Nome | Layout_Notas | Unidade_Config | Regras_Conversao | Centro_Custo_Default |
|---|---|---|---|---|---|
| 500123456 | Tecidos XYZ | NIF no canto sup. dir. | paleta | `{"paleta": {"para": "unidade", "fator": 500}}` | P1 |
| 500654321 | Placas ABC | IVA a vermelho col. 4 | placa | `{"placa": {"para": "m2", "fator": 6}}` | P1 |

### 1.4 Regras de conversão de unidades (exemplos reais)

```json
{
  "paleta_ripas":   { "unidade_erp": "UN",  "fator": 500 },
  "placa":          { "unidade_erp": "M2",  "fator": 6   },
  "caixa_agrafos":  { "unidade_erp": "UN",  "fator": 1000},
  "rolo":           { "unidade_erp": "M",   "fator": 50  }
}
```

### 1.5 Gestão de produtos desconhecidos

Se código ERP não for identificado:
- Inserir placeholder `SSS` (configurado no BC para gerar erro controlado ao registar)
- Variantes por categoria: `ME` (mercadoria), `PL` (plásticos) — a confirmar com Jorge

### 1.6 Critério de aceitação
- Extração correta em ≥90% dos campos nas 2-3 faturas piloto
- Valores totais com IVA reconciliados (soma linhas = total fatura, tolerância ±0.02€)

---

## Fase 2 — Validação de Fornecedor e Cruzamento BC
**Objetivo:** Confirmar fornecedor no BC e cruzar com Nota de Encomenda.  
**Duração estimada:** 1 semana  
**Entregável:** Nodes "Validar Fornecedor" e "Cruzar EV" funcionais

### 2.1 Validação de fornecedor (BC API)

```
GET /ODataV4/Company('Jorge Brandão Gonçalves')/Vendor
    ?$filter=taxRegistrationNumber eq '{NIF}'
    &$select=no,name,taxRegistrationNumber
```

**Lógica:**
1. Buscar pelo NIF extraído
2. Confirmar que o nome do BC bate com o nome extraído (normalizado, sem acento, sem LDA/SA)
3. Se NIF existe mas nome diverge → anomalia "Fornecedor com NIF diferente do registado"
4. Se NIF não existe → anomalia "Fornecedor desconhecido" (nunca inserir sem fornecedor válido)

### 2.2 Cruzamento com Nota de Encomenda (EV)

```
GET /ODataV4/Company('Jorge Brandão Gonçalves')/PurchaseHeader
    ?$filter=buyFromVendorNo eq '{vendorNo}' and documentType eq 'Order'
    &$select=no,buyFromVendorNo,orderDate,status
```

Para cada linha da fatura, cruzar com linhas da EV:
- Produto bate? ✅ / ❌
- Quantidade bate? ✅ / divergência X%
- Preço unitário bate? ✅ / divergência X%

**Tolerâncias configuráveis** (sugeridas):
- Preço: ±2%
- Quantidade: 0% (quantidade tem de bater exato)

### 2.3 Alerta de EV sem receção em 3 dias

Cron job separado (corre diariamente às 08h00):
```
GET EVs com orderDate < hoje - 3 dias AND status = 'Open' (sem receção)
→ Email para compras com lista de encomendas em atraso
```

### 2.4 Critério de aceitação
- Fornecedor validado corretamente em 100% dos casos piloto
- Cruzamento com EV identifica discrepâncias reais (teste com fatura propositadamente errada)

---

## Fase 3 — Verificação de Receção e Guia de Remessa
**Objetivo:** Garantir que só se insere o que foi fisicamente recebido.  
**Duração estimada:** 3-4 dias  
**Entregável:** Node "Verificar Receção" funcional

### 3.1 Formas de comprovar receção (por ordem de prioridade)

1. **Guia de remessa no OneDrive** — ficheiro com número de guia extraído da fatura, digitalizado e assinado, presente na pasta `faturas por inserir/` ou subpasta `guias/`
2. **Assinatura na própria fatura** — OCR deteta campo de assinatura preenchido
3. **Receção registada no BC** — consultar se a EV correspondente já tem receção dada no BC

### 3.2 Lógica de verificação

```
numero_guia = extraído da fatura pelo OCR
SE numero_guia:
    Procurar ficheiro com nome contendo numero_guia na pasta OneDrive
    SE encontrado → receção OK (guardar ID do ficheiro)
    SE não encontrado → verificar BC (EV tem receção?)
        SE receção no BC → OK
        SE não → ANOMALIA: "Sem comprovativo de receção"
SE sem numero_guia:
    Verificar assinatura na fatura (OCR)
    SE assinatura → OK
    SE não → ANOMALIA: "Sem guia de remessa e sem assinatura"
```

### 3.3 Critério de aceitação
- Sistema recusa inserção sem prova de receção (teste com fatura sem guia)
- Email de anomalia enviado corretamente com detalhe do problema

---

## Fase 4 — Inserção no BC e Notificações
**Objetivo:** Inserir fatura no BC e notificar a equipa.  
**Duração estimada:** 1 semana  
**Entregável:** Workflow completo end-to-end operacional

### 4.1 Inserção no BC

**Cabeçalho:**
```
POST /api/v2.0/companies({companyId})/purchaseInvoices
{
  "vendorNumber": "{vendorNo}",
  "vendorInvoiceNumber": "{numero_fatura}",
  "invoiceDate": "{data_fatura}",
  "postingDate": "{hoje}",
  "currencyCode": "EUR"
}
```

**Linhas:**
```
POST /api/v2.0/companies({companyId})/purchaseInvoices({id})/purchaseInvoiceLines
{
  "itemId": "{codigo_erp_ou_SSS}",
  "quantity": "{quantidade_convertida}",
  "unitOfMeasureCode": "{unidade_erp}",
  "directUnitCost": "{preco_unitario}",
  "taxPercent": "{taxa_iva}",
  "departmentCode": "{centro_custo}"
}
```

**Estado após inserção:** "Open" (não contabilizado) — aguarda validação humana para "Receive" ou "Receive and Invoice"

### 4.2 Verificação de duplicados (pré-inserção)

```
GET purchaseInvoices?$filter=vendorInvoiceNumber eq '{numero_fatura}' and vendorNumber eq '{vendorNo}'
SE existe → NÃO inserir → email "Fatura duplicada detetada: {numero_fatura}"
```

Adicionalmente: guardar hash SHA256 do PDF numa tabela Google Sheets "Faturas_Processadas" para deteção mesmo que o número de fatura seja diferente.

### 4.3 Mover PDF após inserção

```
POST /drive/items/{fileId}/copy → destino: faturas inseridas/{YYYY}/{MM}/
DELETE /drive/items/{fileId} da pasta "faturas por inserir"
```

### 4.4 Templates de email de notificação

**Sucesso:**
```
Assunto: ✅ Fatura inserida: {fornecedor} — {numero_fatura}
Corpo: Fatura de {fornecedor} ({data}) inserida no BC.
       Total: {valor_com_iva}€ | Produtos: {n_linhas} linhas
       Aguarda validação humana no BC.
```

**Anomalia:**
```
Assunto: ⚠️ Anomalia na fatura: {fornecedor} — {numero_fatura}
Corpo: [lista de anomalias detetadas]
       - Discrepância de preço: produto X — fatura {Y}€ vs EV {Z}€
       - Sem comprovativo de receção
       - Produto não identificado (SSS inserido)
       Ficheiro: {nome_pdf}
```

**Atraso EV (cron diário):**
```
Assunto: 🕐 Encomendas sem receção há +3 dias
Corpo: [tabela com nº EV, fornecedor, data encomenda, dias em atraso]
```

### 4.5 Routing de emails

| Tipo de anomalia | Destinatário |
|---|---|
| Discrepância preço/quantidade tecidos | comprasdetecidos@cabeceiras.pt |
| Produto não identificado (SSS) | compras1@cabeceiras.pt |
| Sem guia de remessa | compras2@cabeceiras.pt |
| Fornecedor desconhecido | compras1@cabeceiras.pt |
| Fatura duplicada | compras1@cabeceiras.pt + Jorge |

### 4.6 Critério de aceitação
- Fatura inserida corretamente no BC em estado "Open"
- PDF movido para `faturas inseridas/YYYY/MM/`
- Email enviado em menos de 2 minutos após deteção do PDF

---

## Fase 5 — Registo de Auditoria e Aprendizagem
**Objetivo:** Rastreabilidade total e melhoria contínua.  
**Duração estimada:** 2-3 dias  
**Entregável:** Google Sheet de log + mecanismo de feedback

### 5.1 Google Sheet "Log_Faturas" (auditoria)

Colunas:
| data_processamento | ficheiro_pdf | nif_fornecedor | nome_fornecedor | numero_fatura | data_fatura | total_com_iva | anomalias | estado | bc_invoice_id | operador |
|---|---|---|---|---|---|---|---|---|---|---|

### 5.2 Google Sheet "Fornecedores_Config" (dicionário)

Mantida automaticamente: quando um produto SSS é corrigido pelo humano no BC, registar a correspondência `descricao_fatura → codigo_erp_real` para usar nas próximas faturas do mesmo fornecedor.

### 5.3 Mecanismo de feedback (aprendizagem)

- Quando humano confirma no BC que uma anomalia era falso positivo → registar em "Padroes_Confirmados"
- Após 3 confirmações do mesmo padrão → remover esse tipo de alerta para aquele fornecedor
- Exemplo: fornecedor A sempre fatura "paleta" mas recebe-se como "UN" → regra criada automaticamente

---

## Fase 6 — Piloto Controlado
**Objetivo:** Validar o sistema com dados reais antes de escalar.  
**Duração estimada:** 2 semanas  
**Entregável:** Relatório de piloto + ajustes finais

### 6.1 Seleção dos fornecedores piloto

Selecionar **3 fornecedores** com critérios:
- 1 fornecedor com layout simples e unidade padrão (UN)
- 1 fornecedor com conversão de unidades (ex: paletas)
- 1 fornecedor que envia guia de remessa separada

**Ação de Jorge:** identificar os 3 fornecedores e enviar 2-3 faturas recentes de cada.

### 6.2 Procedimento do piloto

1. Digitalizar faturas piloto → depositar em `faturas por inserir/`
2. Deixar o agente correr (sem intervenção)
3. Verificar no BC se foram inseridas corretamente
4. Comparar com inserção manual que existia
5. Registar discrepâncias e corrigir o workflow
6. Repetir até 0 erros em 10 faturas consecutivas

### 6.3 Critérios de Go/No-Go para produção

| Critério | Meta |
|---|---|
| Taxa de extração correta | ≥ 95% dos campos |
| Falsos positivos de anomalia | ≤ 10% |
| Tempo de processamento por fatura | < 90 segundos |
| Faturas duplicadas aceites | 0% |
| Faturas sem receção inseridas | 0% |

---

## Fase 7 — Produção
**Objetivo:** Sistema a correr autónomo com monitorização.  
**Duração estimada:** 1 semana de transição  
**Entregável:** Go-live + runbook operacional

### 7.1 Procedimento de arranque

1. Scanner configurado para depositar PDFs em `faturas por inserir/` no OneDrive de `brandao@cabeceiras.pt`
2. Workflow n8n ativado (polling 5 minutos)
3. Equipa informada: formação de 30 min sobre o que muda no dia-a-dia
4. Responsável de validação humana definido (quem faz Receção/Registo no BC)

### 7.2 O que muda no dia-a-dia da equipa

| Antes | Depois |
|---|---|
| Inserir fatura manualmente no BC | Digitalizar no scanner |
| Verificar preços e quantidades manualmente | Receber email se há anomalia |
| Lembrar quais encomendas estão em atraso | Email diário automático às 08h00 |
| Criar produto novo quando não existe | Procurar código SSS no BC e substituir |
| Associar guia de remessa manualmente | Digitalizar guia no scanner (mesmo processo) |

### 7.3 Monitorização pós Go-Live

- **Diariamente:** verificar Google Sheet "Log_Faturas" — coluna "estado" não deve ter erros não tratados
- **Semanalmente:** relatório automático (n8n cron sexta 17h) com:
  - Nº faturas processadas
  - Nº anomalias detetadas vs confirmadas
  - Fornecedores com mais erros
  - Produtos SSS por corrigir
- **Mensal:** revisão do dicionário de fornecedores e regras de unidades

### 7.4 Runbook — Problemas comuns

| Problema | Causa provável | Solução |
|---|---|---|
| PDF não detetado | Scanner mandou para pasta errada | Verificar configuração do scanner |
| Extração incorreta | Novo layout de fornecedor | Adicionar entry em Fornecedores_Config |
| Falso positivo de duplicado | Nº fatura reutilizado por fornecedor | Verificar campo + limpar hash na Sheet |
| Produto SSS acumulando | Muitos produtos novos | Sessão semanal de criação de produtos no BC |
| BC retorna 401 | Client Secret expirado | Renovar no Azure (próxima expiração: 26/02/2028) |

---

## Cronograma Resumido

| Fase | Semanas | Entregável |
|---|---|---|
| 0 — Infraestrutura | **Concluída** | Credenciais + OneDrive + n8n |
| 1 — OCR + Extração | Semana 1-2 | Motor de leitura de faturas |
| 2 — Validação BC | Semana 2-3 | Validação fornecedor + cruzamento EV |
| 3 — Verificação Receção | Semana 3 | Validação guia remessa |
| 4 — Inserção + Notificações | Semana 4 | Workflow end-to-end |
| 5 — Auditoria + Aprendizagem | Semana 4-5 | Log + feedback |
| 6 — Piloto (3 fornecedores) | Semana 5-6 | Validação com dados reais |
| 7 — Produção | Semana 7 | Go-live |

**Total estimado: 6-7 semanas**

---

## Questões Abertas — A Confirmar com Jorge

| # | Questão | Impacto |
|---|---|---|
| 1 | Quais os 3 fornecedores piloto? | Fase 6 |
| 2 | Códigos placeholder: só `SSS` ou também `ME`, `PL`? | Fase 1 |
| 3 | Emails de compras: qual routing exato por tipo de anomalia? | Fase 4 |
| 4 | Tolerância de preço aceite: ±2% ou outra? | Fase 2 |
| 5 | Centro de custo default quando não se sabe: P1? | Fase 1 |
| 6 | Guias de remessa: vão para a mesma pasta `faturas por inserir/` ou pasta separada `guias/`? | Fase 3 |
| 7 | Scanner já está configurado? Qual o modelo/software? | Fase 7 |
| 8 | Quem é o responsável de validação humana no BC (Receção/Registo)? | Fase 7 |

---

## Dependências Técnicas

| Dependência | Responsável | Estado |
|---|---|---|
| OpenAI API key (GPT-4o) | Wise/Robson | ⏳ confirmar se já existe no n8n |
| Company ID do BC (UUID) | Robson — buscar via API | ⏳ pendente |
| Endpoint BC para `purchaseOrders` (EVs) | Robson — testar | ⏳ pendente |
| Endpoint BC para `vendors` (por NIF) | Robson — testar | ⏳ pendente |
| Configuração do scanner → OneDrive | Jorge / IT Cabeceiras | ⏳ pendente |
| Emails de compras (compras1/2/3) configurados no n8n | Robson | ⏳ pendente |
