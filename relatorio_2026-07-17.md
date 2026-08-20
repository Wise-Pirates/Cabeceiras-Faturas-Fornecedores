# Relatório — Projeto 21 — Sexta 2026-07-17

**Cliente:** Cabeceiras.pt (Jorge Brandão)
**Foco:** Análise do documento "Estratégia Automação Facturas" do Jorge + implementação dos 4 requisitos

---

## 1. Contexto

Jorge criou documento `.docx` "Estratégia Automação Facturas" com 4 requisitos + 6 imagens ilustrativas. Sessão dedicada a análise + implementação.

---

## 2. 4 requisitos do Jorge (documento)

**1. Cód. Comprador (JB)** — todas as ECs criadas pela automação devem ter `Cód. Comprador = JB` no cabeçalho.

**2. Pastas OneDrive** — gralha do Jorge, escreveu "faturas inseridas" mas quis dizer "faturas por inserir". Sem alteração.

**3. Duplicados** — criar EC com linha "Comentário" com descrição `DOCUMENTO DUPLICADO`.

**4. Verificar duplicados** em Encomendas Compra + Faturas Compra Registadas.

---

## 3. Descoberta técnica crítica — BC aceita `type=' '` (espaço) como Comentário

Inicialmente disse ao Jorge que BC não aceitava `type='Comment'` via API. Jorge respondeu: "está nas opções". Ao investigar novamente descobri:

- BC OData rejeita `type='Comment'`
- BC OData aceita **`type=' '` (single space)** como comentário
- Confirmado com POST teste: linha criada com sucesso, descrição preservada

**Regra guardada:** `feedback_esgotar_hipoteses_antes_dizer_nao` — nunca dizer "sistema não suporta" ao 1º erro; testar variantes enum, ver dados existentes.

---

## 4. Fixes implementados

### 4.1 Cód. Comprador = JB
`Preparar EC Header` — adicionado `purchaserCode: 'JB'` no payload header.

### 4.2 Detecção duplicados corrigida
Bug pré-existente: query `vendorInvoiceNumber eq '356126/2248'` mas BC guarda com prefixo `'FT 356126/2248'` → nunca batia.

**Fix:** ambas queries (`Verificar EC Existente BC` + `Verificar Duplicado BC`) passam a usar `endswith(vendorInvoiceNumber, '{{numero}}')`. Escalável até milhares (filtrado por vendor).

### 4.3 Linha Comentário DOCUMENTO DUPLICADO
`Verificar EC Criado`: quando `bc_duplicado=true` OU `ec_existente=true`, adiciona linha extra no fim:
```
type: ' '  (Comentário)
description: DOCUMENTO DUPLICADO — ja registada em EC...
```

### 4.4 Anomalias bloqueantes removidas
`Avaliar Resultado BC` e `Avaliar EC Existente` — quando detectam duplicado, **não** adicionam anomalia. Sistema continua a criar EC + adiciona comentário.

---

## 5. Testes de duplicados

- **Ferragens duplicada** (FT 356126/2248) — reprocessada. Sistema criou EC nova com linha Comentário `DOCUMENTO DUPLICADO — ja registada em EC2602374` ✅

---

## 6. Bug adicional descoberto — NIF do CLIENTE Jorge extraído por engano

Doc AI extrai `510718531` (NIF Jorge Brandão Gonçalves) em vez do NIF do fornecedor. Aconteceu com Elastron/Würth/PLASPUR/Ferragens.

**Fix parcial:** `Parse e Validar` — se NIF vazio + nome extraído → fallback por nome único distintivo (já implementado em sessões anteriores).

Corrige para NIF vazio. NIF "lixo" (código postal, telefone) só corrigido no 20/07.

---

## 7. Estado final

- Workflow: DESATIVADO (fim sessão)
- 4 requisitos Jorge: 3 implementados totalmente, 1 (pastas) resolvido
- Fixes duplicados: validados com Ferragens (EC2602433)
- Regra `esgotar-hipoteses-antes-dizer-nao` guardada
