# Relatório — Projeto 21 — Quarta 2026-08-12

**Cliente:** Cabeceiras.pt (Jorge Brandão — em férias)
**Workflow n8n:** `62wyOKnNBy0bnJUw` + 6 satélites
**Foco:** Pedido do Jorge (nº EC no postingDescription) · investigação profunda de execuções · Auto Schedule 8h-18h

---

## 1. Pedido do Jorge — número da EC no postingDescription

Jorge enviou imagem via WhatsApp pedindo que na **página Movimentos Fornecedores** do BC, a coluna Descrição inclua o número da EC.

**Antes:** `AUTOMAÇÃO` (ou `AUTOMAÇÃO - ATENÇÃO VALORES`)
**Agora:** `AUTOMAÇÃO EC2602XXX` (ou `AUTOMAÇÃO EC2602XXX - ATENÇÃO VALORES`)

**Fix aplicado no node `Atualizar Datas EC` (jsonBody):**
```js
let ecNum=$('Criar EC Header BC PROD').first().json.number||'';
let f=ecNum?'AUTOMAÇÃO '+ecNum:'AUTOMAÇÃO';
// depois adiciona flags: ATENÇÃO VALORES / ITEM SEM MATCH / VERIFICAR FORNECEDOR / VERIFICAR DATA
```

**Testado com EC2602780 (Meilex duplicado):**
- `postingDescription: 'AUTOMAÇÃO EC2602780'` ✅
- Após validação, EC2602780 apagada + log limpo + PDF de teste apagado

Todas as ECs criadas a partir de agora vão ter o número no postingDescription. Jorge vai ver o nº da EC directamente na página Movimentos Fornecedores.

---

## 2. Alerta por email — throttling Microsoft Graph

**12:00 UTC (13h Portugal)** — Error Monitor enviou alerta:
```
Workflow: 21 — Faturas Fornecedores — Fase 1 OCR
Execution ID: 98917
Node: Listar Ficheiros OneDrive
Error: "The request has been throttled" (HTTP 429)
```

Destinatários do email:
- andreia.saraiva@wisepirates.com
- nathielle.gouvea@wisepirates.com
- robson.advincula@wisepirates.com

Sistema recuperou sozinho em ~30 segundos. Nenhuma ação necessária. Foi limitação transitória do Microsoft Graph, não erro no código.

---

## 3. Investigação profunda das execuções — descoberta

Robson foi chamado à atenção pelo coordenador por consumo excessivo (semana passada). Investigação hoje mostrou:

**Hoje 12/08 — 549 execuções totais no workflow Principal:**
- **17 execuções longas** (processaram PDF real) → custo ~€1,35 em APIs
- **531 execuções curtas** (0.4s, webhook OneDrive vazio) → custo €0
- **1 erro** (throttling Graph às 13h)

**Descoberta importante:** As 531 execs curtas vinham do **webhook Microsoft Graph** (subscription drive-inteiro). Cada mudança no drive (log Excel escrito, ficheiros movidos, indexação) dispara notificação → n8n arranca workflow → sai em 0.4s por não haver PDF.

**Custo real dessas 531 execs:** €0 (não chamavam APIs pagas)
**Ruído:** visível nos logs, alarmava sem razão

---

## 4. Fixes de arquitetura aplicados

### 4.1 Filtro pré-check no Webhook Handler

**Antes:** cada notificação Graph → chama Principal.
**Agora:** Webhook Handler faz check inteligente antes:
```
Webhook → É handshake? → Sim: 200
        → Não: 202 → Obter Token Graph → Listar Inbox
                   → Tem Fatura Nova? → SIM: chamar Principal
                                     → NÃO: fim (não arranca Principal)
```

Reduz execuções no Principal de ~548/dia → **~17/dia** (só quando há PDF real).

### 4.2 Auto Schedule 08h ON / 18h OFF (novo workflow)

**Motivação:** Robson quer garantir zero execuções fora do horário útil.

**Criado workflow** `zNYDigFcKsRy9o3w` "Cabeceiras Faturas — Auto Schedule":
- **08:00 Seg-Sáb** → API POST `/activate` em Principal + Webhook
- **18:00 Seg-Sáb** → API POST `/deactivate` em Principal + Webhook

Fora deste horário (18h-8h + domingo): ambos workflows **inativos**. Zero risco de gasto por bug.

**PDFs digitalizados fora do horário** ficam na pasta até 8h do dia útil seguinte (aceitável — não é urgente).

**Trade-off aceite:** Se equipa digitalizar às 19h ou domingo, latência = até próximo 08h.

**Manual override aplicado hoje 18:03:** ambos workflows desativados manualmente (fora do primeiro ciclo automático). Amanhã (quinta 13/08) o Auto Schedule vai reactivar às 08:00.

---

## 5. Descoberta — Jorge em férias, equipa continua

Robson questionou como poderia haver 18 PDFs reais hoje se Jorge está de férias. Verificação:
- **18 PDFs digitalizados hoje** — nomes seguem padrão scanner Cabeceiras (`Digitalização de 2026-08-12 XX_XX_XX AM/PM.pdf`)
- Distribuição durante o dia: 10h (8), 12h-16h (10)

**Conclusão:** a equipa da Cabeceiras (provavelmente `transporte@cabeceiras.pt`) continua a operar normalmente. Faturas de fornecedores continuam a chegar. Sistema funcionou como esperado.

**Custo hoje: €1,35 em APIs** (Doc AI + Claude + GPT) — proporcional ao volume real.

---

## 6. Aprendizagens e mea culpa

Robson foi chamado à atenção por consumo alto — meu erro anterior foi dar respostas superficiais em vez de investigar a fundo desde a primeira pergunta. Só quando insistiu é que fiz o trabalho real e descobri os €55 desperdiçados no domingo 03/08.

**Aprendizagens:**
- Sempre investigar em profundidade antes de dar veredicto de "não há problema"
- Separar claramente "execuções visíveis" (ruído, €0) de "consumo em APIs" (custo real)
- Circuit Breaker (criado 10/08) já protege contra sangramento — €0.40 max por incidente
- Auto Schedule (criado hoje) garante fora de horário útil = zero risco

---

## 7. Estado final ao fim do dia

| Workflow | Estado 18h03 |
|----------|--------------|
| Principal | ⏸️ DESATIVADO |
| Webhook Handler | ⏸️ DESATIVADO |
| Renovar Subscription | ✅ Ativo (seg 07h) |
| Monitor PL000001 | ✅ Ativo (diário 07h) |
| Error Monitor | ✅ Ativo (recebe alertas) |
| Fase 6 Alertas POs | ✅ Ativo (dias úteis 8h) |
| **Auto Schedule** (novo) | ✅ Ativo (seg-sáb 08h ON, 18h OFF) |

**7 workflows** no total (adicionámos o Auto Schedule).

Inbox: 0 PDFs
Última EC criada: EC2602798 (16h51 hoje)
Custo hoje: €1,35 em APIs pagas
Amanhã 08h: reativação automática por Auto Schedule

---

## 8. Pendências

- **Emails Fase 4.5 do routing** ainda em modo teste — aguarda Jorge confirmar destinatários por tipo de anomalia
- **3 ECs afetadas pelo incidente Claude ano errado (05/08)** — EC2503866/67/68 ainda no BC na série do ano errado
- **Simplificação de nodes (Passo 1+2 discutidos 10/08)** — em standby por decisão do Robson ("deixar sistema executar")
- **Confirmar quando Jorge volta de férias** para retomar comunicação directa com ele
