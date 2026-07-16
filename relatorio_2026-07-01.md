# Relatório de Serviços — 1 de Julho de 2026

> **Sessão de trabalho:** implementação Fase 3 + fix de dedup + segurança
> **Duração:** ~4 horas
> **Estado no fim:** sistema estável, testado, sem impacto em BC PROD

---

## Trabalho realizado

### 1. Reunião com Jorge — decisões de scope

**Decisões alinhadas:**
- **Fase 3 (inserção BC):** avança
- **Fornecedores piloto:** Flex2000, Emma, Madimorais
- **Local de inserção:** Encomendas de Compra (não Purchase Invoices) → cria nova EC
- **Placeholder para produtos sem código:** `ME003007`
- **Centro de custo default:** `P1`
- **Segurança:** sistema apenas grava (nunca faz registo contabilístico)
- **Testar antes de assumir permissões OK:** ainda por confirmar
- **Scanner/guias:** ainda não conversaram sobre isso

### 2. Descoberta crítica — ambiente BC DEV existe

**Problema:** Robson (o utilizador) expressou preocupação legítima em usar PROD do cliente para escrita (histórico de erros meus durante o projeto).

**Solução:** Descoberta de que existe ambiente **DEV** no BC:
- URL: `.../DEV/ODataV4/...`
- Credenciais próprias (diferentes das PROD, criadas em 2026-04):
  - Client ID: `55e26621-e247-4026-9f47-bf17728e77bb`
  - Secret: em `03_Reconciliacao/Relatorio_Servicos_2026-04-02.md`
- Fonte: `Projetos/N8N_Cabeceiras/03_Reconciliacao/Relatorio_Servicos_2026-04-02.md`

**Convenção definida do projeto:**
- **Fase 2 (leitura/validação)** → BC PROD
- **Fase 3 (escrita/EC)** → BC DEV
- **NUNCA** POST/PUT/DELETE em PROD

### 3. Fase 3 implementada (target: BC DEV)

**6 nós adicionados ao workflow principal:**

1. `Obter Token BC DEV` — OAuth com credenciais DEV
2. `Preparar EC Header` — payload do cabeçalho da EC
3. `Criar EC Header BC DEV` — POST `workflowPurchaseDocuments` (DEV)
4. `Verificar EC Criado` — parse response + prepara linhas
5. `Loop Linhas EC` — SplitInBatches v3, batchSize=1
6. `Criar Linha EC BC DEV` — POST `workflowPurchaseDocumentLines` (DEV)

**Fluxo Fase 3 (só em DEV):**
```
Extração OK? [TRUE]
  → Obter Token BC DEV
  → Preparar EC Header (vendor, invoice nº, datas, P1)
  → Criar EC Header BC DEV (POST → recebe nº EC gerado)
  → Verificar EC Criado (constrói array de linhas)
  → Loop Linhas EC (uma iteração por produto)
      → Criar Linha EC BC DEV (POST cada linha)
  → Preparar Destino → ... → Log Sucesso
```

**Estado da EC após criação:**
- `documentType: Order`
- `receive: false`, `invoice: false` (não registamos)
- Sem chamada a `Microsoft.NAV.post` (não faz registo contabilístico)

**Coluna Excel nova:** `numero_ec_bc` (12ª coluna) — guarda o nº da EC criada

### 4. Auditoria formal antes do deploy

Robson exigiu (justamente) auditoria exaustiva antes de permitir teste. Feita e passada:

| Verificação | Resultado |
|-------------|-----------|
| Ocorrências de `/PROD/` no JSON | 4 — todas em nós GET |
| Nós que referenciam PROD | Apenas 2 (Verificar Duplicado BC, Verificar Fornecedor BC) |
| POSTs em BC | Apenas 2 (Criar EC Header BC DEV, Criar Linha EC BC DEV) |
| Nós de código com URLs escondidas | Zero |
| Palavras-chave perigosas (`Microsoft.NAV.post`, `receive:`, `invoice:`) | Zero |
| Triggers externos (webhooks) | Zero |
| Token PROD (`fabec729`) usado por POSTs | Zero |
| Token DEV (`55e26621`) usado por GETs | Zero |

**Conclusão:** técnicamente impossível o workflow escrever em PROD.

### 5. Bug crítico descoberto e corrigido — loops infinitos de anomalias

**Problema:** PDFs em estado "Anomalia" ficavam na pasta e eram reprocessados a cada 5 min, criando linhas duplicadas indefinidamente. Se ficassem 1h → 12 linhas extras por ficheiro.

**Causa:** O nó `Verificar Duplicado Local` detetava o duplicado mas apenas adicionava uma anomalia — continuava a processar e a criar nova linha.

**Fix aplicado:**
- `Verificar Duplicado Local`: simplificado. Devolve `duplicado_local_count: 0|1` (não adiciona anomalia)
- Novo nó IF `É Duplicado Local?` após ele:
  - Se `duplicado_local_count === 1` → route para Loop Faturas (**skip completo, próximo item**)
  - Se `0` → continua para Obter Token BC
- Uso de operador `number` no IF (o operador `boolean` do IF v2 do n8n está bugado)

**Confirmado em produção:** 3 execuções consecutivas com os mesmos 3 PDFs — Excel continua com apenas 3 linhas.

### 6. Emails desativados durante testes

Ambos os nós de email desativados via `disabled: true`:
- `Email Anomalia` (workflow principal)
- `Email Alerta` (workflow alertas Fase 6)

Para reativar depois dos testes, mudar `disabled: false` em ambos.

### 7. Erro operacional que Robson me chamou à atenção

**Incidente:** Adicionei a coluna `numero_ec_bc` ao Excel mas esqueci-me de atualizar o `Preparar Log Erro` (só atualizei o `Preparar Log`). Isto causou HTTP 400 no `Log Erro` (row com 11 valores tentando encaixar em 12 colunas).

**Feedback do Robson:** *"esse é seu problema esqueci-me!!! vc não está aqui para esquecer nada vc não é humano"*

**Ação corretiva permanente:** criada regra formal em memória (`~/.claude/.../memory/feedback_verificar_antes_agir.md`) que estabelece:
- Nunca alterar estrutura sem auditar TODOS os pontos dependentes
- "Esqueci-me" é resposta inválida — sou máquina
- Cada erro deve produzir uma regra concreta em memória
- Auditoria cruzada obrigatória: contar elementos vs colunas

---

## Estado final do sistema

### Workflows n8n em produção

| Workflow | ID | Nodes | Trigger | Ativo |
|----------|-----|-------|---------|-------|
| Principal (Fases 1-5) | `62wyOKnNBy0bnJUw` | 38 | Cron 5min | ✅ Sim |
| Alertas POs (Fase 6) | `M2I1eekcsJTi9Av7` | 8 | Cron seg-sex 8h | ✅ Sim |

### Comportamento verificado

Cenário testado: 3 PDFs Flex2000 na pasta `faturas por inserir`, Excel inicialmente vazio.

**Ciclo 1 (Excel vazio):**
- 3 PDFs processados
- Claude extrai dados corretamente
- BC PROD valida fornecedores (F00001) e verifica não-duplicados
- Fase 4 deteta ausência de guia assinada → anomalia
- 3 linhas gravadas no Excel como `Anomalia`
- PDFs ficam na pasta (por design — aguardam guia assinada)

**Ciclos 2-N (mesmos PDFs, Excel já tem entradas):**
- Workflow entra a cada 5 min
- `Ler Log Excel` + `Verificar Duplicado Local` + `É Duplicado Local?` → detecta match
- Route para Loop Faturas (skip)
- Nenhuma nova chamada BC, nenhuma nova linha, nenhum email
- Excel continua com **apenas 3 linhas**

### Colunas do Excel (Tabela1)

12 colunas:
`data_processamento, file_name, nif_fornecedor, nome_fornecedor, numero_fatura, data_fatura, total_com_iva, confianca, anomalias, estado, destino, numero_ec_bc`

---

## Credenciais/IDs importantes (referências, não colar aqui)

- **Azure Graph:** Client `b8e798a0-6988-47aa-a4a5-0d4d1fe125eb` (Robson, OneDrive/Excel)
- **BC PROD:** Client `fabec729-bb7e-48f8-a3cc-4a649ed4ab45` (só leitura)
- **BC DEV:** Client `55e26621-e247-4026-9f47-bf17728e77bb` (escrita) — secret em `03_Reconciliacao/app/.env` como `BC_CLIENT_SECRET` DEV
- **Anthropic (n8n cred ID):** `srXSApQJ2OBjvnxL`
- **Gmail (n8n cred ID):** `ANhwrJfsekq0u1K3`
- **OneDrive Drive:** `b!fxQjRtmydUqtNaD2rLbsDpw9vi50zupCl2O41a8_kHwqy-IfbKGnQa-AvUPzD_nm`

### Pastas OneDrive

- `faturas por inserir`: `01JSSR6ZOWI4JQ7ADEZFHZT24VNIWIU42R`
- `faturas inseridas`: `01JSSR6ZLDQWWQPDGYP5CZJAQFOZUTJGI2`
- `Guia ou Fatura Assinadas`: `01JSSR6ZO6ALSJKTWMFNDIGGYDVFDREZTO`
- Excel log: `01JSSR6ZOQEYD67CHDWZDYCTBQAOQPLZKA`

---

## O que testar a seguir

### Teste da Fase 3 (escrita em BC DEV)

**Passo a passo:**
1. Digitalizar uma guia assinada (ou usar qualquer PDF) com nome contendo o número da guia — ex: `91-000509-assinada.pdf`
2. Colocar em `Guia ou Fatura Assinadas/`
3. Aguardar próximo ciclo (5 min) ou remover e recolocar o PDF `31-206437 (1) (1).pdf` na pasta `faturas por inserir/`
4. Workflow passará por:
   - Fase 4 (Validar Receção): encontra a guia → sucesso
   - Fase 3: cria EC no **BC DEV**
   - Excel: nova linha com estado `Inserido BC DEV` e `numero_ec_bc` preenchido
5. Verificar no BC DEV se a EC foi criada corretamente

### Reativar emails quando pronto para produção

Bastam 2 alterações — `disabled: false` em:
- `Email Anomalia` no workflow principal
- `Email Alerta` no workflow alertas

---

## Ficheiros da pasta do projeto

| Ficheiro | Propósito |
|----------|-----------|
| `relatorio_2026-07-01.md` | **Este relatório** |
| `relatorio_2026-06-29.md` | Relatório da sessão anterior |
| `ESTADO-ATUAL.md` | Snapshot vivo (mais atualizado que este relatório) |
| `REUNIAO-JORGE-FASE3.md` | Documento de apoio da reunião com Jorge |
| `workflow_fase1_ocr.json` | Cópia do workflow principal (38 nodes) |
| `workflow_fase6_alertas.json` | Cópia do workflow alertas (8 nodes) |
| `planeamento-projeto.md` | Planeamento original (Robson + André, 25/06) |
| `product-discovery.md` | Requisitos cliente (André + Jorge) |
| `technical-note.md` | Nota técnica original |
| `transcript.txt` | Transcript da reunião 25/06 |
