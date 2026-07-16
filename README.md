# Sistema de Automação de Faturas de Fornecedores

Automação end-to-end para processamento de faturas de fornecedores: OCR → validação → Business Central (BC).

## Arquitetura

```
Scanner → OneDrive (pasta "faturas por inserir")
  ↓ (poll 3min)
n8n workflow
  ├─ Google Document AI (cabeçalho: NIF, total, data, fornecedor)
  ├─ Claude Sonnet 4.5 (extração literal de linhas)
  ├─ Excel "Codigos Fornecedores" (dicionário PK → código BC)
  ├─ Business Central OData v4 (criar Purchase Order)
  └─ Log Excel + move PDF para pasta "faturas inseridas"
```

## Estado atual

- **Workflow principal (Fase 1):** OCR + validação + criação de EC (Encomenda de Compra) no BC
- **Workflow Fase 6:** alertas de POs em atraso (cron diário 08h, seg-sex)
- **Custo por fatura:** ~$0,02 (Document AI + Claude Sonnet 4.5)

## Filosofia de design

1. **Document AI** para cabeçalho (funciona 100%): NIF, total, data, fornecedor
2. **Claude Sonnet 4.5** para linhas (extração literal, sem inventar)
3. **Excel Codigos Fornecedores** para mapear PKs → códigos BC
4. **Match SÓ por PK** (não por descrição — evita falsos positivos)
5. **Nunca inventar** items no BC
6. **Nunca bloquear** — sistema sempre entrega valor ao operador humano
7. **Flags no `postingDescription`** sinalizam onde revisar:
   - `AUTOMAÇÃO` — tudo automático OK
   - `- ATENÇÃO VALORES` — Σ linhas ≠ total OCR
   - `- ITEM SEM MATCH` — usou placeholder ME003007
   - `- VERIFICAR FORNECEDOR` — match por nome (não por NIF)
   - `- VERIFICAR DATA` — data convertida de formato PT `DD.MM.YYYY`

## Estrutura do repositório

- `workflow_fase1_ocr.json` — workflow principal n8n (69 nodes)
- `workflow_fase6_alertas.json` — alertas POs atrasadas
- `planeamento-projeto.md` — visão geral e roadmap
- `product-discovery.md` — descoberta inicial de requisitos
- `ESTADO-ATUAL.md` — capacidades e estado técnico
- `CHECKLIST-BC-DEV-FIX.md` — checklist configuração BC DEV
- `PERGUNTAS-JORGE-*.md` — perguntas pendentes ao cliente
- `REUNIAO-JORGE-*.md` — notas de reuniões
- `relatorio_YYYY-MM-DD.md` — relatórios diários de sessões de desenvolvimento

## Segurança

O repositório **NÃO** contém:
- Credenciais Google Service Account (`cabeceiras-ocr-*.json`)
- Ficheiros Excel com dados do cliente
- Faturas reais (PDF/imagens)
- Screenshots com dados do BC
- Ficheiro `.env` com secrets

Ver `.gitignore` para lista completa.

## Stack

- **n8n** (self-hosted no Railway) — orquestração
- **Google Document AI** — OCR de cabeçalho
- **Anthropic Claude Sonnet 4.5** — extração de linhas
- **Microsoft Graph API** — OneDrive + Excel
- **Business Central OData v4** — ERP

## Cliente

Cabeceiras.pt (Jorge Brandão Gonçalves Unip., Lda.)
