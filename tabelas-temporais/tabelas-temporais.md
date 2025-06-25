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

### 📌 4.1 O que é?

**Business-Time Temporal Table** é um tipo de tabela que registra a **validade de uma informação com base em regras de negócio**, independentemente do momento em que foi inserida ou alterada no banco de dados.

Enquanto as *System-Time Tables* tratam do **quando o dado foi modificado no sistema**, as *Business-Time Tables* tratam do **período em que o dado é considerado válido para fins de negócio**.

### 💡 Exemplos de uso

- Vigência de contratos e cláusulas
- Alterações tarifárias com data futura
- Planejamento de tabelas de preços
- Regras normativas com efeito retroativo

---

### 🎯 4.2 Por que usar?

- Permite **antecipar alterações** com data futura
- Suporta **retificações retroativas** (ex: legislação ou erro)
- Melhora **conformidade com regras de negócio** e exigências jurídicas
- Habilita **análises temporais** como "Qual seria o valor em vigor em 01/07/2024?"

---

### 🔬 4.3 Como funciona?

#### Intervalo de validade

Duas colunas são utilizadas para indicar a vigência da linha:

| Coluna     | Significado                         |
|------------|--------------------------------------|
| `valid_start` | Data de início da vigência da linha |
| `valid_end`   | Data de fim da vigência da linha    |

Essas colunas podem ser **do tipo DATE ou TIMESTAMP**, conforme a granularidade desejada.

#### Declaração temporal

```sql
PERIOD BUSINESS_TIME (valid_start, valid_end)
```

Diferente das system-time, o **banco não preenche nem atualiza os valores** dessas colunas. A **aplicação é responsável** por definir os períodos de validade.

---

### 🛠️ 4.4 Exemplo prático

```sql
CREATE TABLE tarifa (
   id_tarifa     INTEGER NOT NULL,
   valor         DECIMAL(10,2),
   vigencia_ini  DATE NOT NULL,
   vigencia_fim  DATE NOT NULL,
   PERIOD BUSINESS_TIME (vigencia_ini, vigencia_fim),
   PRIMARY KEY (id_tarifa, vigencia_ini)
);
```

#### Inserções com datas futuras

```sql
-- Tarifa em vigor a partir de 2025
INSERT INTO tarifa VALUES (1, 150.00, '2025-01-01', '9999-12-31');
```

#### Atualização com divisão de vigência

```sql
-- Atualizar valor a partir de 2026, sem apagar histórico
UPDATE tarifa
SET vigencia_fim = '2025-12-31'
WHERE id_tarifa = 1 AND vigencia_ini = '2025-01-01';

INSERT INTO tarifa VALUES (1, 170.00, '2026-01-01', '9999-12-31');
```

---

### 🧠 4.5 Como modelar no PowerDesigner

Em modelos conceituais e lógicos:

- Criar dois atributos: `vigencia_ini`, `vigencia_fim` ou equivalente
- Anotar como "Business Validity Period"
- Definir claramente que a aplicação controlará os valores

No modelo físico:

- Usar `DATE` ou `TIMESTAMP(12)` conforme necessidade
- Criar **constraints de integridade temporal** (não nativas)
- Declarar `PERIOD BUSINESS_TIME (...)` no campo "Check Parameters" ou "Table Options"
- Simular indexação por período, se necessário

---

### 🔎 4.6 Considerações técnicas

| Aspecto | Observações |
|--------|-------------|
| Controle de vigência | Aplicação define. DB2 **não atualiza automaticamente** os períodos |
| Integridade | Deve-se evitar sobreposição de períodos com chaves compostas ou constraints |
| Performance | Avaliar uso de **índices compostos** incluindo colunas de vigência |
| Exposição de dados | Usar views para controlar que apenas registros "vigentes" sejam mostrados |

---

### 📎 4.7 Glossário aplicado

- **Business Time**: Intervalo definido pela aplicação para refletir a validade de uma informação
- **PERIOD BUSINESS_TIME**: Comando SQL que marca duas colunas como intervalo temporal de negócio
- **Retroatividade**: Possibilidade de corrigir dados anteriores com nova versão sobreposta
- **Vigência**: Período em que determinada regra ou valor é válida sob o ponto de vista do negócio

---

## 5. Bitemporal Tables (System + Business Time)

(Conteúdo será aprimorado no próximo ciclo conforme novo padrão)

---

## 6. Consultas Temporais

(Conteúdo será aprimorado no próximo ciclo conforme novo padrão)

---

## 7. Alterações na Estrutura e Impactos

(Conteúdo será aprimorado no próximo ciclo conforme novo padrão)

---

## 8. Boas Práticas e Cuidados Operacionais

(Conteúdo será aprimorado no próximo ciclo conforme novo padrão)

---

## 9. Referências Oficiais

- [IBM Documentation - DB2 13 for z/OS: System-period temporal tables](https://www.ibm.com/docs/en/db2-for-zos/13?topic=data-system-period-temporal-tables)  
- [IBM Redbooks: Managing Time-Based Data with Temporal Tables in DB2 for z/OS](https://www.redbooks.ibm.com/abstracts/sg248079.html)  
- [IBM SQL Reference](https://www.ibm.com/docs/en/db2-for-zos/13?topic=reference-sql-statements)  
- [Temporal Tables and Bitemporal Data](https://www.ibm.com/docs/en/db2-for-zos/13?topic=data-temporal-tables-bitemporal)

---

> **Nota:** Este conteúdo será continuamente refinado com base em práticas reais e documentação oficial. O próximo passo é aplicar o mesmo nível de refinamento aos capítulos 4 e 5. Caso queira iniciar por algum item específico, indique e avançamos com precisão.
