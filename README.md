# Documento de Requisitos — CreditFlow

## 1. Contexto e objetivo

O **CreditFlow** é um projeto de estudo que simula o núcleo de dados de um sistema de gestão de recuperação de crédito (cobrança). O objetivo é praticar, de ponta a ponta, competências de banco de dados relacional em SQL Server: modelagem, normalização, escrita de T-SQL, stored procedures, índices, otimização de performance e construção de dashboards analíticos.

Não é um sistema de produção nem se destina a uso comercial. É um projeto de portfólio e aprendizado.

## 2. Escopo funcional

O sistema deve ser capaz de representar:

- Devedores (pessoa física ou jurídica) e seus contratos de dívida
- O parcelamento de cada contrato
- Tentativas de negociação/renegociação da dívida
- Operadores responsáveis pela cobrança
- Pagamentos recebidos
- O histórico de mudança de status de cada contrato ao longo do tempo (auditoria temporal)

## 3. Fora de escopo (por ora)

- Autenticação/autorização de usuários do sistema
- Integrações externas reais (bureaus de crédito, gateways de pagamento)
- Regras jurídicas de cobrança (ex: prazos legais de prescrição) — podem ser adicionadas em iteração futura
- Interface web/aplicativo (o front-end analítico será um dashboard, não um CRUD completo)

## 4. Entidades e atributos

### 4.1 Devedores

Representa a pessoa (física ou jurídica) que possui uma dívida.

| Campo | Tipo | Observação |
|---|---|---|
| devedor_id | INT (PK, IDENTITY) | Chave substituta |
| tipo_pessoa | CHAR(1) | 'F' = física, 'J' = jurídica |
| documento | VARCHAR(18) | CPF (11 dígitos) ou CNPJ (14 dígitos), armazenado sem formatação |
| nome_razao_social | VARCHAR(150) | Nome (PF) ou razão social (PJ) |
| data_nascimento_fundacao | DATE | Nascimento (PF) ou fundação (PJ) — nulo permitido |
| telefone | VARCHAR(20) | |
| email | VARCHAR(150) | |
| cidade | VARCHAR(100) | |
| uf | CHAR(2) | |
| data_cadastro | DATETIME2 | Default: data/hora atual |

**Decisão de modelagem:** optei por uma única tabela `Devedores` com o campo `tipo_pessoa` como discriminador, em vez de duas tabelas separadas (`PessoaFisica` / `PessoaJuridica`). Isso é o padrão conhecido como **Single Table Inheritance** simplificado. A alternativa (tabela `Devedores` genérica + tabelas `DevedorPF`/`DevedorPJ` com FK 1:1) seria mais "correta" em termos de normalização estrita, mas adicionaria complexidade desproporcional ao ganho para este projeto — o volume de campos exclusivos de cada tipo é pequeno. Vou documentar essa ressalva no README como uma decisão consciente de trade-off, o que por si só é um bom ponto para discutir em entrevista.

### 4.2 Contratos

Representa a dívida original.

| Campo | Tipo | Observação |
|---|---|---|
| contrato_id | INT (PK, IDENTITY) | |
| devedor_id | INT (FK → Devedores) | |
| numero_contrato | VARCHAR(30) | Identificador externo/negócio |
| tipo_divida | VARCHAR(50) | Ex: 'Cartão de Crédito', 'Empréstimo Pessoal', 'Financiamento' |
| valor_original | DECIMAL(14,2) | Valor da dívida na origem |
| data_contratacao | DATE | |
| data_vencimento_original | DATE | |
| status_atual | VARCHAR(20) | 'Em Aberto', 'Negociado', 'Pago', 'Judicial', 'Cancelado' |
| carteira_origem | VARCHAR(50) | Simula de qual "banco/carteira" a dívida veio |

### 4.3 Parcelas

Parcelamento de um contrato (seja o original ou fruto de negociação).

| Campo | Tipo | Observação |
|---|---|---|
| parcela_id | INT (PK, IDENTITY) | |
| contrato_id | INT (FK → Contratos) | |
| numero_parcela | INT | Sequencial dentro do contrato |
| valor_parcela | DECIMAL(14,2) | |
| data_vencimento | DATE | |
| data_pagamento | DATE | Nulo se ainda não paga |
| status_parcela | VARCHAR(20) | 'Pendente', 'Paga', 'Atrasada', 'Cancelada' |

**Nota de negócio:** o campo `status_parcela` combinado com `data_vencimento` é o que vai alimentar o cálculo de **aging de inadimplência** (30/60/90/120+ dias), uma das métricas centrais do dashboard.

### 4.4 Operadores

Quem realiza a cobrança.

| Campo | Tipo | Observação |
|---|---|---|
| operador_id | INT (PK, IDENTITY) | |
| nome | VARCHAR(100) | |
| data_admissao | DATE | |
| ativo | BIT | |

### 4.5 Negociacoes

Histórico de tentativas de acordo entre operador e devedor sobre um contrato.

| Campo | Tipo | Observação |
|---|---|---|
| negociacao_id | INT (PK, IDENTITY) | |
| contrato_id | INT (FK → Contratos) | |
| operador_id | INT (FK → Operadores) | |
| data_negociacao | DATETIME2 | |
| valor_proposto | DECIMAL(14,2) | |
| desconto_percentual | DECIMAL(5,2) | |
| resultado | VARCHAR(20) | 'Aceita', 'Recusada', 'Sem Retorno' |
| observacoes | VARCHAR(500) | |

### 4.6 Pagamentos

Valores efetivamente recebidos.

| Campo | Tipo | Observação |
|---|---|---|
| pagamento_id | INT (PK, IDENTITY) | |
| parcela_id | INT (FK → Parcelas) | |
| valor_pago | DECIMAL(14,2) | |
| data_pagamento | DATETIME2 | |
| forma_pagamento | VARCHAR(30) | 'PIX', 'Boleto', 'Cartão', 'Transferência' |

### 4.7 StatusHistorico

Auditoria temporal de mudança de status de um contrato.

| Campo | Tipo | Observação |
|---|---|---|
| historico_id | INT (PK, IDENTITY) | |
| contrato_id | INT (FK → Contratos) | |
| status_anterior | VARCHAR(20) | Nulo na primeira inserção |
| status_novo | VARCHAR(20) | |
| data_mudanca | DATETIME2 | Default: data/hora atual |
| motivo | VARCHAR(200) | Opcional |

**Por que essa tabela e não só um `UPDATE` direto no campo `status_atual` de Contratos?** Porque sem histórico, perdemos a capacidade de responder perguntas de negócio como "quanto tempo em média um contrato leva para sair de 'Em Aberto' para 'Negociado'?" — que é exatamente o tipo de métrica que dashboards de cobrança precisam mostrar. Isso também nos dá terreno para praticar **triggers** (Fase 4/5), populando `StatusHistorico` automaticamente sempre que `Contratos.status_atual` mudar.

## 5. Relacionamentos (cardinalidade)

```
Devedores (1) ----< (N) Contratos
Contratos (1) ----< (N) Parcelas
Contratos (1) ----< (N) Negociacoes
Contratos (1) ----< (N) StatusHistorico
Operadores (1) ----< (N) Negociacoes
Parcelas   (1) ----< (N) Pagamentos
```

## 6. Regras de negócio iniciais

1. Um devedor pode ter múltiplos contratos (dívidas de origens diferentes).
2. Um contrato só pode ter parcelas associadas a ele mesmo (sem parcelas "órfãs").
3. Uma parcela pode ter mais de um pagamento (pagamento parcial) — por isso `Pagamentos` é uma tabela separada, não um campo em `Parcelas`.
4. Toda mudança de `status_atual` em `Contratos` deve gerar um registro em `StatusHistorico`.
5. `documento` (CPF/CNPJ) deve ser único por devedor.
6. Uma negociação com resultado "Aceita" deve, futuramente, poder gerar novas parcelas (renegociação) — regra a ser implementada via stored procedure na Fase 4.

## 7. Métricas de negócio que o modelo precisa suportar

Essas métricas vão guiar o desenho das views e stored procedures na Fase 4:

- Taxa de recuperação de crédito (valor recuperado / valor total em cobrança)
- Aging de inadimplência (distribuição de parcelas atrasadas por faixa de dias)
- Efetividade de negociação por operador (% de negociações aceitas)
- Tempo médio de resolução de contrato (da entrada em cobrança até status "Pago")
- Volume de dívida por tipo (cartão, empréstimo, financiamento) e por carteira de origem

## 8. Glossário

| Termo | Significado |
|---|---|
| Aging | Classificação de uma dívida/parcela pelo tempo de atraso (ex: 30/60/90/120+ dias) |
| Carteira | Conjunto de contratos vindos de uma mesma origem/credor |
| Negociação | Proposta de acordo para quitação ou parcelamento da dívida |
| Recuperação de crédito | Processo de reaver valores de dívidas em atraso |
