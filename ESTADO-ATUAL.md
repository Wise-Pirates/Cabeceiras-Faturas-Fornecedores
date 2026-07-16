# Estado Atual do Projeto — Snapshot

> **Última atualização:** 2026-07-03 (redesign completo do mapping — Excel-first)
> **Workflow principal n8n:** `21 — Faturas Fornecedores — Fase 1 OCR` (ID: `62wyOKnNBy0bnJUw`) — **56 nodes** — **DESATIVADO**
> **Novo Excel:** `Codigos Fornecedores.xlsx` (ID `01JSSR6ZOQ4BV4R2JA4JEJJD4BPSVMLY2K`) com tabela formal `MapeamentoItems`
> **Ver relatório do dia:** `relatorio_2026-07-03.md`
> **Pendente:** teste com nova arquitetura Excel-first
> **Workflow alertas n8n:** `21 — Faturas Fornecedores — Fase 6 Alertas POs` (ID: `M2I1eekcsJTi9Av7`) — **8 nodes**
> **Estado:** Fases 1, 2, 3, 4, 5, 6 IMPLEMENTADAS e TESTADAS. Fase 3 escreve APENAS em BC DEV. Emails temporariamente desativados.
> **Sistema estável:** 3 PDFs Flex2000 em múltiplos ciclos sem duplicação, sem loops, sem impacto em PROD.

---

## 1. O que já está a funcionar (Fase 1 — CONCLUÍDA)

### Workflow n8n (23 nodes, ativo)

```
Trigger 5min → Obter Token Graph → Listar Ficheiros OneDrive → Filtrar PDFs → Tem PDFs?
                                                                                 │
                                                                                 ▼
                                                                          Loop Faturas (batch=1)
                                                                                 │
                                                                                 ▼
                  Download PDF → Preparar Base64 → Preparar Payload Claude → Claude: Extrair Fatura
                                                                                 │
                                                                                 ▼
                                                                          Parse e Validar
                                                                                 │
                                                                                 ▼
                                                                          Extração OK?
                                                                          ┌──────┴───────┐
                                                                          ▼              ▼
                                                                       (sucesso)    (anomalia)
                                                                          │              │
                                                                          ▼              ▼
                                                                Preparar Destino   Preparar Log Erro
                                                                          │              │
                                                                          ▼              ▼
                                                                Criar Pasta Ano      Log Erro
                                                                          │              │
                                                                          ▼              ▼
                                                                Criar Pasta Mês   Email Anomalia
                                                                          │              │
                                                                          ▼              └──→ Loop
                                                                Obter ID Pasta Mês
                                                                          │
                                                                          ▼
                                                                     Mover PDF
                                                                          │
                                                                          ▼
                                                                    Preparar Log
                                                                          │
                                                                          ▼
                                                                    Log Sucesso ─→ Loop
```

### Capacidades já implementadas

| Funcionalidade | Estado | Notas |
|----------------|--------|-------|
| Polling OneDrive (5 min) | ✅ | Pasta: `faturas por inserir` |
| Download PDF | ✅ | Via Graph API |
| OCR com Claude Sonnet 4.6 | ✅ | Extrai 11 campos + linhas |
| Validação NIF (regex português) | ✅ | 9 dígitos, prefixo válido |
| Validação totais (tolerância 0,05€) | ✅ | Aceita arredondamentos OCR |
| Validação data (YYYY-MM-DD) | ✅ | |
| Detecção de anomalias | ✅ | Lista de problemas no log |
| Move PDF para `inseridas/YYYY/MM` | ✅ | Cria pastas se não existem |
| Log no Excel | ✅ | Via Graph API Tables |
| Email automático (anomalia) | ✅ | `wp@`, `brandao@` |
| Contexto piloto no prompt Claude | ✅ | Flex2000, Emma, madimorais |

### Resultado de teste real (29/06/2026)

3 faturas Flex2000 processadas com sucesso:

| Fatura | Total | Estado |
|--------|-------|--------|
| FT 31/206437 | 8.711,35€ | Extraido |
| FT 31/206787 | 3.009,59€ | Extraido |
| NC 82/382 | 5.724,09€ | Extraido |

---

## 1.5. Fase 2 — Validação BC (IMPLEMENTADA COMPLETA, aguarda teste)

### Capacidades acrescentadas
- **Token BC OAuth2** com app registration própria (diferente da do Graph)
- **Validação de fornecedor por NIF** via `workflowVendors` filtrado por `vatRegistrationNumber`
- **Deteção de duplicado de fatura** via `VendorLedgerEntriesWebService` filtrado por `External_Document_No`
- **Deteção de fornecedor bloqueado** — se vendor.blocked tem valor, marca anomalia
- Recolha do **vendor_no do BC** (ex: F00001 para Flex2000) — necessário para Fase 3 (inserção)

### Fluxo atualizado
```
Parse e Validar → Obter Token BC → Verificar Duplicado BC → Verificar Fornecedor BC → Avaliar Resultado BC → Extração OK?
                                                                                                              ├── (sucesso) → Preparar Destino → ... → Log Sucesso
                                                                                                              └── (anomalia) → Preparar Log Erro
```

### App registration BC (DIFERENTE da Graph)
- **Tenant:** `f41c8222-df66-449c-93b5-c1879e641cb2` (mesmo)
- **Client ID:** `fabec729-bb7e-48f8-a3cc-4a649ed4ab45`
- **Client Secret:** está em `Projetos/N8N_Cabeceiras/03_Reconciliacao/app/.env`
- **Company:** `Jorge Brand%C3%A3o Gon%C3%A7alves` (URL-encoded)

### WebServices BC usados
| WebService | Para quê | Filtro |
|------------|----------|--------|
| `VendorLedgerEntriesWebService` | Detectar duplicado de fatura | `External_Document_No eq 'XXX'` |
| `workflowVendors` | Validar fornecedor por NIF + obter `number` | `vatRegistrationNumber eq 'NIF'` |

### Anomalias possíveis na Fase 2
1. **Fornecedor não encontrado**: `Fornecedor NIF XXX nao existe no BC` → PDF fica para revisão, Jorge tem que criar o fornecedor
2. **Fornecedor bloqueado**: `Fornecedor XXX (Nome) esta BLOQUEADO no BC` → não processar
3. **Duplicado**: `Duplicado no BC: fatura ja registada como XXX` → não processar

### Decisões e normalização
- Número da fatura: remove prefixos `FT `/`NC ` antes de filtrar no BC (Claude extrai com prefixo, BC armazena sem)
- NIF é comparado tal-qual entre Claude OCR e BC (9 dígitos, sem formatação)
- Test confirmado: Flex2000 NIF 504663232 → `number=F00001`, `name=Flex2000- Prod.Flexiveis, SA`, `blocked=' '`

### ✅ NÃO precisa de intervenção do Jorge no BC
O endpoint `workflowVendors` já estava publicado por defeito. Inicialmente pensei que era preciso publicar um VendorTable WebService, mas o `workflowVendors` já expõe `vatRegistrationNumber` (NIF) e ~120 campos do fornecedor. **Nenhuma alteração no BC é necessária para Fase 2.**

---

## 2. Credenciais e IDs (NÃO PERDER)

### Azure / Graph
- **Tenant:** `f41c8222-df66-449c-93b5-c1879e641cb2`
- **Client ID:** `b8e798a0-6988-47aa-a4a5-0d4d1fe125eb`
- **Client Secret:** `<REDACTED_GRAPH_CLIENT_SECRET>`

### n8n
- **URL:** `https://primary-production-0fe7d.up.railway.app`
- **Workflow ID Fase 1:** `62wyOKnNBy0bnJUw`
- **API Key:** guardado nas memórias do Claude
- **Credenciais usadas:**
  - Anthropic API: `srXSApQJ2OBjvnxL` (Cabeceiras_Orcamentos)
  - Gmail: `ANhwrJfsekq0u1K3` (Gmail ACS)

### OneDrive (brandao@cabeceiras.pt)
- **Drive ID:** `b!fxQjRtmydUqtNaD2rLbsDpw9vi50zupCl2O41a8_kHwqy-IfbKGnQa-AvUPzD_nm`
- **Pasta `faturas por inserir`:** `01JSSR6ZOWI4JQ7ADEZFHZT24VNIWIU42R`
- **Pasta `faturas inseridas`:** `01JSSR6ZLDQWWQPDGYP5CZJAQFOZUTJGI2`
- **Pasta `Guia ou Fatura Assinadas`:** `01JSSR6ZO6ALSJKTWMFNDIGGYDVFDREZTO` (criada 29/06)
- **Excel Log:** `01JSSR6ZOQEYD67CHDWZDYCTBQAOQPLZKA`
  - Worksheet: `Log_Faturas`
  - Tabela: `Tabela1` (criada via Graph API)
- **Caminho completo:** `Cabeceiras-PARTILHA/Dep.  Financeiro/Automacao-tesouraria/Faturas/`
  - Nota: o `Dep.  Financeiro` tem **dois espaços** (encoded `%20%20`)

### Business Central (Fase 2 USA, Fase 3 vai usar)
- **Tenant:** `f41c8222-df66-449c-93b5-c1879e641cb2` (mesmo)
- **Client ID BC:** `fabec729-bb7e-48f8-a3cc-4a649ed4ab45` (DIFERENTE da do Graph)
- **Client Secret BC:** está em `Projetos/N8N_Cabeceiras/03_Reconciliacao/app/.env` (`BC_CLIENT_SECRET`)
- **Environment:** `PROD`
- **Company name (OData):** `Jorge Brand%C3%A3o Gon%C3%A7alves` (URL-encoded)
- **OData base:** `https://api.businesscentral.dynamics.com/v2.0/[TENANT]/PROD/ODataV4`
- **WebService de fornecedores:** `VendorLedgerEntriesWebService` (publicado e a funcionar)
- **API v2.0 (`/api/v2.0/companies`):** REJEITA estas credenciais (auth fail) — usar SEMPRE OData com nome de empresa

---

## 3. Respostas confirmadas pelo Jorge

| Pergunta | Resposta |
|----------|----------|
| Fornecedores piloto | Flex2000, Emma, madimorais |
| Código placeholder ERP | `ME003007` |
| Tolerância de preço | 0% (Fase 2 — comparação BC); OCR usa 0,05€ |
| Centro de custo default | `P1` |
| Pasta para guias assinadas | `Guia ou Fatura Assinadas` (separada) |
| Quem valida no BC | Tesouraria |
| Routing email anomalia | `wp@cabeceiras.pt` + `brandao@cabeceiras.pt` (teste); produção: `compras@` + `encomendas@` + `brandao@` |
| Pergunta 7 (scanner) | **Pendente** — verificar se já está configurado para OneDrive |

---

## 4. Decisões importantes tomadas durante o desenvolvimento

1. **Tolerância OCR 0,05€** — para aceitar arredondamentos naturais nos cálculos do Claude
2. **IF node com condição numérica** — `anomalias.length === 0` em vez de boolean (o operador `boolean` da v2 do IF estava bugado)
3. **Excel via Graph API tabela** (não via n8n Microsoft Excel node) — o node tinha erros de operação
4. **Body em modo `json` com `JSON.parse()`** — `specifyBody: "string"` enviava o body como chave de formulário
5. **Loop Faturas (SplitInBatches v3)** — outputs invertidos: Branch 1 = loop (não Branch 0)
6. **Branches do IF v2** — Output 0 = TRUE, Output 1 = FALSE (convenção n8n)

---

## 5. Roadmap (o que falta)

### Fase 2 — Validação BC (IMPLEMENTADA COMPLETA)
- [x] Token BC com app registration própria
- [x] Consulta `VendorLedgerEntriesWebService` por `External_Document_No` (duplicado)
- [x] Consulta `workflowVendors` por `vatRegistrationNumber` (validação fornecedor)
- [x] Deteção de fornecedor bloqueado
- [x] Recolha do `bc_vendor_no` para usar futuramente
- [ ] Cruzar com Notas de Encomenda (linhas) — adiada até Jorge ter POs no BC

### Fase 3 — Inserção BC — **AGUARDA REUNIÃO COM JORGE**
Documentação do projeto deixa claro que **inserção automática no BC é o core do projeto** (`product-discovery.md` secção 2.4, transcript linhas 11/46/53/66). A proteção contra erros é o estado "Gravado" (não regista contabilisticamente) que o próprio Jorge desenhou.

Robson e Jorge vão reunir-se para alinhar antes de implementar.
Pontos de alinhamento em `REUNIAO-JORGE-FASE3.md`.

### Fase 4 — Validação de Receção (IMPLEMENTADA)
- [x] Claude deteta `tem_assinatura_recepcao` no PDF (carimbo, assinatura, data de entrega)
- [x] Sistema lista pasta `Guia ou Fatura Assinadas` (criada: `01JSSR6ZO6ALSJKTWMFNDIGGYDVFDREZTO`)
- [x] Verifica se alguma guia/fatura assinada referencia o número da fatura ou da guia
- [x] Se NENHUMA prova (sem assinatura na fatura E sem guia separada) → anomalia "Sem prova de recepção"

**Como o Jorge usa:**
1. Quando chegar uma guia assinada (manual ou scanner), digitalizar e colocar em `Faturas/Guia ou Fatura Assinadas/`
2. O nome do ficheiro deve conter o número da guia (ex: `91-000516-assinada.pdf`) OU o número da fatura
3. Sistema cruza automaticamente

### Fase 5 — Duplicados Locais (IMPLEMENTADA)
- [x] Antes das chamadas BC, lê o log Excel via Graph API
- [x] Procura linhas com mesmo NIF + mesmo número de fatura (normalizado)
- [x] Ignora linhas com estado `ERRO` (pode tentar de novo)
- [x] Se encontrar → anomalia "Duplicado local: já processada em <data> como <ficheiro>"
- **Benefício:** poupa chamadas BC desnecessárias se a fatura já foi processada

### Fase 6 — Alertas Encomendas Atrasadas (IMPLEMENTADA — workflow separado)
- [x] Workflow novo: `21 — Faturas Fornecedores — Fase 6 Alertas POs` (ativo)
- [x] Cron: dias úteis às 8h00 (segunda a sexta)
- [x] Query BC `workflowPurchaseDocuments` por `documentType eq 'Order'` + `orderDate <= hoje-3dias`
- [x] Envia email HTML com tabela de POs atrasadas para `wp@`/`brandao@`
- [x] Cores: laranja se 3-7 dias atraso, vermelho se >7 dias
- [x] Não envia email se não houver POs atrasadas (silêncio = tudo OK)

**Estado actual:** vai disparar amanhã às 8h. Hoje (sem POs ainda na fase de pilot) provavelmente não terá nada para reportar.

### Fase 3 — Inserção BC
- [ ] POST `/purchaseInvoices` com cabeçalho
- [ ] POST `/purchaseInvoiceLines` para cada linha
- [ ] Mapear produtos: se não encontrar código → `ME003007`
- [ ] Set `costCenter = 'P1'`
- [ ] Estado: gravado (não registado)

### Fase 4 — Validação de Receção
- [ ] Adicionar ao prompt Claude: detetar assinatura/carimbo
- [ ] Procurar guia na pasta `Guia ou Fatura Assinadas` por número
- [ ] Bloquear inserção BC sem prova de receção

### Fase 5 — Deteção de Duplicados
- [ ] Consultar Excel log antes de processar
- [ ] Filtrar por NIF + número da fatura
- [ ] Se existir → marcar `Duplicado`

### Fase 6 — Alertas Proativos
- [ ] Cron diário (8h): listar Notas de Encomenda há 3+ dias sem fatura
- [ ] Email/Teams ao responsável

### Fase 7 — Aprendizagem e Configuração
- [ ] Tabela de templates por fornecedor (Excel ou JSON)
- [ ] Feedback loop: humano corrige → guarda exemplo
- [ ] Cores no log: vermelho (anomalia), amarelo (revisão), verde (OK)

---

## 6. Checklist de produção (responsável)

### Eu (técnico)
- [x] Fase 1 implementada e testada
- [ ] Fases 2–6
- [ ] Migrar secrets para env vars do n8n
- [ ] 20+ faturas reais de cada fornecedor piloto

### Jorge (cliente)
- [ ] Confirmar scanner → OneDrive (pergunta 7)
- [ ] Garantir que `ME003007` existe no BC e dá erro ao registar
- [ ] Confirmar permissões BC API (`purchaseInvoice` write)
- [ ] Equipa começar a criar Notas de Encomenda no BC

### Tesouraria
- [ ] Treino: abrir log, validar anomalias, criar produtos
- [ ] Definir SLA: faturas "Extraido" → validadas em 24h
