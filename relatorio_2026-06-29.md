# Relatório Completo — Projeto 21 Faturas Fornecedores

> **Data:** 2026-06-29
> **Cliente:** Cabeceiras.pt — Jorge Brandão (`brandao@cabeceiras.pt`)
> **Equipa:** Robson Advincula (WisePirates)
> **Autor do relatório:** Claude Code (Sonnet 4.6)
> **Estado geral:** 5 fases em produção · 1 fase aguarda reunião · 1 fase opcional pendente

---

## 1. RESUMO EXECUTIVO

Sistema automático de receção, validação e processamento de faturas de fornecedores para Cabeceiras.pt. Substitui o trabalho manual de digitar dados das faturas no ERP Business Central, com validações automáticas em múltiplos níveis.

**Em produção (5 fases operacionais):**
- Lê PDFs com Claude OCR
- Valida fornecedor no BC (NIF, bloqueado)
- Deteta duplicados (locais + no BC)
- Verifica prova de receção (assinatura/guia)
- Alerta diário sobre encomendas atrasadas no BC

**Pendente:** Fase 3 (inserção automática no BC) — aguarda reunião com Jorge.

---

## 2. ARQUITETURA DO SISTEMA

### 2.1 Componentes

```
┌─────────────────┐   PDF    ┌──────────────────────┐
│ SCANNER físico  │ ───────► │ OneDrive: faturas    │
│ (Cabeceiras)    │          │ por inserir/         │
└─────────────────┘          └──────────┬───────────┘
                                        │ polling 5 min
                                        ▼
                             ┌──────────────────────┐
                             │ n8n Workflow         │
                             │ Fase 1+2+4+5 OCR     │ 31 nodes
                             │ (Railway)            │
                             └──────────┬───────────┘
                                        │
              ┌─────────────────────────┼─────────────────────────┐
              ▼                         ▼                         ▼
    ┌─────────────────┐     ┌──────────────────┐    ┌──────────────────┐
    │ Claude API      │     │ MS Graph API     │    │ BC OData API     │
    │ (OCR vision)    │     │ (OneDrive/Excel) │    │ (vendor/PO/lentry)│
    └─────────────────┘     └──────────────────┘    └──────────────────┘

┌──────────────────────┐                ┌──────────────────────┐
│ n8n Workflow         │  cron diário   │ Gmail (Brandão)      │
│ Fase 6 Alertas POs   │ ──────────────►│ wp@/brandao@         │
│ (8 nodes)            │  seg-sex 8h    │                      │
└──────────────────────┘                └──────────────────────┘
```

### 2.2 Stack tecnológico

| Camada | Tecnologia | Onde |
|--------|------------|------|
| Orquestração | n8n | Railway: `https://primary-production-0fe7d.up.railway.app` |
| OCR | Claude Sonnet 4.6 + `pdfs-2024-09-25` beta | API direta via HTTP |
| Storage | OneDrive (SharePoint) | brandao@cabeceiras.pt |
| Log | Excel via Graph API Tables | `Log_Faturas.xlsx` |
| ERP | Business Central via OData v4 | `Jorge Brandão Gonçalves` (PROD) |
| Email | Gmail OAuth | Credencial `ANhwrJfsekq0u1K3` |

---

## 3. ESTADO DAS FASES (DETALHADO)

### ✅ Fase 0 — Infraestrutura (CONCLUÍDA)
- 2 Azure App Registrations operacionais (Graph + BC)
- Pastas OneDrive criadas
- Permissões Graph: `Files.Read.All`, `Files.ReadWrite.All`, `Sites.Read.All`
- Workflow n8n base

### ✅ Fase 1 — Motor OCR (CONCLUÍDA — em produção)

**Capacidades:**
- Polling OneDrive a cada 5 minutos (`faturas por inserir`)
- Download PDF via Graph API
- Conversão para base64 e envio ao Claude
- Extração de 11 campos: NIF, nome, data, número, guia remessa, linhas (descrição, qtd, unidade, preço, IVA), totais
- Validação: NIF regex português, totais com tolerância 0,05€, data formato
- Move PDF para `inseridas/YYYY/MM/` após sucesso
- Cria pastas ano/mês automaticamente

**Test confirmado (29/06/2026):** 3 faturas Flex2000 processadas (FT 31/206437, FT 31/206787, NC 82/382)

### ✅ Fase 2 — Validação BC (CONCLUÍDA COMPLETA)

**Capacidades:**
- Token BC com app registration dedicada (`fabec729-bb7e-48f8-a3cc-4a649ed4ab45`)
- Lookup de fornecedor por NIF via `workflowVendors?$filter=vatRegistrationNumber eq 'XXX'`
- Detecção de fornecedor bloqueado (campo `blocked`)
- Recolha de `bc_vendor_no` (ex: F00001 para Flex2000)
- Detecção de duplicado de fatura via `VendorLedgerEntriesWebService?$filter=External_Document_No eq 'XXX'`

**Anomalias possíveis:**
- `Fornecedor NIF XXX não existe no BC` → Jorge tem que criar fornecedor
- `Fornecedor X (Nome) está BLOQUEADO no BC` → não processar
- `Duplicado no BC: já registada como X` → não processar

### ⏸️ Fase 3 — Inserção BC (AGUARDA REUNIÃO JORGE)

**Documentação confirma que é core do projeto:**
- Transcript Jorge linha 11: *"E que as insira no ERP."*
- Transcript Jorge linha 53: *"Ele insere só; depois um responsável valida humanamente."*
- `product-discovery.md` secção 2.4: *"Inserção no ERP e Validação Humana"*
- `planeamento-projeto.md` Fase 4 com endpoints prontos

**Endpoint preparado:**
```
POST /api/v2.0/companies({id})/purchaseInvoices
POST /api/v2.0/companies({id})/purchaseInvoices({id})/purchaseInvoiceLines
Estado: "Open" (não contabilizado)
```

**Proteção contra erros (desenhada pelo Jorge):** estado "Gravado" sem registo contabilístico — Tesouraria valida e regista manualmente.

**Próximo passo:** reunião alinhar 6 perguntas (ver `REUNIAO-JORGE-FASE3.md`):
- A) Avançamos com a Fase 3?
- B) Limitar ao Flex2000 nas primeiras 2 semanas?
- C) `ME003007` configurado para gerar erro ao registar?
- D) App `fabec729-...` tem permissões para criar `purchaseInvoice`?
- E) Centro de custo `P1` por defeito é OK?
- F) Scanner do estofo pronto para digitalizar guias?

### ✅ Fase 4 — Validação de Receção (CONCLUÍDA — em produção)

**Capacidades:**
- Claude analisa o PDF e detecta `tem_assinatura_recepcao` (carimbo, assinatura, "Recebido", data de entrega)
- Sistema lista pasta `Guia ou Fatura Assinadas` (criada 29/06)
- Cruza por número de guia OU número de fatura (normalizado: sem espaços/traços/slashes)
- Se nenhuma prova → anomalia "Sem prova de receção"

**Como funciona para a equipa:**
1. Guia assinada chega → scanner ou digitalização → pasta `Guia ou Fatura Assinadas/`
2. Nome do ficheiro deve conter nº da guia ou fatura (ex: `91-000516-assinada.pdf`)
3. Sistema cruza automaticamente na próxima execução

### ✅ Fase 5 — Duplicados Locais (CONCLUÍDA — em produção)

**Capacidades:**
- Antes das chamadas BC (poupa custos), lê o log Excel via Graph API
- Procura match por NIF + número fatura normalizado
- Ignora linhas com estado `ERRO` (podem ser retentadas)
- Anomalia: `Duplicado local: já processada em <data> como <ficheiro>`

### ✅ Fase 6 — Alertas POs Atrasadas (CONCLUÍDA — em produção)

**Workflow separado** (ID `M2I1eekcsJTi9Av7`)

**Capacidades:**
- Cron: seg-sex às 8h00
- Query: `workflowPurchaseDocuments?$filter=documentType eq 'Order' and orderDate le YYYY-MM-DD`
- Filtra POs com mais de 3 dias sem receção
- Email HTML com tabela: laranja (3-7 dias atraso), vermelho (>7 dias)
- Envia para `wp@` e `brandao@`
- Silêncio se 0 POs atrasadas

### ⏳ Fase 7 — Templates por Fornecedor + Aprendizagem (PENDENTE)

**Documentado em:** `product-discovery.md` linhas 59, 65, 66 e `technical-note.md` linha 40

**Duas sub-fases:**
- **7a Templates:** config inicial por fornecedor (cores visuais OU UI) para acelerar OCR
- **7b Feedback loop:** sistema aprende com correções da Tesouraria

**Recomendação:** Adiar até ter ~100 faturas processadas (precisamos de dados reais para configurar templates).

---

## 4. CREDENCIAIS E IDS (PROTEGIDOS — NÃO PARTILHAR)

### 4.1 Azure (2 app registrations)

| App | Para quê | Client ID | Validade |
|-----|----------|-----------|----------|
| **Graph** | OneDrive + Excel | `b8e798a0-6988-47aa-a4a5-0d4d1fe125eb` | secret válido até 2028-02-26 |
| **BC** | Business Central | `fabec729-bb7e-48f8-a3cc-4a649ed4ab45` | (do projeto Reconciliação) |

**Tenant ID (mesmo nas duas):** `f41c8222-df66-449c-93b5-c1879e641cb2`

**Secrets:**
- Graph: `<REDACTED_GRAPH_CLIENT_SECRET>`
- BC: em `Projetos/N8N_Cabeceiras/03_Reconciliacao/app/.env` (`BC_CLIENT_SECRET`)

### 4.2 n8n (Railway)

- **URL:** `https://primary-production-0fe7d.up.railway.app`
- **Workflow principal:** `62wyOKnNBy0bnJUw` (31 nodes, ativo)
- **Workflow alertas:** `M2I1eekcsJTi9Av7` (8 nodes, ativo)
- **API key:** guardada em memória `feedback_n8n_workflow_debug.md`

**Credenciais n8n usadas:**
- Anthropic API: `srXSApQJ2OBjvnxL` (Cabeceiras_Orcamentos)
- Gmail: `ANhwrJfsekq0u1K3` (Gmail ACS)

### 4.3 OneDrive (`brandao@cabeceiras.pt`)

- **Drive ID:** `b!fxQjRtmydUqtNaD2rLbsDpw9vi50zupCl2O41a8_kHwqy-IfbKGnQa-AvUPzD_nm`
- **Pasta raiz:** `Cabeceiras-PARTILHA/Dep.  Financeiro/Automacao-tesouraria/Faturas/`
  - ⚠️ `Dep.  Financeiro` tem **2 espaços** (URL-encode `%20%20`)

| Pasta | ID |
|-------|-----|
| `faturas por inserir` | `01JSSR6ZOWI4JQ7ADEZFHZT24VNIWIU42R` |
| `faturas inseridas` | `01JSSR6ZLDQWWQPDGYP5CZJAQFOZUTJGI2` |
| `Guia ou Fatura Assinadas` | `01JSSR6ZO6ALSJKTWMFNDIGGYDVFDREZTO` |

**Excel Log:** `01JSSR6ZOQEYD67CHDWZDYCTBQAOQPLZKA`
- Worksheet: `Log_Faturas`
- Tabela: `Tabela1` (11 colunas A–K)
- Colunas: data_processamento, file_name, nif, nome, numero, data, total, confianca, anomalias, estado, destino

### 4.4 Business Central

- **Environment:** `PROD`
- **Company OData:** `Jorge Brand%C3%A3o Gon%C3%A7alves` (URL-encoded)
- **OData base:** `https://api.businesscentral.dynamics.com/v2.0/[TENANT]/PROD/ODataV4`
- ⚠️ `/api/v2.0/companies` rejeita estas credenciais — usar SEMPRE OData

**WebServices BC usados:**
| WebService | Para quê |
|------------|----------|
| `workflowVendors` | Validar fornecedor por NIF, obter `number` |
| `VendorLedgerEntriesWebService` | Detetar duplicado de fatura por `External_Document_No` |
| `workflowPurchaseDocuments` | Listar POs (Encomendas de Compra) |

---

## 5. RESPOSTAS CONFIRMADAS PELO JORGE

| Pergunta | Resposta |
|----------|----------|
| Fornecedores piloto | Flex2000, Emma, Madimorais |
| Código placeholder ERP | `ME003007` |
| Tolerância de preço vs BC | 0% (Fase 3 — comparação com PO) |
| Tolerância OCR (somatórios) | 0,05€ (Robson decidiu) |
| Centro de custo default | `P1` |
| Pasta para guias assinadas | `Guia ou Fatura Assinadas` (separada) |
| Email anomalias (teste) | `wp@cabeceiras.pt` + `brandao@cabeceiras.pt` |
| Email anomalias (produção) | `compras@` + `encomendas@` + `brandao@` |
| Quem valida no BC | Tesouraria |
| Pergunta 7 (scanner) | **Ainda em aberto** — verificar se já está configurado para OneDrive |

---

## 6. DECISÕES TÉCNICAS IMPORTANTES

1. **Tolerância OCR 0,05€** — para aceitar arredondamentos naturais nos cálculos do Claude
2. **IF node com condição numérica** — `anomalias.length === 0` em vez de boolean (operador `boolean` do n8n IF v2 está bugado, manda tudo para a mesma branch)
3. **Excel via Graph API tabela** (não via n8n Microsoft Excel node) — o node tinha erros de operação `Cannot read properties of undefined (reading 'execute')`
4. **Body em modo `json` com `JSON.parse()`** — `specifyBody: "string"` envia o body como chave de formulário (bug)
5. **SplitInBatches v3** — outputs invertidos: Branch 1 = loop (não Branch 0)
6. **IF v2 branches** — Output 0 = TRUE, Output 1 = FALSE (convenção n8n)
7. **`workflowVendors` chave para fornecedores no BC** — expõe `vatRegistrationNumber` (NIF) + ~120 campos sem precisar publicar nada novo
8. **BC sempre via OData com Company name** — `/api/v2.0/companies` rejeita as credenciais
9. **Normalização do número da fatura** — remove prefixos `FT `/`NC ` antes de filtrar no BC (Claude extrai com prefixo, BC armazena sem)

---

## 7. BUGS RESOLVIDOS DURANTE DESENVOLVIMENTO

| Bug | Solução |
|-----|---------|
| Workflow import: `active` is read-only | Removido `active` antes do POST |
| Workflow import: `tags` is read-only | Removidos `tags`, `staticData`, `shared`, `meta`, `pinData`, `versionId` |
| Resposta com control characters | Usar `decode('utf-8', errors='replace')` |
| Workflow duplicado criado | Apagado o duplicado via DELETE API |
| OneDrive `/users/brandao/drive` falha com client credentials | Usar `/drives/{driveId}` |
| Claude API "model: Field required" | Mudar de `specifyBody: "string"` para `"json"` com `JSON.parse()` |
| Excel Microsoft node "Cannot read properties of undefined" | Substituir por HTTP Request direto à Graph API |
| Excel rows criadas com valores vazios | Adicionar nó `Preparar Log` que constrói payload antes do HTTP |
| IF v2 com `boolean` manda tudo para branch 1 | Trocar para comparação numérica |
| BC `/api/v2.0/companies` rejeita credenciais | Usar OData com `Company('Nome')` |
| "Name already exists" ao mover PDF | Pastas destino limpas pelo Jorge antes de testar |

---

## 8. FICHEIROS NA PASTA DO PROJETO

| Ficheiro | Propósito |
|----------|-----------|
| `relatorio_2026-06-29.md` | **Este relatório** |
| `ESTADO-ATUAL.md` | Snapshot vivo (estado, credenciais, decisões, roadmap) |
| `REUNIAO-JORGE-FASE3.md` | Documento de apoio para reunião sobre Fase 3 |
| `planeamento-projeto.md` | Planeamento original (Robson + André) |
| `product-discovery.md` | Requisitos do cliente (André + Jorge) |
| `technical-note.md` | Nota técnica original |
| `transcript.txt` | Transcript da reunião 25/06 com Jorge |
| `workflow_fase1_ocr.json` | JSON do workflow principal (31 nodes, sincronizado) |
| `workflow_fase6_alertas.json` | JSON do workflow alertas (8 nodes, sincronizado) |
| `Projeto Cabeçeiras [...].pdf` | PDF de apresentação original |
| `06-25 Palestra [...].mp3` | Áudio da reunião 25/06 |

---

## 9. TESTES REALIZADOS

### 29/06/2026 — Teste end-to-end com 3 faturas Flex2000

| Ficheiro | Nº Fatura | Total | Resultado |
|----------|-----------|-------|-----------|
| `31-206437 (1).pdf` | FT 31/206437 | 8.711,35€ | ✅ Extraido |
| `31-206787 (1).pdf` | FT 31/206787 | 3.009,59€ | ✅ Extraido |
| `82-000382 (1).pdf` | NC 82/382 | 5.724,09€ | ✅ Extraido |

**Validações:**
- Claude extraiu NIF corretamente (504663232)
- Validação no BC encontrou: `F00001 | Flex2000- Prod.Flexiveis, SA | blocked=' '`
- Tolerância 0,05€ aceitou pequenas divergências de arredondamento (antes falhava com 0%)
- Excel registou 3 linhas com sucesso (após várias correções de bugs do n8n)
- PDFs movidos para `faturas inseridas/2026/06/`

---

## 10. PRÓXIMOS PASSOS

### Imediato (1-2 dias)
1. **Reunião Robson + Jorge** para Fase 3 — usar `REUNIAO-JORGE-FASE3.md`
2. Decidir: avançar Fase 3 ou parar aqui (deliverable atual já cobre 80% do valor)
3. Confirmar pergunta 7 (scanner físico) com Jorge
4. Confirmar permissões BC API para `purchaseInvoice` (se avançar Fase 3)

### Se Jorge aprovar Fase 3 (2-3 semanas)
1. Implementar inserção em estado "Open" (Gravado)
2. Testes com 5-10 faturas piloto
3. Tesouraria valida durante 2 semanas
4. Massificar para outros fornecedores

### Fase 7 (eventual — meses)
1. Recolher 100+ faturas processadas
2. Analisar padrões de erro por fornecedor
3. Configurar templates específicos
4. Implementar feedback loop

---

## 11. RISCOS E MITIGAÇÕES

| Risco | Probabilidade | Impacto | Mitigação |
|-------|---------------|---------|-----------|
| Claude lê fatura errada | Média | Alto se Fase 3 ativa | Estado "Gravado" + validação Tesouraria |
| n8n Railway cair | Baixa | Alto | Faturas ficam em `faturas por inserir`, retoma sozinho |
| BC API muda | Baixa | Alto | Endpoints estáveis, OData v4 mantém compatibilidade |
| Credenciais expiram | Média (Azure secrets têm validade) | Alto | Graph secret válido até 2028-02-26; BC desconhecido |
| Custos Claude API | Baixa | Médio | ~$0.01 por fatura, ~200 faturas/mês = $2/mês |
| Fornecedor não no BC | Média (esperada) | Baixo | Anomalia detetada, Jorge cria fornecedor |
| Guia assinada não digitalizada | Alta (mudança de processo) | Médio | Anomalia detetada, PDF fica para revisão |

---

## 12. CONTACTOS

| Pessoa | Papel | Contacto |
|--------|-------|----------|
| Robson Advincula | Desenvolvimento + manutenção | `advxautomate@gmail.com` |
| Jorge Brandão | Cliente | `brandao@cabeceiras.pt` |
| André Silva | WisePirates | (project owner) |

---

## 13. CHECKLIST DE GO-LIVE PLENO

Para considerar projeto 100% concluído:

### Técnico
- [x] Fases 1, 2, 4, 5, 6 em produção
- [ ] Fase 3 implementada (depende de reunião)
- [ ] Migrar secrets para env vars do n8n
- [ ] 50+ faturas reais processadas com sucesso

### Cliente (Jorge)
- [ ] Confirmar scanner → OneDrive (pergunta 7)
- [ ] Garantir que `ME003007` existe no BC e gera erro ao registar
- [ ] Confirmar permissões BC API para `purchaseInvoice`
- [ ] Equipa começa a criar Notas de Encomenda no BC

### Operacional (Tesouraria)
- [ ] Treino 1h: abrir log, validar anomalias, criar produtos
- [ ] SLA definido: faturas "Extraido" → validadas em 24h
- [ ] Procedimento de digitalização de guias assinadas
- [ ] Rollback documentado (desativar workflow se houver problemas)

---

## 14. URL ÚTEIS

| Serviço | URL |
|---------|-----|
| n8n workflows | `https://primary-production-0fe7d.up.railway.app` |
| OneDrive Faturas | `https://cabeceiraspt-my.sharepoint.com/personal/brandao_cabeceiras_pt/Documents/Cabeceiras-PARTILHA/Dep.  Financeiro/Automacao-tesouraria/Faturas/` |
| Excel Log | `https://cabeceiraspt-my.sharepoint.com/:x:/r/personal/brandao_cabeceiras_pt/_layouts/15/Doc.aspx?sourcedoc={EF0726D0-E388-47B6-814C-3003A0F5E540}` |
| BC PROD | `https://api.businesscentral.dynamics.com/v2.0/f41c8222-df66-449c-93b5-c1879e641cb2/PROD` |

---

**FIM DO RELATÓRIO — 2026-06-29**

> Este relatório está sincronizado com:
> - Memória persistente: `~/.claude/projects/-Users-robsonadvincula/memory/project_faturas_fornecedores.md`
> - Estado atual: `ESTADO-ATUAL.md`
> - Workflows JSON em produção: `workflow_fase1_ocr.json` + `workflow_fase6_alertas.json`
