# Nota Técnica — Automação de Processamento de Faturas com Scanner, Agente e Integração ERP/RPA

> Data: 2026-06-25 · Cliente: Cabeceiras.pt · Autor: André Silva (Wise / Process Automation)

## 1. Objetivo da Nota
- **Processo:** implementar um fluxo automatizado de receção, leitura, validação e inserção de faturas de fornecedores, usando scanner, um agente inteligente (integrado com ERP/RPA) e organização documental em OneDrive.
- **Problemática:** elevado volume de faturas com alta propensão a erro humano (campos fiscais, unidades, códigos de produto, discrepâncias com encomendas/receções), falta de padronização entre layouts de fornecedores e necessidade de cruzamento com guias de remessa e notas de encomenda.
- **Pedido específico:**
  - Organizar faturas digitalizadas em OneDrive, com pastas `faturas por inserir` e `faturas inseridas`, distribuídas por ano/mês conforme a data da fatura.
  - Inserir automaticamente faturas no ERP via agente, com cheques técnicos (fornecedor, data, valores, IVA, produtos, quantidades, preço unitário, centro de custo).
  - Cruzar faturas com notas de encomenda (EV) e guias de remessa assinadas; sinalizar discrepâncias e anomalias.
  - Notificar a equipa por e-mail/Teams sobre desvios (quantidade, preço, receção pendente) e atrasos (ex.: material não recebido em 3 dias após encomenda).
  - Permitir validação humana antes do registo contabilístico definitivo; usar códigos placeholder (ex.: `SSS`) quando o produto não existir, forçando erro controlado até correção.

## 2. Síntese dos Elementos Disponíveis
- **Infraestrutura documental:**
  - Scanner envia PDFs para OneDrive na pasta `faturas por inserir`.
  - Pasta `faturas inseridas` organizada por data da fatura (ano/mês).
- **ERP/RPA:**
  - Funções existentes: Inserir · Receção · Registar · Receção+Registo.
  - Fluxo atual: inserção inicial (receção) anulável; registo contabilístico só após validação.
  - Notas de encomenda (EV) já suportadas pelo ERP; possibilidade de associar fatura à EV.
  - Troca/atualização de códigos de produto automática no ERP.
- **Operacional:**
  - Fornecedores com layouts variados; ~90% dos produtos têm código ERP, ~10% necessitam de criação manual.
  - Centros de custo (ex.: P1 = fábrica; VNG = loja Vila Nova de Gaia).
  - Guias de remessa frequentemente enviadas sem preços, exigem assinatura de receção.
  - Gestão por e-mails segmentados (compras1, compras2, compras3; comprasdetecidos@...) com restrição de visibilidade de preços por caixa.
  - Leitura via modelo de visão para extrair campos (NIF/contribuinte, nome do fornecedor, morada, data, valores com/sem IVA, linhas de produto, unidades/quantidades, preço unitário).
  - Necessidade de evitar pagamentos sem comprovativo de receção (guia assinada ou fatura com assinatura).
  - Casos de medição/unidades heterogéneas por fornecedor (placa vs m²; paleta vs unidade; caixa vs unidade).
- **Comunicação:** notificações preferencialmente por e-mail (caixa "zero inbox" diária), com possibilidade de Teams.

## 3. Análise Técnica

### Arquitetura do fluxo
- **Ingestão:** Scanner → OneDrive (`faturas por inserir`) → classificação por fornecedor e data (ano/mês) após extração.
- **Extração e validação:**
  - Modelo de visão lê campos-chave: NIF e nome do fornecedor (dupla confirmação), data da fatura, total sem IVA, total com IVA, número da fatura, linhas (código ERP, descrição, unidades, quantidade, preço unitário, taxa de IVA, centro de custo).
  - **Marcação por cores (config inicial por fornecedor):** na 1.ª fatura de cada fornecedor, sinalizar campos por cor para acelerar a localização (ex.: vermelho = valor sem IVA, verde = valor com IVA, azul = outros valores). Constrói um dicionário de layout por fornecedor.
  - **Regras por fornecedor para unidades/medidas:** normalização (caixa ≠ 1 unidade; paleta = n unidades; placa com área em m²) e tabelas de conversão específicas. Exemplos do cliente: paleta de ripas = 500 unidades; placa = 6 m².
  - **Cheques automáticos:**
    - Fornecedor: validar NIF e nome no ERP (dupla confirmação); criar/atualizar se necessário.
    - Datas: consistência entre data da fatura e período de inserção.
    - Valores: reconciliação linha a linha e totais (sem IVA, IVA, com IVA); tolerâncias configuráveis.
    - Centro de custo: atribuição default (ex.: P1) com ajuste humano possível (ex.: VNG).
    - Produtos: verificar existência do código ERP; se não existir, aplicar placeholder (SSS/ME/PL) que provoca erro controlado no registo, exigindo correção posterior.
    - Documentos associados: verificar guia de remessa assinada; associação fatura→guia→EV antes do registo.
  - **Cruzamento com EV:** quantidades e preços da fatura devem bater com a EV; sinalizar desvios. Se material não chegar em até 3 dias após a EV, enviar alerta para compras (e-mail/Teams).
- **Inserção no ERP:**
  - Agente insere a fatura em estado "Receção" (não contabiliza); aguarda validação humana.
  - Após validação humana: "Registo" contabilístico, exceto quando houver placeholder (erro controlado).
- **Notificações e exceções:**
  - E-mail automático com resumo de anomalias (discrepâncias, ausência de guia assinada, unidades incoerentes, código inexistente).
  - Canal alternativo: Teams, conforme preferência operacional.

### Normas e boas práticas
- **Conformidade fiscal:** validação de NIF do fornecedor; cálculo correto de IVA por taxa; integridade dos totais.
- **Controlo interno:**
  - Segregação de funções (encomenda, receção, validação, registo).
  - Pagamento condicionado a comprovativo de receção (guia assinada ou assinatura em fatura).
  - Trilho de auditoria: guardar PDFs, metadados de extração, logs de validação, histórico de notificações.
- **Gestão de dados:**
  - Catálogo de fornecedores com regras de unidades e preços padrão.
  - Dicionário de layouts por fornecedor (inclui marcação por cores e mapeamentos).

### Considerações específicas
- **Aprendizagem incremental:** memorizar novas correspondências de códigos e unidades; reduzir falsos positivos de anomalia com base em confirmações repetidas.
- **Segurança e acesso:** e-mails segmentados por departamento conectados ao agente; restrições de visibilidade de preços conforme caixas (compras1/2/3).
- **Organização documental:** estrutura por ano/mês garantida; relação fatura→EV→guia armazenada.

## 4. Conclusão / Posição
É viável implementar um fluxo automatizado com scanner, agente e ERP/RPA que:
- Organiza faturas no OneDrive por data e fornecedor.
- Extrai e valida campos críticos com modelo de visão robusto, considerando layouts e regras por fornecedor.
- Cruza faturas com EV e guias de remessa assinadas antes de qualquer registo contabilístico.
- Insere faturas no ERP em estado de receção, exige validação humana para registo, com erros controlados via códigos placeholder.
- Notifica a equipa por e-mail/Teams sobre discrepâncias e atrasos (incl. alerta se não houver receção em até 3 dias após EV).

**Reservas:**
- A qualidade do modelo de visão é determinante; será necessário treino/configuração por fornecedor e manutenção do dicionário de layouts.
- Regras de conversão de unidades devem ser formalizadas por fornecedor para minimizar erros.
- O processo exige disciplina na digitalização de guias de remessa com assinatura, para garantir pagamento apenas mediante comprovativo.

**Recomendações:**
- Fase piloto com 2-3 fornecedores representativos (layouts e unidades diversos).
- Definição de tolerâncias de preço/quantidade e SLA de receção (ex.: 3 dias) documentados.
- Logs detalhados e relatórios semanais de anomalias para ajuste fino.

## 5. Anexos (a produzir)
- Fluxograma: Scanner → OneDrive → Extração/Validação → Cruzamento EV/Guia → Inserção ERP (Receção) → Validação Humana → Registo Contabilístico → Arquivo em `faturas inseridas`.
- Mapeamentos por fornecedor: exemplos de layouts com marcação de campos (NIF, totais em cores, linhas).
- Tabela de conversões de unidades por fornecedor (ex.: paleta → unidades; placa → m²).
- Modelos de e-mail/Teams de notificação de anomalias e atrasos.
- Lista de estados no ERP/RPA e regras de transição (Inserir, Receção, Registar, Receção+Registo).
