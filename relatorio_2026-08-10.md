# Relatório — Projeto 21 — Segunda 2026-08-10

**Cliente:** Cabeceiras.pt (Jorge Brandão)
**Workflow n8n:** `62wyOKnNBy0bnJUw` + 5 satélites
**Foco:** Retorno de baixa · investigação de consumo alto · Circuit Breaker + reordenação PDF · descoberta 730 execs falhadas por 1 PDF preso

---

## 1. Contexto

Robson voltou de baixa. Coordenador havia chamado a atenção por consumo acima do normal no sistema. Sessão dedicada a investigar, implementar proteções e alinhar próximos passos.

---

## 2. Balanço da semana 03-09/08 (durante ausência)

**Volume:**
| Dia | ECs criadas |
|-----|-------------|
| Segunda 04/08 | 19 |
| Terça 05/08 | 27 (pico) |
| Quarta 06/08 | 8 |
| Quinta 07/08 | 14 |
| **Total** | **68 ECs** |

Sexta 08 e sábado 09 sem digitalizações. Custo REAL registado no log Excel: **€5,01** em APIs para essas 68 faturas.

Todos os workflows mantiveram-se ativos e a subscription do Graph foi renovada automaticamente (segunda 07h). Zero intervenção manual necessária durante a semana.

---

## 3. Investigação — 730 execuções longas em 03-04/08

**Consumo alto identificado:**

| Dia | Execs longas (>5s) | ECs criadas | Custo estimado |
|-----|--------------------|-----------  |----------------|
| **DOMINGO 03/08** | **513** | **0** | **~€38** |
| **SEGUNDA 04/08** | **217** | 19 | ~€15 |
| Quarta 05/08 | 27 | 27 | ~€2 (normal) |

**Causa raiz (descoberta em amostra de 30 execuções):**
- **1 PDF único** da Cadeiras Pinto, Lda ficou preso na pasta `faturas por inserir/`
- Doc AI não conseguia extrair `numero_fatura`
- Sistema chamava Doc AI + Claude, falhava a criar EC, PDF ficava na pasta
- Cada notificação subsequente do OneDrive re-disparava o processamento **do mesmo PDF**
- **730 execuções × €0.075 = ~€55 desperdiçados em 2 dias por 1 PDF**

**Padrão da amostra 30/30:**
- 30/30 do mesmo fornecedor (Cadeiras Pinto)
- 30/30 mesma anomalia ("Sem numero de fatura")
- 30/30 sem EC criada
- Apenas 1 nome de PDF distinto (mesmo ficheiro processado repetidamente)

---

## 4. Documentos descobertos no repo `cabeceiras-geral`

Nathielle e Flow OS (Wise Pirates) fizeram auditorias independentes ao sistema em 06/08. Achados relevantes:

- **Achado 1 (crítico):** duas execuções podem processar a mesma fatura em paralelo (pode duplicar ECs)
- **Achado 2 (crítico):** se o arquivo do PDF falhar, criam-se ECs em cadeia
- **Achado 3-8:** secrets em backups, webhook sem auth, retries zero, 13 nodes mortos, alertas sem detalhe do erro

**Documento `INCIDENTE-2026-08-05` — outro incidente independente:**
- Claude alucinou ano ("2025" em vez de "2026") em fatura Elastron
- BC criou EC2503869 na série do ano errado
- 3 ECs afetadas: EC2503866, EC2503867, EC2503868 (ainda não tratadas)
- Fix em duas camadas já aplicado (05/08) no prompt e no Merge

---

## 5. Fixes implementados hoje

### 5.1 Circuit Breaker (proteção contra sangramento)

Novo node `Circuit Breaker Check` no workflow principal. Após **5 falhas consecutivas sem criar EC** → workflow auto-desativa + email urgente para wp@ + brandao@ + robson.advincula@.

**Cobertura:** o incidente de 03/08 teria parado ao 5º PDF (~€0.40) em vez de gastar €55.

**Novos nodes adicionados:**
- `Circuit Breaker Check` (Code) — usa `$getWorkflowStaticData` para contador
- `Deve Pausar?` (IF)
- `Desativar Workflow (Circuit Breaker)` (HTTP → n8n API)
- `Email Circuit Breaker` (Gmail)

### 5.2 Configuração Railway — N8N_API_KEY

Env var `N8N_API_KEY` adicionada em **4 serviços** (Primary + Worker 1/2/4) via Railway CLI. Redeploy dos 4 serviços feito. Webhook validado com 200 após restart.

Sem esta env var, o Circuit Breaker não conseguia chamar a API do n8n para se auto-desativar.

### 5.3 Teste real do Circuit Breaker

Injectei temporariamente `circuit_should_pause=true` no node, coloquei PDF de teste, aguardei processamento:
- ✅ Circuit Breaker Check disparou
- ✅ IF `Deve Pausar?` avaliou true
- ✅ HTTP call desativou workflow (`active=False`)
- ✅ Email Circuit Breaker enviado

Após validação, restaurei código original + reactivei workflow + limpei artefactos de teste.

### 5.4 Move PDF cedo (paralelo com Loop Linhas)

**Antes:** `Atualizar Datas EC → Preparar Destino → ... → Mover PDF → Preparar Log` (T+28s)

**Depois:** `Verificar EC Criado` bifurca em paralelo para:
- Branch A: `EC Header OK? → Loop Linhas EC → Atualizar Datas EC → Preparar Log`
- Branch B: `Preparar Destino → Criar Pasta Ano → Criar Pasta Mês → Obter ID → Mover PDF` (fim)

**Teste real (exec 96041, EC2602761):**
- Verificar EC Criado: T+23.2s
- Mover PDF: **T+26.9s** (em paralelo com Loop Linhas EC)
- Antes seria só ~T+28-29s

**Impacto:** reduz janela de risco onde outra notificação webhook podia disparar 2ª execução para o mesmo PDF.

### 5.5 Monitor PL000001 (email já com Robson em CC)

Adicionado `robson.advincula@wisepirates.com` aos destinatários do email do Monitor PL000001.

---

## 6. Discussão sobre complexidade do sistema

Robson identificou **preocupação real:** 103 nodes distribuídos por 6 workflows = grande superfície de erro.

Propostas apresentadas (não implementadas):

**Passo 1 — Remover 13 nodes mortos do Wf 21 principal (identificados pelo audit).**
Zero risco funcional (não executam). 76 → 63 nodes.

**Passo 2 — Consolidar 3 workflows periféricos em 1 "Manutenção Diária".**
Renovar Subscription (4) + Monitor PL000001 (7) + Fase 6 Alertas POs (8) → 1 workflow ~15 nodes. 6 → 4 workflows.

**Decisão Robson:** **hoje não fazer nada.** Deixar sistema correr esta semana. Se estabilidade for OK, retomar plano de simplificação depois.

---

## 7. Estado final

**Workflows ativos (6):**
| Workflow | ID | Nodes |
|----------|-----|-------|
| 21 — Faturas Fornecedores — Fase 1 OCR | `62wyOKnNBy0bnJUw` | 76 |
| Cabeceiras Faturas — OneDrive Webhook | `v8yIHOn4wGYHKhq7` | 5 |
| Cabeceiras Faturas — Renovar Subscription | `VT15XtXEohrnEkRi` | 4 |
| Cabeceiras Faturas — Monitor PL000001 | `a9qKfkCqECA060Rk` | 7 |
| 🚨 Error Monitor | `CbwykycXHKv2VbTc` | 3 |
| Fase 6 Alertas POs | `M2I1eekcsJTi9Av7` | 8 |

**Subscription Graph:** ativa até 01/09/2026 (renovação automática segunda 07h)
**Inbox faturas por inserir:** vazia
**Última EC:** EC2602748 (Ernesto Romano, 07/08)

---

## 8. Pendências futuras (backlog, sem urgência)

- **Simplificação (Passo 1+2 da secção 6)** — quando Robson autorizar
- **3 ECs afetadas pelo incidente de 05/08 (EC2503866, EC2503867, EC2503868)** — ainda em série do ano errado; podem ficar como estão ou serem apagadas e recriadas
- **Dedup por hash SHA256** — cobriria 100% dos casos como o de 03/08. Só se voltar a acontecer.
- **Confirmar emails de compras com Jorge** para ativar routing Fase 4.5 (pendente há semanas)
- **Cadastrar produtos Excel para fornecedores com muitos placeholders** (Topbois, FSM, Nogueira & Ribeiro)

---

## 9. Aprendizagens do dia

- **1 PDF sozinho pode custar €55 em 48h** se ficar preso em loop de reprocessamento. Circuit Breaker resolve.
- **Move PDF tarde = janela para duplicação de execuções paralelas.** Reordenação para paralelo com Loop Linhas EC elimina o problema.
- **Env vars Railway não são aplicadas em runtime até redeploy.** `--skip-deploys` no `railway variables --set` guarda a var mas não afeta processos em execução.
- **Wise Pirates fez auditoria independente do sistema em 06/08** — documentação valiosa que devemos consultar antes de fazer alterações. Reduz risco de retrabalho.
