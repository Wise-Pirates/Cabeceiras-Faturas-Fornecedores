# Checklist para Developer BC — Fix ambiente DEV

> **Objetivo:** desbloquear criação de Encomendas de Compra no ambiente **DEV** do Business Central da Cabeceiras
> **Impacto:** apenas ambiente DEV (PROD intocado)
> **Tempo estimado:** 2 minutos

---

## Contexto

Sistema automatizado de processamento de faturas de fornecedores está em teste no BC DEV. Ao tentar criar uma Encomenda de Compra (via API ou manualmente pelo browser), o BC devolve:

```
Application_FieldValidationException:
"Starting No. must have a value in No. Series Line:
 Series Code=C_EC, Line No.=2825000.
 It cannot be zero or empty."

CorrelationId: b7e8dbd7-2b2a-46ab-8c3a-51fd299b99b9
```

Isto acontece **até ao clicar "Novo" manualmente** em "Encomendas de Compras" no BC DEV. Não é problema da API — é configuração do ambiente.

---

## Fix

**Ambiente:** BC DEV (URL abaixo)
**Série a corrigir:** `C_EC`
**Linha específica:** `Line No. = 2825000`
**Campo em falta:** `Starting No.` (Nº Inicial)

### Passo a passo

1. Aceder ao BC DEV:
   ```
   https://businesscentral.dynamics.com/f41c8222-df66-449c-93b5-c1879e641cb2/DEV
   ```

2. 🔍 Lupa → **"Nº de Séries"** (No. Series)

3. Abrir a série com **Código = `C_EC`**

4. Ir a **"Linhas"** (No. Series Lines) — pode estar no ribbon "Navegar" ou "Relacionado"

5. Encontrar a linha com:
   - `Line No. = 2825000`
   - `Nº Inicial (Starting No.) = (vazio)`

6. Preencher o **Nº Inicial**. Sugestão (para dar continuidade ao padrão existente):
   - Última EC gerada foi `EC2503097` (Dez 2025)
   - Para 2026, sugere-se: `EC26000001` ou similar
   - **⚠️ Cuidado:** `Nº Inicial` e `Nº Final` têm de ter o **mesmo número de caracteres**. Se preencher Nº Final também, garantir mesmo comprimento.

7. Confirmar campo **"Aberto" (Open) = ✅**

8. Guardar (Ctrl+S)

---

## Verificação após guardar

Testar manualmente no BC DEV:

1. Ir a **"Encomendas de Compras"**
2. Clicar **"Novo"**
3. Confirmar que aparece um número no campo `Nº`

Se aparecer número → problema resolvido, o sistema automatizado retomará no próximo ciclo (até 5 min).
Se ainda der erro → contactar Robson (`advxautomate@gmail.com`) com o novo `CorrelationId`.

---

## O que o sistema fará quando o DEV estiver corrigido

Sem qualquer ação adicional necessária:
- Próximo ciclo (a cada 5 min) tenta processar novamente
- PDFs em `faturas por inserir` são reprocessados
- ECs criadas em **DEV** (não PROD)
- Log Excel atualizado com estado `Inserido BC DEV` e número da EC gerada

---

## Detalhes técnicos (para consulta)

- **Tenant ID:** `f41c8222-df66-449c-93b5-c1879e641cb2`
- **Environment:** `DEV`
- **Company:** `Jorge Brandão Gonçalves`
- **App Registration usada pelo sistema:** Client ID `55e26621-e247-4026-9f47-bf17728e77bb`
- **Endpoint API que estamos a chamar:** `POST .../DEV/ODataV4/Company('Jorge Brand%C3%A3o Gon%C3%A7alves')/workflowPurchaseDocuments`
- **Payload mínimo testado:** `{"documentType": "Order", "buyFromVendorNumber": "F00001"}`

Nenhuma alteração no PROD é necessária.
