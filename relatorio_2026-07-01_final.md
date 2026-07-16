# Relatório Final — 1 de Julho de 2026

> Sessão longa de trabalho. Sistema em teste PROD, várias iterações de correção.
> **Estado no fim:** workflow com Fases 1-6 + Fase 3 completa (com mapping items), **DESATIVADO** aguardando decisão do Jorge sobre EC2602130.

---

## 1. RESUMO EXECUTIVO

Hoje foi feita a implementação da **Fase 3 (inserção BC)** com todas as suas ramificações e sofremos vários incidentes de aprendizagem. O sistema chegou a criar 2 ECs no BC PROD (EC2602129 corrigida pelo Jorge, EC2602130 pendente). Foram identificados 5 problemas e todos foram corrigidos ao longo do dia.

**No fim do dia o workflow está:**
- ✅ Deployed com 51 nodes + todas as correções
- ✅ Desativado (nenhuma execução automática)
- ⏸️ Aguardando Jorge apagar EC2602130 e responder sobre mapping

---

## 2. TRABALHO REALIZADO

### 2.1 Reunião de Jorge (manhã)
- Confirmou: Fase 3 avança, empresas piloto Flex2000/Emma/Madimorais
- Confirmou: inserção em Encomendas de Compras (não Purchase Invoices)
- Confirmou: apenas grava, sem registo contabilístico
- Confirmou: `ME003007` só como fallback quando produto não existe

### 2.2 Descoberta do ambiente DEV
- Robson recusou (justamente) testar diretamente em PROD
- Encontradas credenciais DEV noutro projeto (Reconciliação)
- **Client ID DEV:** `55e26621-e247-4026-9f47-bf17728e77bb`
- Descoberta: BC DEV com série `C_EC` mal configurada (falta Nº Inicial)
- Sem admin do BC disponível → migração para teste em PROD

### 2.3 Teste em PROD com salvaguardas
- Alterado workflow de DEV para PROD apenas para escrita
- Adicionadas 9 verificações de auditoria antes do deploy
- **EC2602129 CRIADA COM SUCESSO** — primeira fatura automática no BC do cliente
- Jorge corrigiu manualmente 2 linhas (ME003259, ME003260 em vez de ME003007)

### 2.4 Bugs encontrados e correções

| # | Problema | Fix |
|---|----------|-----|
| 1 | Emails reativados por overwrite de cache | Regra em memória: sempre pull LIVE antes de PUT |
| 2 | Row length no `Log Erro` (12 vs 11) | Row count auditado em todos os log nodes |
| 3 | `data_fatura undefined` no `Preparar Log` | `.first()` em vez de `.item.json` |
| 4 | `data_fatura undefined` no `Preparar Destino` | Idem (bug persistiu porque não auditei os 2) |
| 5 | Datas na EC ficaram com "hoje" em vez de invoice date | BC ignora datas no POST → PATCH após POST |
| 6 | Todas as linhas com `ME003007` (Claude conservador) | Prompt binário + validação anti-hallucination |
| 7 | Segundo cycle criava EC duplicada | Verificar EC Existente BC (query workflowPurchaseDocuments) |

### 2.5 Item Mapping — arquitetura decidida

Ficou claro que:
- **BC é fonte de verdade** — não faz sentido tabela paralela
- **Claude é único caminho automático** para matching (BC não tem info suficiente)
- **Prompt binário** (existe/não existe) melhor que threshold de confiança
- Sistema **valida** que o código sugerido pelo Claude existe REALMENTE no catálogo (anti-hallucination)

### 2.6 Feedback direto do utilizador (aprendizagens minhas)
- *"Vc não está aqui para esquecer nada, vc não é humano"* → regra em memória
- *"Não podemos alterar a rotina delas — a automação é para ajudar não criar mais trabalho"* → sistema aprende transparente
- *"Estamos no PROD do cliente, não podemos vacilar"* → auditorias mais completas
- Registei todas as regras em `feedback_verificar_antes_agir.md` para nunca mais repetir

---

## 3. ESTADO FINAL DO SISTEMA

### 3.1 Workflow principal (`62wyOKnNBy0bnJUw`)

**51 nodes** | **DESATIVADO** | Última alteração: 2026-07-01

**Fluxo completo:**
```
Trigger 5min → Obter Token Graph → Listar Ficheiros OneDrive
  ↓ (só PDFs)
Loop Faturas → Download PDF → Preparar Base64 → Preparar Payload Claude
  ↓
Claude: Extrair Fatura → Parse e Validar
  ↓
Ler Log Excel → Verificar Duplicado Local → É Duplicado Local?
  ├── SIM → Loop (skip)
  └── NÃO ↓
Obter Token BC → Verificar Duplicado BC → Verificar Fornecedor BC → Avaliar Resultado BC
  ↓
Verificar EC Existente BC → Avaliar EC Existente
  ↓
Listar Guias Assinadas → Validar Receção → Precisa Verificar Guia Claude?
  ├── SIM → Download Guia → Base64 → Claude Verificar Assinatura → Consolidar
  └── NÃO ↓
Extração OK?
  ├── FALSE → Preparar Log Erro → Log Erro (+ Email desativado)
  └── TRUE ↓
Obter Token BC DEV → Preparar EC Header
  ↓
Criar EC Header BC PROD (POST) ← Fase 3 !
  ↓
Atualizar Datas EC (PATCH) ← Fix datas
  ↓
Listar Items Fornecedor BC → Preparar Payload Claude Items → Claude Mapear Items → Aplicar Mapeamento
  ↓
Verificar EC Criado → EC Header OK?
  ├── FALSE → Preparar Log Erro
  └── TRUE ↓
Loop Linhas EC → Criar Linha EC BC PROD (POST) ← Fase 3 !
  ↓ (done)
Preparar Destino → Criar Pasta Ano → Criar Pasta Mês → Obter ID Pasta Mês → Mover PDF
  ↓
Preparar Log → Log Sucesso
  ↓
Loop Faturas
```

### 3.2 Escritas em BC PROD

Exatamente 3 operações:
1. **POST** `workflowPurchaseDocuments` — cria EC
2. **PATCH** `workflowPurchaseDocuments(id)` — atualiza datas
3. **POST** `workflowPurchaseDocumentLines` — cria linhas

### 3.3 Emails

`Email Anomalia` e `Email Alerta` (Fase 6) — **desativados** durante testes

### 3.4 EC criadas no BC PROD hoje

| EC | Estado | Notas |
|----|--------|-------|
| **EC2602129** | ✅ Apagada por Jorge (após correção manual de 2 linhas) | Primeiro teste bem sucedido |
| **EC2602130** | ⏸️ Pendente Jorge apagar | Criada por erro (workflow reativado), 5 linhas com ME003007 |

---

## 4. PENDENTES PARA AMANHÃ

### 4.1 Ações do Jorge
- [ ] Apagar EC2602130 do BC PROD
- [ ] Confirmar mensagem sobre datas (aparecer data da fatura como Data Registo — provavelmente resolvido pelo PATCH que adicionamos)

### 4.2 Testes a fazer amanhã
- [ ] Reprocessar FT_31-206437 com workflow atualizado
- [ ] Verificar se códigos BC são identificados (ME003259 em vez de ME003007)
- [ ] Verificar se datas ficam corretas (2026-06-11 em vez de hoje)
- [ ] Se OK, testar com fatura diferente (novo produto → deve ir para ME003007)

### 4.3 Ações do meu lado (dependendo do resultado do teste)
- [ ] Reativar emails quando confiantes
- [ ] Documentar processo para Tesouraria (como validar ECs)
- [ ] Considerar cache de mappings validados (se muitos matches Claude)

---

## 5. LIÇÕES APRENDIDAS (registadas em memória permanente)

### 5.1 `feedback_verificar_antes_agir.md`
- Nunca fazer alteração sem auditar TODOS os pontos dependentes
- "Esqueci-me" não é resposta válida
- Cada erro produz uma regra concreta

### 5.2 Sobre caches locais
- **PROIBIDO** usar `/tmp/wf_*.json` como base de PUT
- Sempre pull do n8n LIVE antes de qualquer modificação

### 5.3 Sobre auditoria
- Auditoria de segurança ≠ auditoria de correção
- Data continuity precisa ser verificada
- Se um Loop é introduzido no meio, $json a jusante pode ficar incorreto

### 5.4 Sobre BC No. Series
- Configuração de séries pode falhar ao criar novo documento (falta Starting No.)
- Nº Inicial e Nº Final devem ter mesmo comprimento
- Documentado em `reference_bc_no_series.md`

---

## 6. CREDENCIAIS E IDS (para continuidade)

### Azure
- **Tenant:** `f41c8222-df66-449c-93b5-c1879e641cb2`
- **Graph app:** `b8e798a0-6988-47aa-a4a5-0d4d1fe125eb`
- **BC PROD app:** `fabec729-bb7e-48f8-a3cc-4a649ed4ab45` (secret em `03_Reconciliacao/app/.env`)
- **BC DEV app:** `55e26621-e247-4026-9f47-bf17728e77bb` (secret guardado em memória)

### n8n
- **URL:** `https://primary-production-0fe7d.up.railway.app`
- **Workflow principal:** `62wyOKnNBy0bnJUw` — 51 nodes — desativado
- **Workflow alertas Fase 6:** `M2I1eekcsJTi9Av7` — 8 nodes

### OneDrive
- **Drive ID:** `b!fxQjRtmydUqtNaD2rLbsDpw9vi50zupCl2O41a8_kHwqy-IfbKGnQa-AvUPzD_nm`
- **faturas por inserir:** `01JSSR6ZOWI4JQ7ADEZFHZT24VNIWIU42R`
- **faturas inseridas:** `01JSSR6ZLDQWWQPDGYP5CZJAQFOZUTJGI2`
- **Guia ou Fatura Assinadas:** `01JSSR6ZO6ALSJKTWMFNDIGGYDVFDREZTO`
- **Excel Log:** `01JSSR6ZOQEYD67CHDWZDYCTBQAOQPLZKA` (Tabela1, 12 colunas)

### Business Central
- **Environment PROD:** `PROD`
- **Company:** `Jorge Brand%C3%A3o Gon%C3%A7alves`
- **Endpoints usados:** workflowVendors, workflowItems, workflowPurchaseDocuments, workflowPurchaseDocumentLines, VendorLedgerEntriesWebService

---

## 7. COMO CONTINUAR AMANHÃ

1. Abrir esta pasta do projeto
2. Ler `ESTADO-ATUAL.md` para snapshot rápido
3. Se Jorge apagou EC2602130 → reativar workflow para testar
4. Se ainda não → aguardar confirmação
5. Testar com fatura FT_31-206437 novamente
6. Verificar os 3 pontos-chave: códigos corretos, datas corretas, sem duplicados

**Continuação natural do trabalho na próxima sessão.**
