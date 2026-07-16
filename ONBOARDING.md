# Onboarding — Sistema de Automação de Faturas de Fornecedores

**Para quem chega novo ao projeto.** Lê-me primeiro, na íntegra, antes de qualquer alteração.

Estimativa: 30 min de leitura + 30 min a pedir acessos → operacional.

---

## 1. Contexto de negócio

**Cliente:** Cabeceiras.pt (Jorge Brandão Gonçalves Unip., Lda.), armazém em Vilela (Paredes).

**Dor:** Jorge recebia dezenas de faturas de fornecedores por dia (papel + PDFs). Um humano tinha que abrir cada uma, escrever à mão a Encomenda de Compra (EC) no Business Central, escolher fornecedor, digitar linhas, aplicar descontos. Horas por dia, erros frequentes.

**Solução:** este workflow lê cada fatura via OCR, valida contra o BC, cria a EC como "rascunho" (`documentType = Order`), e sinaliza no `postingDescription` onde Jorge precisa revisar. Jorge só valida/ajusta em vez de digitar.

**Estado:** produção limitada. Aguardar validação de cada lote pelo Jorge antes de aumentar volume.

**Contactos:**
- Jorge (cliente principal) — decisão de negócio, validação de ECs
- Equipa do Jorge — preenche col D "Codigo ERP" no Excel `Codigos Fornecedores` para novos PKs

Ler para contexto adicional:
- `product-discovery.md` — descoberta inicial de requisitos
- `planeamento-projeto.md` — visão geral e fases

---

## 2. Setup — acessos a pedir

Peça o seguinte ao owner atual (Robson):

### Azure Portal (3 app registrations)
| App | Client ID | Uso |
|---|---|---|
| Graph API | `b8e798a0-6988-47aa-a4a5-0d4d1fe125eb` | OneDrive + Excel |
| BC PROD | `fabec729-bb7e-48f8-a3cc-4a649ed4ab45` | ERP produção |
| BC DEV | `55e26621-e247-4026-9f47-bf17728e77bb` | ERP DEV (só usada em fase inicial) |

Tenant: `f41c8222-df66-449c-93b5-c1879e641cb2`

Pede: convite como colaborador OU os 3 `client_secret` para setar env vars locais/deploy.

### Google Cloud
- Projeto: `cabeceiras-ocr`
- Processor Document AI: `d9230429c35852ce` (region EU)
- Service Account JSON — **NÃO** está no repo. Pedir ficheiro `cabeceiras-ocr-*.json` diretamente.

### Anthropic + OpenAI
- Anthropic key (para Claude Sonnet 4.5) — usada pela credencial n8n `srXSApQJ2OBjvnxL`
- OpenAI key (para GPT-5 Verificar Assinatura + Mapear Items) — credencial n8n `CogELPgsbfrLHzNg`

### n8n Railway
- URL: `https://primary-production-0fe7d.up.railway.app`
- API key + user/pwd UI — pedir
- Workflow ID: `62wyOKnNBy0bnJUw` (Fase 1 OCR, 69 nodes)
- Workflow ID Fase 6: `M2I1eekcsJTi9Av7` (alertas POs atrasadas)

### Business Central
- URL: `https://api.businesscentral.dynamics.com/v2.0/f41c8222-df66-449c-93b5-c1879e641cb2/PROD/ODataV4/Company('Jorge%20Brand%C3%A3o%20Gon%C3%A7alves')`
- Endpoints custom publicados: `workflowPurchaseDocuments`, `workflowPurchaseDocumentLines`, `workflowVendors`, `VendorLedgerEntriesWebService`, `workflowItems`

### OneDrive (pastas + Excels)
| Recurso | ID |
|---|---|
| Pasta `faturas por inserir` | `01JSSR6ZOWI4JQ7ADEZFHZT24VNIWIU42R` |
| Pasta `faturas inseridas` | `01JSSR6ZLDQWWQPDGYP5CZJAQFOZUTJGI2` |
| Pasta `faturas_com_erro` | `01JSSR6ZMVOBS354FCNVDZG2CAXFQFJUR6` |
| Pasta `Guia ou Fatura Assinadas` | `01JSSR6ZO6ALSJKTWMFNDIGGYDVFDREZTO` |
| Excel `Log_Faturas` | `01JSSR6ZOQEYD67CHDWZDYCTBQAOQPLZKA` (tabela `Tabela1`) |
| Excel `Codigos Fornecedores` | `01JSSR6ZOQ4BV4R2JA4JEJJD4BPSVMLY2K` (tabela `MapeamentoItems`) |
| Drive ID | `b!fxQjRtmydUqtNaD2rLbsDpw9vi50zupCl2O41a8_kHwqy-IfbKGnQa-AvUPzD_nm` |

### Railway
- Workspace: `Wise Pirates - PA`
- Projeto: `N8N Cabeceiras`
- Serviço principal: `Primary`
- Env vars críticas: `GRAPH_CLIENT_SECRET`, `BC_PROD_CLIENT_SECRET`, `BC_DEV_CLIENT_SECRET`

### GitHub
- Repo: https://github.com/robsonadvincula-svg/sistema-automacao-faturas (privado)
- Pedir convite como collaborator

---

## 3. Arquitetura

### Fluxo end-to-end

```
Scanner (Epson)
    ↓
OneDrive "faturas por inserir"
    ↓ (poll 3 min via Trigger)
n8n Workflow (62wyOKnNBy0bnJUw)
    │
    ├─ [1] Filtrar Faturas (PDF/JPG/PNG/WEBP) — 1 ficheiro/poll
    ├─ [2] Download PDF do OneDrive
    ├─ [3] Google Document AI → cabeçalho (NIF, total, data, fornecedor)
    ├─ [4] Parse e Validar (normaliza NIFs, datas PT→ISO)
    ├─ [5] Download PDF (2ª vez, para Claude)
    ├─ [6] Preparar Payload Claude + POST Anthropic API
    │       └─ Claude Sonnet 4.5 devolve linhas literais (não inventa)
    ├─ [7] Merge Linhas GPT-5 (sanity check magnitude 5x — rejeita se Σ Claude >>> total OCR)
    ├─ [8] GPT Verificar Assinatura Fatura (só se não veio detetada)
    ├─ [9] Verificar Duplicado Local (Excel Log_Faturas)
    ├─ [10] Verificar Duplicado BC (VendorLedgerEntries)
    ├─ [11] Ler Excel Codigos Fornecedores (dicionário PK → código BC)
    ├─ [12] Verificar Fornecedor BC (por NIF)
    ├─ [13] Avaliar Resultado BC (matchType: NIF | nome exato | nome único distintivo)
    ├─ [14] Verificar EC Existente (evita duplicar Open)
    ├─ [15] Validar Receção (consolida flags)
    ├─ [16] Aplicar Mapeamento Excel (usa helper pegarValidadoGPT5)
    ├─ [17] Preparar EC Header (calcula postingDescription + flags)
    ├─ [18] POST BC "Criar EC Header"
    ├─ [19] Loop Linhas → POST BC "Criar Linha EC"
    ├─ [20] Atualizar Datas EC (PATCH final, refaz postingDescription)
    ├─ [21] Mover PDF → "faturas inseridas/YYYY/MM/"
    └─ [22] Log Sucesso no Excel

Erros:
    → Preparar Log Erro → PATCH linha existente OU POST nova
    → Mover PDF → "faturas_com_erro" (só em erros hard)
    → Email Anomalia (desativado em teste)
```

### Filosofia
1. **Document AI** para cabeçalho (funciona 100%)
2. **Claude Sonnet 4.5** para linhas (extração literal)
3. **Excel** como dicionário PK → código BC (Jorge/equipa preenchem col D)
4. **Match SÓ por PK** (nunca por descrição — evita falsos positivos)
5. **Nunca inventar** items no BC (fallback `ME003007` placeholder + flag `ITEM SEM MATCH`)
6. **Nunca bloquear** — se algo falha, sistema entrega EC com flag, Jorge corrige manualmente
7. **Flags no `postingDescription`** dizem ao Jorge onde revisar

### Custo por fatura
~$0,02 (Document AI + Claude Sonnet 4.5).

---

## 4. Playbook operacional

Comandos prontos para tarefas comuns. Preenche `$TOKEN`, `$GTOKEN`, `$N8N_KEY` com credenciais do teu setup.

### 4.1 Obter tokens

```bash
# BC (PROD)
TOKEN=$(curl -s -X POST "https://login.microsoftonline.com/f41c8222-df66-449c-93b5-c1879e641cb2/oauth2/v2.0/token" \
  -d "grant_type=client_credentials&client_id=fabec729-bb7e-48f8-a3cc-4a649ed4ab45&client_secret=$BC_PROD_SECRET&scope=https://api.businesscentral.dynamics.com/.default" \
  | python3 -c "import sys,json;print(json.load(sys.stdin)['access_token'])")

# Graph (OneDrive/Excel)
GTOKEN=$(curl -s -X POST "https://login.microsoftonline.com/f41c8222-df66-449c-93b5-c1879e641cb2/oauth2/v2.0/token" \
  -d "grant_type=client_credentials&client_id=b8e798a0-6988-47aa-a4a5-0d4d1fe125eb&client_secret=$GRAPH_SECRET&scope=https://graph.microsoft.com/.default" \
  | python3 -c "import sys,json;print(json.load(sys.stdin)['access_token'])")
```

### 4.2 Ver últimas ECs criadas pelo sistema

```bash
BASE="https://api.businesscentral.dynamics.com/v2.0/f41c8222-df66-449c-93b5-c1879e641cb2/PROD/ODataV4/Company('Jorge%20Brand%C3%A3o%20Gon%C3%A7alves')"
curl -s -H "Authorization: Bearer $TOKEN" \
  "$BASE/workflowPurchaseDocuments?\$filter=yourReference%20eq%20'AUTOMA%C3%87%C3%83O'&\$orderby=number%20desc&\$top=10&\$select=number,buyFromVendorName,vendorInvoiceNumber,documentDate,amountIncludingVat,postingDescription"
```

### 4.3 Ver pasta "faturas por inserir"

```bash
DRIVE="b!fxQjRtmydUqtNaD2rLbsDpw9vi50zupCl2O41a8_kHwqy-IfbKGnQa-AvUPzD_nm"
curl -s -H "Authorization: Bearer $GTOKEN" \
  "https://graph.microsoft.com/v1.0/drives/$DRIVE/items/01JSSR6ZOWI4JQ7ADEZFHZT24VNIWIU42R/children"
```

### 4.4 Ver estado + últimas execs do workflow

```bash
N8N_URL="https://primary-production-0fe7d.up.railway.app"
# Estado
curl -s -H "X-N8N-API-KEY: $N8N_KEY" "$N8N_URL/api/v1/workflows/62wyOKnNBy0bnJUw" \
  | python3 -c "import sys,json;print(f'active={json.load(sys.stdin)[\"active\"]}')"

# Últimas execs
curl -s -H "X-N8N-API-KEY: $N8N_KEY" \
  "$N8N_URL/api/v1/executions?workflowId=62wyOKnNBy0bnJUw&limit=5"

# Detalhe de 1 exec (para debug)
curl -s -H "X-N8N-API-KEY: $N8N_KEY" \
  "$N8N_URL/api/v1/executions/{EXEC_ID}?includeData=true"
```

### 4.5 Desativar / reativar workflow

```bash
# Desativar (SEMPRE antes de qualquer alteração)
curl -s -X POST -H "X-N8N-API-KEY: $N8N_KEY" \
  "$N8N_URL/api/v1/workflows/62wyOKnNBy0bnJUw/deactivate"

# Reativar (só após testar e ter confirmação do owner)
curl -s -X POST -H "X-N8N-API-KEY: $N8N_KEY" \
  "$N8N_URL/api/v1/workflows/62wyOKnNBy0bnJUw/activate"
```

### 4.6 Apagar EC no BC + reprocessar

```bash
# Passo 1 — obter systemId + etag
META=$(curl -s -H "Authorization: Bearer $TOKEN" \
  "$BASE/workflowPurchaseDocuments?\$filter=number%20eq%20'EC2602XXX'")
ID=$(echo "$META" | python3 -c "import sys,json;print(json.load(sys.stdin)['value'][0]['id'])")
ETAG=$(echo "$META" | python3 -c "import sys,json;print(json.load(sys.stdin)['value'][0]['@odata.etag'])")

# Passo 2 — DELETE
curl -X DELETE -H "Authorization: Bearer $TOKEN" -H "If-Match: $ETAG" \
  "$BASE/workflowPurchaseDocuments($ID)"

# Passo 3 — mover PDF de "faturas inseridas/YYYY/MM/ECEC2602XXX_*.pdf" para "faturas por inserir/" com nome original
# (usar Graph PATCH com parentReference + name — ver relatório 15/07 secção 9)

# Passo 4 — limpar linha no Log_Faturas Excel (grep por EC no values → DELETE por index)

# Passo 5 — reativar workflow, poll pega em <3min
```

### 4.7 Alterar workflow via API (deploy correto)

```bash
# 1. SEMPRE desativar primeiro
curl -X POST -H "X-N8N-API-KEY: $N8N_KEY" "$N8N_URL/api/v1/workflows/62wyOKnNBy0bnJUw/deactivate"

# 2. Puxar workflow atual
curl -s -H "X-N8N-API-KEY: $N8N_KEY" "$N8N_URL/api/v1/workflows/62wyOKnNBy0bnJUw" > /tmp/wf.json

# 3. Modificar em Python (nunca editar JSON à mão, quebra)
python3 <<'EOF'
import json
wf = json.load(open('/tmp/wf.json'))
for n in wf['nodes']:
    if n['name'] == 'Nome do Node':
        # modificar n['parameters']...
        pass
# PUT precisa apenas destes 4 campos:
payload = {"name": wf["name"], "nodes": wf["nodes"], "connections": wf["connections"], "settings": wf.get("settings", {})}
open('/tmp/wf_put.json', 'w').write(json.dumps(payload, ensure_ascii=False))
EOF

# 4. PUT
curl -X PUT -H "X-N8N-API-KEY: $N8N_KEY" -H "Content-Type: application/json" \
  --data-binary @/tmp/wf_put.json \
  "$N8N_URL/api/v1/workflows/62wyOKnNBy0bnJUw"

# 5. Reativar SÓ após validar sintaxe + testar simulação
```

### 4.8 Rotar client_secret Azure

```bash
# 1. Azure Portal → App registration → Certificates & secrets → New client secret
# 2. Copiar novo secret
# 3. Railway CLI (owner conta com acesso):
cd /path/projeto
railway link --workspace "Wise Pirates - PA" --project "N8N Cabeceiras" --service "Primary"
railway variables --set "GRAPH_CLIENT_SECRET=novo_valor"  # ou BC_PROD_CLIENT_SECRET / BC_DEV_CLIENT_SECRET
# Railway redeploy automático

# 4. Aguardar 60s + testar: curl OAuth com novo secret (deve devolver access_token)
# 5. Azure Portal → apagar secret antigo
```

---

## 5. Gotchas críticos

**Erros já dados. Não repetir.**

### Extração
- **NIF do Jorge `510718531` aparece por engano** em muitas faturas (Ferragens Carreiras, Würth, PLASPUR, etc.) porque Document AI apanha o `Nº de Contribuinte` da caixa cliente. Sistema tem fallback single-match distintivo em `Avaliar Resultado BC`. Se um novo fornecedor único aparecer no BC com esse NIF, verificar bem.
- **Data PT `02.07.2026`** (pontos) quebra BC OData. Tratada em `normalizeDate()` de `Parse e Validar`. Suporta `DD.MM.YYYY`, `DD/MM/YYYY`, `DD-MM-YYYY` e ISO.
- **Separadores PT em quantidades** — Claude pode ler `1 255,000` como 1.255 milhões (real: 1255). Sanity check em `Merge Linhas GPT-5` rejeita se `Σ Claude / total OCR > 5` ou `< 0.2`. Se rejeita, cai em placeholder `ME003007` com total OCR (Jorge desmembra manualmente).
- **Zeros à direita em preços** — Claude lê `0,5880` como `0.588` em vez de `0.58` (bug FSM). Não resolvido; adiado.
- **Regex de prefixos aceites** em `vendorInvoiceNumber`: `FT|FAT|NC|FR|FRA|FC|FS|FE|GT|GS|GR|VD`. Sempre normalizar antes de comparar duplicados.
- **PKs no Excel** `Codigos Fornecedores` têm `\xa0` (non-break space) — normalizar antes de match.

### BC OData
- **NÃO suporta `OR`** entre campos distintos. Fazer 2 queries e merge em memória.
- **Company URL sempre** com `Company('Jorge%20Brand%C3%A3o%20Gon%C3%A7alves')`. `/api/v2.0/companies` rejeita as credenciais.
- **DELETE precisa `If-Match: <etag>`** header. GET primeiro, extrair `@odata.etag`, depois DELETE.
- **postingDescription é reescrito no PATCH final** (node `Atualizar Datas EC`). Alterações ao POST no `Preparar EC Header` são sobrescritas — atualizar ambos.
- **Meilex e Ernesto Romano não imprimem PK na fatura** — sempre `ME003007`, edição manual pelo Jorge. Aceito por design.

### n8n
- **Code nodes bloqueiam HTTP externo** (`fetch`, `$http`, `require('https')`). Fazer HTTP em nodes HTTP dedicados; Code node só processa/transforma.
- **Trigger polling entrega 1 ficheiro por poll** (opção A escolhida). Cada poll = 1 fatura. Ordena alfabeticamente → fatura mais antiga primeiro. Se ficar presa (ex: erro de vendor), bloqueia as outras.
- **`await this.helpers.httpRequest`** funciona em Code node (n8n envolve em async function) apesar de `node --check` local falhar.
- **Nunca alterar workflow com ele ATIVO**. Sempre `POST /deactivate` primeiro. Ver `feedback_parar_workflow_antes_corrigir.md`.
- **Timing:** confirmar `updatedAt` do workflow vs `startedAt` da exec antes de investigar bugs. Um poll pode ter pegado antes do deploy.

### Segurança
- **Nunca hard-code secrets** em bodyParameters, headers ou Code. Usar `{{ $env.NOME_SECRET }}` com env var no Railway. Antes de commit → grep de padrões (`[a-zA-Z0-9]{3}[.~][a-zA-Z0-9_-]{34,}`, `sk-...`, `AIza...`, `gh[pous]_...`, `Bearer ...`).

---

## 6. Ordem de leitura

Recomendada para quem chega novo:

1. **Este ficheiro (`ONBOARDING.md`)** — visão global (30 min)
2. **`README.md`** — sumário + arquitetura + segurança
3. **`product-discovery.md`** — porquê o projeto existe
4. **`planeamento-projeto.md`** — fases originais
5. **`ESTADO-ATUAL.md`** — snapshot do último ponto
6. **`relatorio_2026-07-15.md`** — sessão mais recente (5 bugs + 13 ECs)
7. **`relatorio_2026-07-14.md`** — introdução do Claude Sonnet 4.5
8. **`relatorio_2026-07-09.md`** — 4 fornecedores diferentes testados, decisão "match só por PK"
9. **`relatorio_2026-07-07.md`** — refactor gpt-5 + Responses API + descoberta PDF-nativo vs PDF-scan
10. **Outros relatórios diários** — só se precisar de contexto histórico específico
11. **`PERGUNTAS-JORGE-SEGUNDA-2026-07-06.md`** — perguntas pendentes ao cliente
12. **`REUNIAO-JORGE-FASE3.md`** — notas de reunião presencial
13. **`CHECKLIST-BC-DEV-FIX.md`** — configuração BC DEV

---

## 7. Primeiros passos concretos (dia 1)

1. Pedir todos os acessos da secção 2
2. Ler ONBOARDING + README + product-discovery + planeamento (2h)
3. `git clone https://github.com/robsonadvincula-svg/sistema-automacao-faturas`
4. Instalar Railway CLI (`brew install railway`), fazer `railway login` + link ao serviço
5. Setar env vars locais (`.env`) ou usar `railway run` para injetar
6. Fazer os curls da secção 4 (só GET, sem PUT/DELETE) para confirmar acessos
7. Abrir n8n UI, ver workflow `62wyOKnNBy0bnJUw` (sem ativar)
8. Ver 3 últimas execs — clicar nos nodes um a um, entender o fluxo real
9. Ler relatórios 07-15 → 07-09 (nesta ordem) — cobre 90% dos gotchas

Depois disto: consegue debugar problemas comuns sem partir nada.

---

## 8. Contactos técnicos

- **Owner atual:** Robson Advincula (`robson.advincula@wisepirates.com`)
- **Cliente:** Jorge Brandão — validação final de cada lote de ECs
- **Reporting:** ao Robson via WhatsApp / email

---

## 9. O que este projeto NÃO faz

Para gerir expectativas:

- **Não regista faturas** no BC (não faz "Post"). Cria como Order (rascunho) — Jorge tem que aprovar.
- **Não faz OCR de manuscritos** — só faturas impressas/digitalizadas nítidas.
- **Não substitui PKs incorretos** — se o fornecedor imprime "MP000938" e o BC tem "MP0938", fica `ME003007` até o Excel `Codigos Fornecedores` ser atualizado.
- **Não integra com email** — só OneDrive. Se fornecedor envia por email, alguém tem que guardar o PDF no OneDrive.
- **Não trata guias de remessa isoladas** — só faturas. Fase 4 valida guia assinada como prova de receção, mas não a regista.

---

**Última atualização:** 2026-07-16 (Robson + Claude Sonnet 4.6)
