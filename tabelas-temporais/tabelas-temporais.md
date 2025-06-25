# Tabelas Temporais no DB2 for z/OS

Documentação técnica especializada, orientada à implementação, auditoria e diagnóstico em ambientes críticos e regulados. Fundamentada em documentação oficial da IBM, práticas consolidadas e experiência real em produção.

---

## Índice

1. [Conceito e Finalidade das Temporal Tables](#1-conceito-e-finalidade-das-temporal-tables)  
2. [Tipos de Tabelas Temporais](#2-tipos-de-tabelas-temporais)  
3. [System-Time Temporal Tables](#3-system-time-temporal-tables)  
4. [Business-Time Temporal Tables](#4-business-time-temporal-tables)  
5. [Bitemporal Tables (System + Business Time)](#5-bitemporal-tables-system--business-time)  
6. [Consultas Temporais](#6-consultas-temporais)  
7. [Alterações na Estrutura e Impactos](#7-alterações-na-estrutura-e-impactos)  
8. [Boas Práticas e Cuidados Operacionais](#8-boas-práticas-e-cuidados-operacionais)  
9. [Referências Oficiais](#9-referências-oficiais)  

---

## 1. Conceito e Finalidade das Temporal Tables

### 🔍 O que são?

As **Temporal Tables** permitem capturar e consultar dados com **contexto temporal** diretamente no banco. São ideais para manter o **histórico de alterações** de forma segura, automática e auditável.

### 🎯 Finalidade

- **Auditoria automatizada** sem uso de triggers
- **Rastreabilidade** completa de dados ao longo do tempo
- **Reconstrução de estado histórico**
- **Conformidade regulatória** (LGPD, BC, SOX)
- **Diagnóstico e análise de incidentes**

---

## 2. Tipos de Tabelas Temporais

### 📚 Visão Geral

O DB2 for z/OS oferece três mecanismos principais:

| Tipo                         | Gerenciado por | Finalidade                                |
|-----------------------------|----------------|--------------------------------------------|
| System-Time                 | Sistema         | Histórico de alterações automáticas        |
| Business-Time               | Aplicação       | Validade contratual, projeção de vigência  |
| Bitemporal                  | Sistema + Aplicação | Combinação de histórico + vigência     |

---

## 3. System-Time Temporal Tables

### 📌 3.1 O que é?

Uma **System-Time Temporal Table** é uma tabela especial onde o **DB2 gerencia automaticamente os registros históricos** de acordo com o tempo do sistema. Ao ocorrer um `UPDATE` ou `DELETE`, a versão anterior do dado é copiada para uma **tabela de histórico** associada.

### ❓ Por que usar?

- Garante **versionamento automático**
- Evita uso de triggers ou ETLs customizados
- Preserva versões antigas para **auditoria** ou **investigação**
- Essencial em ambientes onde a **prova de integridade histórica** é mandatória

### 🛠️ 3.2 Como funciona?

#### Colunas obrigatórias:

- `row_begin`: início da validade do registro no sistema
- `row_end`: fim da validade
- Ambas são do tipo `TIMESTAMP(12)` e controladas pelo DB2

#### Cláusula SQL:

```sql
PERIOD SYSTEM_TIME (row_begin, row_end)
```

#### Comportamento automático:

| Operação | Ação do DB2 |
|----------|-------------|
| INSERT   | Define `row_begin = CURRENT_TIMESTAMP`, `row_end = '9999-12-30...'` |
| UPDATE   | Move linha antiga para tabela de histórico com `row_end = CURRENT_TIMESTAMP` |
| DELETE   | Move linha excluída para histórico com intervalo fechado             |

---

### 🔧 3.3 Exemplo de Criação

#### Tabela Base

```sql
CREATE TABLE cliente (
   id_cliente INTEGER NOT NULL PRIMARY KEY,
   nome       VARCHAR(100),
   status     CHAR(1),
   row_begin  TIMESTAMP(12) GENERATED ALWAYS AS ROW BEGIN,
   row_end    TIMESTAMP(12) GENERATED ALWAYS AS ROW END,
   PERIOD SYSTEM_TIME (row_begin, row_end)
)
WITH SYSTEM VERSIONING;
```

#### Tabela de Histórico

```sql
CREATE TABLE cliente_hist LIKE cliente;
```

#### Habilitação do versionamento

```sql
ALTER TABLE cliente
   ADD VERSIONING USE HISTORY TABLE cliente_hist;
```

---

### 🧠 3.4 Como modelar no PowerDesigner

No PowerDesigner, essa estrutura deve ser representada com:

- As colunas `row_begin` e `row_end` declaradas como `TIMESTAMP(12)`
- Uma constraint de período (`PERIOD SYSTEM_TIME`) informada manualmente no **Physical Options**
- Tabela de histórico referenciada no modelo físico como **history table associada**

🔍 **Dica**: usar **Extended Attributes** ou **Labels** para marcar tabelas com versionamento ativo. Também pode-se criar uma View para facilitar acesso e restringir exposições diretas à tabela de histórico.

---

### 📎 3.5 Glossário aplicado

- **System Time**: Referência temporal gerenciada automaticamente pelo DB2 (não configurável pela aplicação)
- **PERIOD SYSTEM_TIME**: Declaração do intervalo temporal válido
- **History Table**: Tabela onde o DB2 grava versões anteriores das linhas da tabela base
- **GENERATED ALWAYS AS ROW BEGIN/END**: Declaração que obriga o DB2 a preencher automaticamente os valores de tempo do sistema

---

## 4. Business-Time Temporal Tables

### 📌 4.1 O que são?

Business-Time Temporal Tables permitem representar **a vigência de dados sob o ponto de vista do negócio**, independentemente do momento em que foram inseridos ou modificados no sistema. A validade dos registros é controlada por **colunas de data definidas pela aplicação**, e não pelo sistema de banco de dados.

---

### 🧠 4.2 Diferença entre System-Time e Business-Time

| Aspecto         | System-Time                     | Business-Time                       |
|------------------|----------------------------------|--------------------------------------|
| Controlado por   | DB2 automaticamente             | Aplicação ou lógica de negócio       |
| Para que serve?  | Auditar mudanças físicas        | Representar vigência do mundo real   |
| Alterado quando? | Sempre que linha for alterada   | Quando vigência de negócio mudar     |
| Exemplo prático  | Gravar que um valor foi alterado em 20/06/2025 | Representar que o valor vale a partir de 01/01/2026 |

---

### 🧭 4.3 Casos de uso típicos

- Tabelas de preços com vigência futura
- Regras contratuais válidas por períodos específicos
- Mudança de alíquotas de impostos com data planejada
- Suspensões ou inclusões retroativas em contratos

---

### 🛠️ 4.4 Como funciona?

- O DBA define colunas de vigência: `vig_inicio`, `vig_fim`
- A cláusula `PERIOD BUSINESS_TIME` é usada para informar que essas colunas controlam o período
- O DB2 valida automaticamente as cláusulas `FOR BUSINESS_TIME` nas queries
- A responsabilidade de manutenção da vigência é da **aplicação**

---

### 🧪 4.5 Exemplo prático

```sql
CREATE TABLE plano_saude (
   id_plano       INTEGER NOT NULL,
   descricao      VARCHAR(100),
   vig_inicio     DATE NOT NULL,
   vig_fim        DATE NOT NULL,
   PERIOD BUSINESS_TIME (vig_inicio, vig_fim),
   PRIMARY KEY (id_plano, vig_inicio)
);
```

A cláusula `PERIOD BUSINESS_TIME` declara o par de colunas que define o tempo de negócio.

---

### ⏳ 4.6 Manipulação de vigência

#### Inclusão futura

```sql
INSERT INTO plano_saude
VALUES (1, 'Plano Ouro', '2026-01-01', '9999-12-31');
```

#### Encerramento parcial

```sql
UPDATE plano_saude
SET vig_fim = '2025-12-31'
WHERE id_plano = 1 AND vig_inicio = '2024-01-01';
```

#### Correção retroativa

```sql
INSERT INTO plano_saude
VALUES (1, 'Plano Ouro Corrigido', '2023-01-01', '2023-12-31');
```

🧠 Aqui, você está **inserindo dados para um período já encerrado**, algo que **só é possível com Business-Time**.

---

### 🧬 4.7 Validação de sobreposição (opcional)

O DB2 **não proíbe sobreposição de vigências**, mas você pode implementar lógica customizada via:

- Trigger
- Procedure de validação antes do insert/update
- Constraint baseada em função temporal (em versões mais recentes)

---

### 🧰 4.8 Modelagem no PowerDesigner

#### Modelo lógico

- Criar colunas `vig_inicio` e `vig_fim` com semântica clara
- Marcar como “validity period” ou “business temporal” (via extended attribute)
- Indicar na documentação que a manutenção é feita pela aplicação

#### Modelo físico

- Declarar explicitamente:

```sql
PERIOD BUSINESS_TIME (vig_inicio, vig_fim)
```

- Incluir `vig_inicio` na chave primária ou índice clusterizado, se necessário

---

### 📊 4.9 Consultas com Business-Time

```sql
SELECT * FROM plano_saude
FOR BUSINESS_TIME AS OF DATE('2025-07-01');
```

> Retorna a versão vigente no negócio na data consultada.

```sql
SELECT * FROM plano_saude
FOR BUSINESS_TIME BETWEEN DATE('2023-01-01') AND DATE('2024-12-31');
```

> Consulta que abrange múltiplas vigências.

---

### 🔍 4.10 Considerações técnicas

| Aspecto                   | Detalhe                                                              |
|----------------------------|----------------------------------------------------------------------|
| Não possui histórico       | DB2 não armazena versões anteriores automaticamente                  |
| Requer controle da app     | Inserções e atualizações precisam cuidar da integridade temporal     |
| Pode conter vigências futuras | Sim, inclusive com datas ainda não iniciadas                        |
| Aceita sobreposição        | Sim, mas isso pode ser indesejado — controlar via lógica externa     |
| Chave composta recomendada | PK envolvendo `id` + `vig_inicio` para evitar duplicação de vigência |

---

### 📎 4.11 Glossário aplicado

- **Business-Time**: Linha do tempo que representa a vigência do dado para fins de negócio
- **Vigência**: Intervalo de início e fim da validade do registro na visão do processo de negócio
- **Retroatividade**: Inclusão de registros com datas anteriores à data atual
- **Encerramento parcial**: Terminar um registro vigente antes da data planejada
- **Correção histórica**: Ajustar períodos anteriores sem apagar o dado original

---

> **Nota:** Tabelas com Business-Time são especialmente úteis em **cenários regulados, contratos públicos e sistemas de benefícios**, onde a rastreabilidade de validade futura e retroativa é mandatória.


---

## 5. Bitemporal Tables (System + Business Time)

### 📌 5.1 O que é?

**Bitemporal Table** é uma tabela que combina dois eixos temporais distintos:

1. **System Time** – refere-se ao momento em que os dados foram armazenados ou modificados no sistema (gerenciado automaticamente pelo DB2).
2. **Business Time** – representa o período em que a informação é válida para o negócio (controlado pela aplicação).

Esse modelo fornece **visibilidade histórica completa**, tanto sob a ótica do **tempo operacional (sistema)** quanto do **tempo de validade negocial**, permitindo análises temporais precisas, comparações e auditorias robustas.

---

### 💡 5.2 Por que usar?

- Permite **consultas de reconstrução total de contexto** (“o que sabíamos em 10/01/2023 sobre o contrato que valia para julho de 2023?”)
- Suporta **atualizações retroativas** e **vigências futuras**, mantendo integridade dos dois tempos
- Essencial para compliance em ambientes com exigência de rastreabilidade dupla
- Evita conflito entre o "valor válido" e o "valor conhecido"

---

### 🔬 5.3 Como funciona?

#### Estrutura da tabela:

| Coluna         | Finalidade                           | Gerenciado por |
|----------------|--------------------------------------|----------------|
| `sys_start`    | Início da validade no sistema        | DB2            |
| `sys_end`      | Fim da validade no sistema           | DB2            |
| `bus_start`    | Início da validade no negócio        | Aplicação      |
| `bus_end`      | Fim da validade no negócio           | Aplicação      |

#### Declarações SQL:

```sql
PERIOD SYSTEM_TIME (sys_start, sys_end),
PERIOD BUSINESS_TIME (bus_start, bus_end)
```

- O DB2 cuida automaticamente da manutenção da **System-Time**
- A aplicação é responsável pela **Business-Time**

---

### 🧪 5.4 Exemplo de criação

```sql
CREATE TABLE contrato (
   id_contrato   INTEGER,
   status        CHAR(1),
   sys_start     TIMESTAMP(12) GENERATED ALWAYS AS ROW BEGIN,
   sys_end       TIMESTAMP(12) GENERATED ALWAYS AS ROW END,
   PERIOD SYSTEM_TIME (sys_start, sys_end),
   vig_inicio    DATE NOT NULL,
   vig_fim       DATE NOT NULL,
   PERIOD BUSINESS_TIME (vig_inicio, vig_fim),
   PRIMARY KEY (id_contrato, vig_inicio)
)
WITH SYSTEM VERSIONING;
```

#### Histórico:

```sql
CREATE TABLE contrato_hist LIKE contrato;
```

#### Ativação do versionamento:

```sql
ALTER TABLE contrato
   ADD VERSIONING USE HISTORY TABLE contrato_hist;
```

---

### 🧠 5.5 Como modelar no PowerDesigner

#### Modelo lógico

- Representar claramente os dois pares de datas:
  - `sys_start`, `sys_end` (System Time)
  - `vig_inicio`, `vig_fim` (Business Time)
- Usar **anotações ou stereotypes** para marcar "system managed" vs "business managed"
- Atribuir roles nas colunas (`validity period`, `audit period`)

#### Modelo físico

- Usar `TIMESTAMP(12)` para `sys_*`, e `DATE` ou `TIMESTAMP` para `vig_*`
- Declarar ambas as cláusulas `PERIOD SYSTEM_TIME` e `PERIOD BUSINESS_TIME`
- Anotar a relação com a tabela de histórico
- Validar impacto em chaves primárias e índices: incluir `vig_inicio` como parte da PK costuma ser prática recomendada

---

### 🧮 5.6 Exemplo de uso avançado

#### Atualização com vigência futura:

```sql
-- Termina vigência atual
UPDATE contrato
SET vig_fim = '2025-12-31'
WHERE id_contrato = 100 AND vig_inicio = '2024-01-01';

-- Nova versão válida a partir de 2026
INSERT INTO contrato VALUES (
   100, 'A',
   DEFAULT, DEFAULT, -- sys_start/sys_end
   '2026-01-01', '9999-12-31'
);
```

➡️ O DB2 registrará automaticamente a versão anterior na `contrato_hist` com a data do sistema, enquanto a vigência de negócio é controlada pela aplicação.

---

### 🔍 5.7 Consultas bitemporais

```sql
SELECT * FROM contrato
FOR SYSTEM_TIME AS OF '2024-03-15-10.00.00'
FOR BUSINESS_TIME AS OF '2024-07-01';
```

🧠 Isso responde:  
> *"Qual era o status do contrato, que estaria vigente em 01/07/2024, de acordo com o que o sistema conhecia em 15/03/2024 às 10h?"*

---

### 🔎 5.8 Considerações técnicas

| Aspecto                  | Detalhes                                                                 |
|--------------------------|--------------------------------------------------------------------------|
| Integridade temporal     | Requer lógica de aplicação ou processos de consistência para evitar sobreposição de períodos |
| Histórico                | O DB2 grava automaticamente apenas o `SYSTEM_TIME`                      |
| Views e exposições       | Recomendado encapsular a complexidade com **views temporais**            |
| Performance              | Índices devem ser avaliados cuidadosamente para colunas temporais       |
| Diagnóstico              | Pode ser usado para identificar **erros retroativos**, **fraudes**, ou **registros indevidos** |

---

### 📎 5.9 Glossário aplicado

- **Bitemporalidade**: Capacidade de registrar e consultar dados em dois eixos temporais: sistema e negócio
- **System Time**: Controle automático do DB2 sobre quando os dados foram inseridos ou alterados
- **Business Time**: Vigência do dado sob a ótica da aplicação ou do negócio
- **FOR SYSTEM_TIME / FOR BUSINESS_TIME**: Cláusulas SQL que permitem consultas retrospectivas ou hipotéticas

---

### 🧭 6.2 Cláusulas básicas

| Cláusula SQL              | Finalidade                                         |
|---------------------------|----------------------------------------------------|
| `FOR SYSTEM_TIME AS OF`   | Estado em instante do sistema                      |
| `FOR SYSTEM_TIME BETWEEN` | Estado entre dois pontos temporais do sistema     |
| `FOR BUSINESS_TIME AS OF` | Estado com base em vigência negocial              |
| `FOR SYSTEM_TIME ... FOR BUSINESS_TIME` | Consultas bitemporais cruzadas |

---

### 🧪 6.3 Exemplos com System-Time

#### 🔎 Estado em momento exato do sistema

```sql
SELECT * FROM cliente
FOR SYSTEM_TIME AS OF '2025-02-01-10.00.00';
```

> Retorna os dados **vigentes no sistema** naquele exato instante, incluindo versões anteriores que já foram arquivadas.

---

#### 🔎 Histórico entre dois pontos

```sql
SELECT * FROM cliente
FOR SYSTEM_TIME BETWEEN
   '2024-01-01-00.00.00' AND '2024-12-31-23.59.59';
```

> Retorna todas as versões do dado que **estiveram vigentes** entre os dois timestamps.

---

#### 🔎 Histórico aberto (desde)

```sql
SELECT * FROM cliente
FOR SYSTEM_TIME FROM
   '2023-01-01-00.00.00' TO CURRENT_TIMESTAMP;
```

> Útil para investigar mudanças ao longo de um ano ou desde um evento específico.

---

### 🧪 6.4 Exemplos com Business-Time

#### 🔎 Estado de negócio em determinada vigência

```sql
SELECT * FROM tarifa
FOR BUSINESS_TIME AS OF DATE('2026-01-01');
```

> Indica qual valor estava **vigente para o cliente** naquela data, independente de quando foi inserido no sistema.

---

#### 🔎 Período de vigência de valores

```sql
SELECT * FROM tarifa
FOR BUSINESS_TIME BETWEEN DATE('2025-01-01') AND DATE('2025-12-31');
```

> Traz os registros cujas vigências **abrangem** o período.

---

### 🔁 6.5 Consultas Bitemporais

#### 🔎 Estado de negócio, conforme conhecido no passado

```sql
SELECT * FROM contrato
FOR SYSTEM_TIME AS OF '2024-03-15-10.00.00'
FOR BUSINESS_TIME AS OF '2024-07-01';
```

> Interpretação:
> *"Com base nas informações que o sistema possuía em 15/03/2024 às 10h, qual era o contrato vigente em 01/07/2024?"*

---

### 📈 6.6 Estratégias de Indexação

#### Por que indexar colunas temporais?

Consultas como `FOR SYSTEM_TIME AS OF` ou `FOR BUSINESS_TIME BETWEEN` podem se tornar **custosas** em tabelas com grandes volumes de histórico. O uso de **índices adequados** reduz significativamente o custo de execução.

| Coluna recomendada          | Tipo de índice           | Observações                                       |
|-----------------------------|---------------------------|--------------------------------------------------|
| `row_begin`, `row_end`      | Composto (`row_end`, `row_begin`) | Melhora `AS OF`, `BETWEEN` e `FROM TO`         |
| `vigencia_ini`, `vigencia_fim` | Composto ou função temporal | Importante para business-time queries            |
| Colunas de identificação    | Incluir em índices compostos | Ex: `(id_contrato, vig_inicio)`                 |

> 🧠 **Dica técnica**: Sempre validar o plano de acesso (`EXPLAIN`) em ambiente de homologação para evitar full scans indesejados.

---

### 🧯 6.7 Considerações de performance

- Evite misturar `FOR SYSTEM_TIME` com `WHERE` em colunas mal indexadas
- Considere **materialized query tables (MQTs)** para cenários com volume altíssimo de histórico
- Utilize **views específicas** para encapsular regras temporais e padronizar o uso pelas aplicações

---

### 📎 6.8 Glossário aplicado

- **FOR SYSTEM_TIME AS OF**: Consulta o estado do dado vigente no sistema naquele instante
- **FOR BUSINESS_TIME AS OF**: Consulta o que o negócio considera válido para determinada data
- **Bitemporal Query**: Combinação das duas cláusulas para refletir visão histórica e negocial
- **Plan Stability**: Estratégia de garantir que o acesso à temporal table continue eficiente mesmo após mudança de dados ou estatísticas

---

## 7. Alterações na Estrutura e Impactos

### 📌 7.1 Introdução

Alterações em tabelas temporais exigem atenção redobrada por parte do DBA. Diferentemente de tabelas convencionais, as temporal tables estão **acopladas a mecanismos internos do DB2** que gerenciam versionamento, histórico e integridade temporal.

Realizar alterações incorretas pode:

- Invalidar o histórico
- Quebrar o versionamento automático
- Impedir consultas temporais
- Romper conformidade regulatória

---

### 🛠️ 7.2 Tipos de alteração e suas implicações

#### 🔧 Adição de colunas

Permitido **com restrições**:

```sql
ALTER TABLE cliente ADD COLUMN email VARCHAR(255);
```

🧠 **Impacto técnico**:

- A nova coluna será **adicionada tanto à tabela base quanto à tabela de histórico**
- O valor será `NULL` nos registros históricos anteriores à inclusão
- Não quebra o versionamento

---

#### 🧨 Alteração de tipo de coluna temporal

**NÃO PERMITIDO diretamente** se a coluna estiver declarada em `PERIOD SYSTEM_TIME` ou `PERIOD BUSINESS_TIME`.

Exemplo proibido:

```sql
ALTER TABLE contrato ALTER COLUMN sys_start SET DATA TYPE TIMESTAMP(6); -- ERRO
```

🔎 Para alterar:

1. Remover versionamento
2. Alterar a coluna
3. Reconfigurar versionamento

⚠️ **Esse processo implica perda de histórico ativo. Precisa ser documentado, versionado e validado.**

---

#### 🔄 Renomear colunas temporais

Permitido apenas se **a coluna for removida do período temporal antes da renomeação.**

Etapas:

```sql
ALTER TABLE cliente DROP PERIOD SYSTEM_TIME;
ALTER TABLE cliente RENAME COLUMN row_end TO row_fim;
ALTER TABLE cliente ADD PERIOD SYSTEM_TIME (row_begin, row_fim);
```

📌 Atenção:
- A tabela de histórico deve ser atualizada também
- Views, procedures e triggers que utilizem essas colunas precisam ser revistos

---

#### ❌ Drop de colunas temporais

Não permitido diretamente.

É necessário primeiro desativar o versionamento:

```sql
ALTER TABLE cliente DROP VERSIONING;
ALTER TABLE cliente DROP PERIOD SYSTEM_TIME;
ALTER TABLE cliente DROP COLUMN row_end;
```

💡 Dica profissional: **nunca descartar colunas temporais sem backup lógico completo e sem uma política formal de retenção.**

---

### 📂 7.3 Alterações na tabela de histórico

A tabela de histórico deve ser **estruturalmente compatível** com a tabela base.

#### ❗ Restrições

- Não pode ter **constraints adicionais** (PK, FK)
- Não pode conter **colunas extras** não presentes na base
- Não pode ter **triggers**

📌 Toda alteração na base deve ser **replicada manualmente** na tabela de histórico.

---

### 🔒 7.4 Versão e proteção de dados históricos

O versionamento é sensível a qualquer alteração de estrutura. Por isso, recomenda-se:

- Criar **procedimentos de auditoria de DDL**
- Manter **controle de versões de scripts de estrutura**
- Usar **ferramentas de diff físico/lógico** (como DB2 Administration Tool ou PowerDesigner compare)

---

### 🧠 7.5 Estratégia segura de alteração

```text
1. Fazer backup lógico da tabela base e da tabela de histórico
2. Suspender aplicações que gravam na tabela (se necessário)
3. Desativar versionamento
4. Realizar alterações desejadas (DDL)
5. Revalidar integridade temporal
6. Reconfigurar versionamento
7. Documentar as alterações no repositório de controle de mudança
```

---

### 📈 7.6 Impactos no PowerDesigner

No modelo físico:

- Alterações em colunas `row_begin`, `row_end`, `vigencia_ini`, `vigencia_fim` devem ser validadas com dependências
- PowerDesigner **não replica automaticamente** as alterações na tabela de histórico; o DBA deve manter a consistência

💡 **Recomenda-se associar ambas as tabelas em um sub-modelo "Temporal Set" com documentação cruzada.**

---

### 🧯 7.7 Impactos operacionais e de conformidade

| Área              | Impacto potencial                                          |
|-------------------|------------------------------------------------------------|
| Auditoria         | Quebra de histórico pode invalidar evidências              |
| LGPD / Regulatórios | Perda de rastreabilidade de alterações em dados pessoais  |
| Diagnóstico        | Impossibilidade de reconstruir eventos passados            |
| Segurança          | Redução da capacidade de detectar ações suspeitas          |
| Homologação        | Necessidade de testagem extensiva após mudanças            |

---

### 📎 7.8 Glossário aplicado

- **Versionamento ativo**: Estado em que a tabela está com controle temporal habilitado
- **DROP VERSIONING**: Comando que desativa o versionamento e desconecta a tabela de histórico
- **DDL temporal**: Conjunto de operações DDL que afetam a definição de tabelas temporais
- **Histórico lógico**: Registro contínuo de versões dos dados em conformidade com System-Time

---

## 8. Boas Práticas e Cuidados Operacionais

### 📌 8.1 Por que boas práticas são críticas?

Tabelas temporais são **pilares da rastreabilidade e da integridade histórica**. Qualquer falha operacional pode comprometer:

- Conformidade com leis (ex: LGPD, Resolução CMN)
- Auditorias externas e internas
- Reconstrução de eventos
- Diagnóstico técnico em incidentes

Neste capítulo, reunimos práticas consolidadas para garantir robustez e confiabilidade.

---

### 🧱 8.2 Estrutura e Projeto

- 📌 **Planejar a temporalidade na fase de modelagem**
  - Avaliar se o dado exige controle `SYSTEM`, `BUSINESS`, ou ambos
  - Indicar explicitamente no PowerDesigner os períodos temporais
  - Documentar responsabilidade pela manutenção (`DB2` vs `Aplicação`)
  
- 🔐 **Separar claramente as tabelas de histórico**
  - Padronizar nomes: `nome_tabela_hist`
  - Definir tablespaces e storage classes distintos
  - Proteger contra escrita direta (ex: revoke insert/update/delete)

- 🧠 **Evitar overuse de temporalidade**
  - Não tornar "tudo temporal". Use para dados que justificam auditoria, vigência ou reversibilidade.
  - Avalie alternativas como journaling, CDC ou logs de aplicação

---

### 🚨 8.3 Cuidados com versionamento

- ✅ **Sempre manter a estrutura idêntica entre base e histórico**
  - Mesmo número, nome e ordem de colunas
  - Mesmo tipo de dado

- 🛑 **Nunca alterar colunas temporais diretamente com versionamento ativo**
  - Use DROP VERSIONING + alteração controlada + reativação

- 📋 **Documentar todas as operações DDL e reconfigurações**
  - Controle de versão de estrutura
  - Relatórios de alteração em histórico

- 🔁 **Realizar auditoria periódica da consistência do histórico**
  - Comparar número de versões por chave primária
  - Verificar datas sobrepostas, nulas ou inconsistentes

---

### 📊 8.4 Performance e indexação

- 🔍 **Indexar colunas de período (`row_end`, `vig_fim`) com inteligência**
  - Usar índices compostos com chaves de negócio
  - Validar planos de execução com `EXPLAIN`

- 🚫 **Evitar full scan em histórico desnecessariamente**
  - Consultas devem ser filtradas por tempo e chave sempre que possível

- 🧮 **Avaliar uso de MQTs (Materialized Query Tables)** para cenários de acesso recorrente e pesado

---

### 🧰 8.5 Operação e manutenção

- 🧯 **Tenha sempre plano de rollback para alterações estruturais**
  - Export lógico antes de qualquer DROP VERSIONING
  - Script de recriação da tabela com histórico intacto

- 📦 **Arquivar históricos obsoletos em storage frio**
  - Retenção pode seguir regra de compliance (ex: 5 anos)
  - Pode-se particionar por ano ou gerar unloads periódicos

- 🔄 **Automatizar testes de versionamento**
  - Scripts que inserem, atualizam e validam versionamento ativo
  - Confirmação da gravação automática no histórico

---

### 🧾 8.6 Auditoria e conformidade

- 📊 **Integrar com ferramentas de data lineage e auditoria**
  - Registrar quando dados foram modificados e por quem
  - Temporal Tables + LOG + RACF = rastreabilidade completa

- 🔒 **Controlar acesso à tabela de histórico**
  - Revoke direto para usuários e aplicações
  - Acesso apenas via views controladas

- 🧑‍⚖️ **Justificar cada uso de DROP VERSIONING**
  - Registrar motivo, impacto, responsável e validação posterior

---

### 🔎 8.7 Testes e homologação

- ✅ **Simular casos reais com retroatividade, vigência futura e múltiplas versões**
- 🧪 **Testar consultas temporais com todas as cláusulas (AS OF, BETWEEN, FROM TO)**
- 📚 **Reproduzir falhas anteriores e verificar se versão atual mitiga os riscos**

---

### 📎 8.8 Glossário aplicado

- **Versionamento ativo**: Tabela operando com histórico automático gerenciado pelo DB2
- **Data lineage**: Rastreabilidade de origem e transformações dos dados
- **Storage frio**: Camada de armazenamento usada para dados acessados com baixa frequência
- **Rollback temporal**: Capacidade de restaurar o estado de um dado com base em versão anterior
- **Consistency scan**: Processo de verificação da integridade temporal entre base e histórico

---

> **Nota:** A excelência na operação de temporal tables exige disciplina, padronização e sinergia entre modelagem, desenvolvimento, auditoria e administração de dados. No próximo fechamento, faremos a **análise final do material**, avaliando oportunidades de refinamento nos capítulos anteriores e consolidando diretrizes de uso.

---

## 9. Referências Oficiais

- [IBM Best Practices - Temporal Data](https://www.ibm.com/docs/en/db2-for-zos/13?topic=data-best-practices-temporal-tables)  
- [IBM Security & Auditing Guidelines](https://www.ibm.com/docs/en/db2-for-zos/13?topic=security-auditing-db2)  
- [IBM Redbooks: DB2 Temporal Tables](https://www.redbooks.ibm.com/abstracts/sg248079.html)  
- [IBM Documentation - DB2 13 for z/OS: System-period temporal tables](https://www.ibm.com/docs/en/db2-for-zos/13?topic=data-system-period-temporal-tables)  
- [IBM Redbooks: Managing Time-Based Data with Temporal Tables in DB2 for z/OS](https://www.redbooks.ibm.com/abstracts/sg248079.html)  
- [IBM SQL Reference](https://www.ibm.com/docs/en/db2-for-zos/13?topic=reference-sql-statements)  
- [Temporal Tables and Bitemporal Data](https://www.ibm.com/docs/en/db2-for-zos/13?topic=data-temporal-tables-bitemporal)

---

> **Nota:** Este conteúdo será continuamente refinado com base em práticas reais e documentação oficial. O próximo passo é aplicar o mesmo nível de refinamento aos capítulos 4 e 5. Caso queira iniciar por algum item específico, indique e avançamos com precisão.
