# Escopo de Projeto: Agente de Automação de Compras e Faturas

> Reunião de Projeto · 2026-06-25 · Cliente: Cabeceiras.pt
> Tema: Automação de Compras e Faturas no ERP com OCR, Validação e Integrações
> Participantes: Jorge Brandão (Cabeceiras) · André Silva (Wise / Process Automation)

Este documento detalha os requisitos para o desenvolvimento de um agente de software (doravante "o Agente") para automatizar o processo de gestão de compras e faturas, desde a receção até ao registo no ERP da empresa.

## 1. Visão Geral e Desafio Principal

O cliente procura otimizar e automatizar o seu fluxo de trabalho de compras, atualmente manual, propenso a erros e consumidor de tempo significativo de recursos humanos. O processo envolve múltiplos departamentos e carece de um sistema centralizado e à prova de falhas para cruzar encomendas, guias de remessa e faturas.

O objetivo principal é libertar a equipa de compras de tarefas burocráticas, reduzir erros de inserção de dados (quantidades, unidades, IVA, produtos) e garantir que apenas as mercadorias efetivamente recebidas e validadas são pagas.

## 2. Fluxo de Trabalho Proposto

### 2.1. Registo de Encomendas
- **Processo atual:** encomendas feitas de forma descentralizada (ex.: o responsável pela produção encomenda tecidos diretamente), sem registo formal no ERP.
- **Processo futuro:** toda a equipa que realiza encomendas (atualmente 5 elementos) passa a registá-las como **"Notas de Encomenda" diretamente no ERP**, que já possui esta funcionalidade.

### 2.2. Digitalização e Processamento de Documentos
1. **Digitalização:** um scanner digitaliza os documentos (faturas, guias de remessa) para uma pasta específica no OneDrive (ex.: `faturas por inserir`).
2. **Leitura e extração de dados:** o Agente, com um modelo de visão computacional, monitoriza a pasta, lê os PDFs e extrai de cada fatura:
   - NIF/Contribuinte do fornecedor (identificação inequívoca)
   - Data da fatura
   - Número da fatura
   - Referência à guia de remessa (se aplicável)
   - Código ERP do produto
   - Quantidade
   - Unidade (metros, m², unidades, etc.)
   - Preço por unidade
   - Valor sem IVA
   - Valor com IVA
3. **Organização de ficheiros:** após o processamento, o Agente move os PDFs para uma pasta de arquivo organizada por ano e mês da data da fatura (ex.: `faturas inseridas/2026/06/`).

### 2.3. Validação e Cruzamento de Informação
O core da automação reside na capacidade de validar e cruzar dados:
1. **Validação de receção:** verificar prova de que a mercadoria foi recebida. A prova pode ser:
   - Assinatura na própria fatura digitalizada.
   - Guia de remessa correspondente (identificada pelo número na fatura) assinada e digitalizada.
   - Confirmação de receção registada diretamente no ERP.
2. **Cruzamento com Nota de Encomenda:** cruzar os dados da fatura processada com a Nota de Encomenda correspondente no ERP.
3. **Deteção de anomalias:** se houver discrepância (preço, quantidade, produto) ou faltar a prova de receção, o Agente:
   - Sinaliza o documento como contendo anomalia.
   - Envia notificação por **e-mail** (preferencial) ou **Teams** à equipa responsável, detalhando a anomalia.

### 2.4. Inserção no ERP e Validação Humana
- **Inserção automática:** se todas as validações passarem, o Agente insere os dados no ERP, transformando a "Nota de Encomenda" em "Encomenda de Venda" (o equivalente a uma fatura no fluxo do cliente). O Agente apenas insere ("grava"); não efetua o registo contabilístico final.
- **Gestão de produtos não identificados:** se o Agente não conseguir identificar um código de produto (ou se o produto for novo), insere um código placeholder pré-definido (ex.: `SSS`). Este código é configurado no ERP para gerar erro ao registar, forçando revisão manual.
- **Validação humana final:** um responsável revê as faturas inseridas, valida as anomalias sinalizadas, corrige os produtos `SSS` (criando o novo produto ou associando a existente) e dá a "receção" ou "receção e registo" final no ERP.

## 3. Histórias de Utilizador
- **Como gestor de compras,** quero que as faturas recebidas sejam lidas e inseridas automaticamente no ERP, para alocar o meu tempo a tarefas de maior valor (ex.: negociação com fornecedores).
- **Como responsável de contabilidade,** quero a certeza de que cada fatura inserida corresponde a uma encomenda real e a mercadoria efetivamente recebida, para evitar pagamentos duplicados ou indevidos.
- **Como colaborador que faz encomendas,** quero ser notificado automaticamente se uma fatura não corresponder à minha encomenda (preço ou quantidade), para resolver a discrepância com o fornecedor rapidamente.
- **Como administrador do sistema,** quero que o Agente sinalize produtos novos ou não reconhecidos com um código específico, para os localizar e criar/corrigir no ERP facilmente.

## 4. Requisitos Técnicos e Complexidades
- **Modelo de visão (OCR+AI):** sucesso depende de um modelo de visão robusto, capaz de ler layouts de faturas de múltiplos fornecedores e identificar os campos-chave. Capacidade de aprendizagem para novos layouts, possivelmente com interface de configuração inicial ("neste fornecedor, o NIF está aqui").
- **Integração com ERP (via API):**
  - Consultar a base de dados de fornecedores.
  - Consultar as Notas de Encomenda.
  - Inserir Encomendas de Venda (faturas).
  - Verificar registos de receção.
- **Regras de negócio por fornecedor:** configurável para regras específicas (ex.: Fornecedor A fatura por unidade, Fornecedor B por metro quadrado).
- **Aprendizagem contínua:** mecanismo de feedback. Quando um humano corrige um erro de interpretação, a correção refina o modelo e reduz anomalias futuras no mesmo cenário.

## 5. Partes Interessadas (Stakeholders)
- **Gestor de Compras/Contabilidade:** principal utilizador e validador; fornece as regras de negócio e valida resultados.
- **Equipa de Produção e Lojas:** registam as Notas de Encomenda e realizam a receção física.
- **Equipa de Desenvolvimento (Wise / Robson):** constrói, treina e mantém o Agente e as integrações.
- **Cliente (Jorge Brandão):** ponto de contacto para alinhamento estratégico e aprovação do escopo.
