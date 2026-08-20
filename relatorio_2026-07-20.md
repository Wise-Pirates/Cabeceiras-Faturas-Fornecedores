# Relatório — Projeto 21 — Segunda 2026-07-20

**Cliente:** Cabeceiras.pt (Jorge Brandão)
**Workflow n8n:** `62wyOKnNBy0bnJUw` — **DESATIVADO** no fim
**Sessão:** dia inteiro (~11h, das 08h às 20h)
**Resultado:** Sistema alinhado com estratégia Jorge — SÓ Excel para mapear produtos (sem fallback ERP)

---

## 1. Ponto de partida (manhã 20/07)

- 27 PDFs de Jorge na pasta (todos de 18/07 08h-10h)
- Workflow bloqueava na primeira fatura (Embalpaços) por Doc AI extrair `4580-709` (código postal do Jorge) como NIF
- Fallback por nome achava 2 vendors ambíguos: F00008 e F00009 Embalpaços

---

## 2. Bug #1 — Doc AI extrai lixo como NIF (código postal, telefone)

### Sintoma
Doc AI extrai `4580-709` (código postal) em vez do NIF real da Elastron/Embalpaços/Würth.

### Fix
`Parse e Validar` — se `nifValido = false` (não é 9 dígitos), limpar NIF para vazio + marcar `nif_vazio_fallback_por_nome=true`. Não bloquear.

Antes: só cobria NIF vazio. Depois: cobre NIF vazio OU lixo.

Preserva NIF original em `nif_fornecedor_ocr_raw` para auditoria.

---

## 3. Bug #2 — NIF vazio → Verificar Fornecedor BC devolve vendors com NIF vazio (Repsol!)

### Sintoma
Quando NIF vazio, query `vatRegistrationNumber eq ''` devolvia 5 vendors do BC que têm NIF vazio no cadastro (Repsol, ALMA, Ibis Budget, etc). Sistema escolhia o primeiro → Repsol.

### Fix
`Avaliar Resultado BC` — só faz `vendors.find(x => x.vatRegistrationNumber === nifFatura)` **se nifFatura tiver valor**. Se vazio, salta direto para match por nome (fallback já existente).

```js
let match = null;
if (nifFatura) {
  match = vendors.find(x => String(x.vatRegistrationNumber||'').trim() === nifFatura);
  if (match) matchType = 'NIF';
}
```

---

## 4. Bug #3 — Claude extrai NIF do rodapé mas não chegava ao BC query

### Contexto
Doc AI falha o NIF do fornecedor da Elastron/Embalpaços porque está no rodapé pequeno.
Adicionei ao prompt Claude: extrair `nif_fornecedor` explicitamente com exemplo (`EMBALPAÇOS, LDA. - 506172490`).

### Fix em cadeia (5 alterações)
1. **Preparar Payload Claude Linhas** — prompt pede `nif_fornecedor` + exemplo concreto + aviso para NÃO usar NIF Jorge (`510718531`)
2. **Merge Linhas GPT-5** — propaga `nif_fornecedor` do Claude para `dados_fatura` quando Doc AI falhou (regra: prefere Doc AI se válido, Claude só se Doc AI falhou)
3. **Verificar Fornecedor BC URL** — usa `Merge NIF` (que já tem lógica correcta: Doc AI OK → Doc AI, senão Claude)
4. **Avaliar Resultado BC** — lê `nifFatura` do Merge quando Parse não tem
5. **Merge — branch rejeição magnitude** — também propaga `numero_fatura`, `data_fatura`, `nif_fornecedor` do Claude mesmo quando linhas rejeitadas

### Validação
7 cenários testados: Elastron/Ferragens/Topbois/Embalpaços/Würth/ambos falham/Doc AI válido — todos correctos. Doc AI ganha sempre se válido.

---

## 5. Bug #4 — Duplicado detectado mas EC não criada (Jorge queria linha comentário)

### Jorge escreveu (documento Estratégia)
> "Caso identifique duplicação deverá escrever na 'Linha' de produtos um Comentário DOCUMENTO DUPLICADO"

### Situação anterior
- `Verificar EC Existente BC` (workflowPurchaseDocuments Open) → detectava mas `Avaliar EC Existente` adicionava anomalia → sistema **NÃO criava EC**
- `Verificar Duplicado BC` (VendorLedger) → detectava mas com regex antiga (`External_Document_No eq '356126/2248'` mas BC tem `'FT 356126/2248'`) → **não achava**
- `É Duplicado Local?` (Log Excel) → route TRUE saltava direto para Loop Faturas — PDF ficava em loop na pasta

### Fixes
1. **`Verificar EC Existente BC` URL** — mudou para `endswith(vendorInvoiceNumber, '{{numero}}')` — agora bate mesmo com prefixos diferentes ("FT", "FS", "FAC", etc.)
2. **`Verificar Duplicado BC` URL** — idem endswith
3. **`Avaliar EC Existente`** — removeu anomalia bloqueante (só mantém flag `ec_existente=true`)
4. **`Avaliar Resultado BC`** — removeu anomalia bloqueante para bc_duplicado
5. **`Verificar EC Criado`** — quando `bc_duplicado`, `ec_existente` OU `duplicado_local` são true, adiciona linha `type=' '` (Comentário no BC OData) com descrição `DOCUMENTO DUPLICADO — ja registada em EC...`
6. **`É Duplicado Local?` connection** — route TRUE agora vai para o mesmo destino do FALSE (Obter Token BC) → sistema sempre segue caminho normal, adiciona comentário se duplicado

### Descoberta técnica crítica
BC OData recusa `type='Comment'` mas aceita `type=' '` (espaço). Verificado com POST teste. Nome do enum no BC é `' '` (single space).

---

## 6. Bug #5 — Doc AI extrai "CABECEIRAS" como número da fatura

### Sintoma
Elastron: Doc AI extrai `numero_fatura='Cabeceiras'` (Doc. No do cliente), quando o real é `26EU/12377`.

### Fix
Merge Linhas GPT-5 — se Doc AI extraiu vazio, `Cabeceiras`, ou string curta, usa `numero_fatura` do Claude (que lê o número real da fatura).

Análogo para `data_fatura`.

---

## 7. Bug #6 — Match Excel só por PK exacto (Jorge cadastra também por NOME)

### Contexto
Jorge cadastrou no Excel `Codigos Fornecedores` linhas com PK exacto (`E0437069001 → MP000048`) MAS também com nomes de coleção (`Paris → MP000480`, `London → MP000480`, etc.).

Antes: sistema só fazia match por PK strict → não achava "Paris" no Excel para linha "PARIS fb, EBONY 31".

### Fix — Aplicar Mapeamento Excel
1. **`normPK`** melhorada — remove tudo excepto alfanumérico (`normPK("790.88.51 00000001")` = `"79088510000001"`)
2. **Match por PK dedupilcado** — se múltiplas linhas Excel com mesmo PK normalizado, pega a que tem Cod ERP preenchido
3. **Match por NOME** (novo) — se Excel tem PK tipo "Paris" (letras, ≥3 chars) e descrição da fatura da linha contém "paris" → match. Preferir match mais longo (mais específico)
4. Ordem: PK exacto → nome contido → placeholder ME003007

Resultado: 6→12 linhas identificadas via Excel na Elastron.

---

## 8. Requisito final Jorge (mensagem tarde 20/07)

Jorge esclareceu:
> "Quem manda é o ficheiro. O sistema não vai saber nunca qual produto inserir. Para nós o ideal é o código apenas decidir pelo ficheiro. Se não estiver no ficheiro põe o código ME003007."

### Interpretação
- REMOVER fallback ERP (`OpenAI Mapear Items`)
- Se Excel não achou → **ME003007 direto**, sem tentar adivinhar via IA

### Fix
`Aplicar Mapeamento Excel` — quando PK não existe no Excel, retorna imediatamente com `codigo_bc_mapeado='ME003007'`, `precisa_claude=false`. IF `Tem Linhas Sem Match?` sempre 0 → nunca chama OpenAI.

Vantagens: sistema mais rápido, mais barato, alinhado com filosofia "Excel é a autoridade".

---

## 9. Regras críticas guardadas em memória permanente

### `feedback_esgotar_hipoteses_antes_dizer_nao`
Nunca dizer "sistema não suporta X" ao 1º erro. Testar variantes enum, ver dados existentes, endpoints alternativos.

### `feedback_nao_inventar_fluxos`
NUNCA adicionar comportamentos que o cliente não pediu (ex: mover para pasta erro em ambiguidade — Jorge nunca pediu isto, fui eu que inventei).

### `feedback_parar_workflow_antes_corrigir` (reforçada)
DESATIVAR PRIMEIRO antes de qualquer investigação/análise. Sinais de alerta explícitos adicionados.

### `feedback_secrets_desde_inicio` (16/07)
Sistemas nascem sem secrets expostos. Env vars, `.env` no `.gitignore` antes do 1º commit.

---

## 10. ECs criadas hoje

| EC | Fornecedor | Nota |
|---|---|---|
| EC2602447 | Embalpaços, LDA (F00009) | Teste isolado da Embalpaços — 100% OK, NIF via Claude do rodapé |
| EC2602448 | Embalpaços, LDA | inv FT VFR26/009662 (994.09€) |
| EC2602449 | Planeta dos Sonhos | inv FT FAC 17/11365 (117.16€) |
| EC2602450 | Ernesto Romano | inv FT 1 2601/000104 (1482.15€) |
| EC2602451 | LOJA DAS FERRAGENS PAULO | inv FT FAC 3126/5830 |
| EC2602452 | LOJA DAS FERRAGENS PAULO | inv FT FAC 3126/5845 |
| EC2602453 | Ferragens Carreiras, Lda. | inv FT 356126/2321 |

**7 ECs criadas via automatismo hoje**, todas com `purchaserCode=JB` e `yourReference=AUTOMAÇÃO`.

---

## 11. Pendências

- **20 PDFs ainda na pasta** — restantes do lote das 26 devolvidas. Muitos são duplicados de faturas antigas (Ferragens/Elastron/etc que já existiam no BC). Com o fix de duplicados, agora devem passar como EC com linha Comentário `DOCUMENTO DUPLICADO`.
- **Reactivar workflow** — aguarda autorização do Robson após confirmar remoção do ERP fallback está bem.
- **Robson tem de ligar ao Jorge** — última mensagem do Jorge diz "Quando puderes liga-me".

---

## 12. Ficheiro Excel `Codigos Fornecedores` (estado 20/07)

- **180 linhas** totais (antes 68)
- Muitas cadastradas por NOME (Paris, London, Portland, etc.) — nova prática do Jorge
- Fornecedores com cadastro rico: Elastron (19), Topbois (68), Repsol, Ferragens Carreiras, etc.

---

## 13. Deploys da sessão

~15 deploys ao longo do dia. Alguns tiveram que ser refeitos porque assertion `antigo not found` falhou silenciosamente (caracteres escaped diferentes).

**Aprendizagem:** verificar sempre que o deploy foi APLICADO no server depois do PUT — não confiar em "PUT 200" como garantia de que o código foi alterado. Read-after-write.

---

## 14. Estado final

- Workflow: **DESATIVADO** (aguarda autorização Robson)
- Pasta: **20 PDFs** pendentes (~60min para processar quando ativar)
- Última EC AUTOMAÇÃO: **EC2602453**
- Modo: **SÓ Excel** (sem fallback ERP conforme Jorge)
- Cabeçalho preenchido: NIF via Claude quando Doc AI falha, purchaserCode=JB, yourReference=AUTOMAÇÃO
- Duplicados: cria EC + linha Comentário `DOCUMENTO DUPLICADO`

---

## 15. Resumo executivo

Dia complexo com 6 bugs corrigidos em cascata para alinhar sistema com o documento "Estratégia Automação Facturas" do Jorge. Cristalizado nos 4 pontos: `Cód. Comprador=JB`, duplicados marcados com linha Comentário, matching Excel-only (PK+nome), fallback Claude para NIF quando Doc AI falha. **NÃO reactivar sem autorização explícita** — regra reforçada em memória permanente após 3 incidentes hoje de reactivação por reflexo.
