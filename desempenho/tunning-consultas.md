# 🛠️ Tuning e Performance – DB2 for z/OS

Este documento tem como objetivo reunir práticas recomendadas, técnicas e comandos para identificação e resolução de problemas de performance no ambiente DB2 for z/OS, com foco na atuação de um DBA de desenvolvimento.

---

## 🔍 Identificação de Problemas de Performance

Antes de otimizar, precisamos identificar exatamente onde está o problema. Abaixo estão os principais sintomas, formas de identificação, exemplos de SQLCODEs e como interpretá-los.

---

## 🎯 Objetivos do Tuning

- Reduzir o tempo de resposta das queries
- Minimizar o uso de recursos (CPU, I/O, locks)
- Garantir escalabilidade e estabilidade das aplicações
- Reduzir riscos de abends e deadlocks
- Otimizar custos operacionais com MIPS

---

## 🔎 Como Identificar Problemas de Performance

### 1. Queries demoradas

- **Sintoma**: Aplicações com lentidão, principalmente em batch ou relatórios
- **Causa comum**: Falta de índice, acesso em table scan, joins mal otimizados
- **SQLCODEs relacionados**: 
  - `-911`: Timeout por lock
  - `-805`: Plano não encontrado
  - `-811`: Retorno de múltiplas linhas quando era esperada uma só

### 2. Uso excessivo de CPU

- **Sintoma**: Jobs com uso alto de CPU
- **Causa comum**: Loops desnecessários, joins cartesianas, má escolha de acesso
- **Ferramentas úteis**: SMF, monitoramento de pacotes, indicadores de bufferpool

### 3. Lock contention / deadlocks

- **Sintoma**: Jobs ou transações travando
- **Causa comum**: Falta de commit, escalonamento de locks
- **SQLCODEs**:
  - `-911`: Rollback automático por timeout
  - `-913`: Deadlock detectado

### 4. Falta de plano ou plano desatualizado

- **Sintoma**: Queries que antes eram rápidas agora lentas
- **Causa comum**: Estatísticas defasadas, alteração estrutural sem rebind
- **SQLCODEs**:
  - `-805`: DBRM não encontrado no pacote
  - `-818`: Timestamp mismatch entre plano e DBRM

---

## 📊 Ferramentas e Comandos Úteis

### RUNSTATS

Atualiza estatísticas de tabelas e índices, fundamentais para o otimizador:

```sql
RUNSTATS TABLESPACE DB1.TS1 TABLE(ALL) INDEX(ALL);
```

> 🔎 Execute após cargas, alterações de volume, reorganizações ou antes de REBINDs.

---

### REBIND

Força recompilação de pacotes, refletindo estatísticas atualizadas:

```sql
REBIND PACKAGE(COLLECTION.PACKAGE1) VALIDATE(BIND) EXPLAIN(YES);
```

---

### EXPLAIN

Mostra o plano de acesso gerado para instruções SQL. Usado via `EXPLAIN PLAN SET...` ou na BIND/REBIND com `EXPLAIN(YES)`. Tabelas como `PLAN_TABLE` armazenam o resultado.

> ✅ Fundamental para análise de índice utilizado, tipo de join, ordem de execução, acessos sequenciais.

---

### DISPLAY THREAD / MONITOR

Permite monitorar sessões ativas, locks, consumo de recursos:

```sql
-DISPLAY THREAD(*) TYPE(ACTIVE)
```

> Combine com IFCIDs via DB2 PM para detalhes aprofundados.

---

## 🧠 Dicas de Especialista

- ✅ Mantenha estatísticas atualizadas com RUNSTATS bem parametrizado.
- 🚫 Evite funções no WHERE que impeçam uso de índice.
- 🧮 Avalie a cardinalidade das colunas antes de criar índices.
- 📉 Cuidado com índices em colunas de baixa seletividade (ex: ‘sexo’, ‘UF’).
- 📐 Use índices compostos quando os filtros envolvem múltiplas colunas.
- 🛠️ Prefira `INNER JOIN` explícito, evite subqueries desnecessárias.
- 🔄 Recompile pacotes periodicamente e após alterações de estrutura ou carga.
- 📋 Mantenha scripts de tuning versionados com QUERYNO e comentários.

---


## 🛠️ EXPLAIN: Ferramenta Essencial para Análise de Performance

O EXPLAIN é fundamental para entender como o otimizador DB2 planeja executar o SQL. Ele revela o caminho escolhido, índices usados, tipos de acesso e estimativas de custo.

### Como funciona

- O comando EXPLAIN gera informações detalhadas sobre o plano de acesso antes da execução do SQL.  
- Essas informações são armazenadas em tabelas específicas do catálogo DB2 (EXPLAIN_INSTANCE, EXPLAIN_OBJECT, EXPLAIN_STATEMENT, etc).

### Comando básico:

```sql
EXPLAIN PLAN FOR
SELECT NOME, ENDERECO FROM CLIENTES WHERE ID = 123;

---

## 🔍 O que analisar no plano (EXPLAIN)

A ferramenta **EXPLAIN** permite gerar o plano de acesso do DB2 para uma instrução SQL. O resultado é armazenado em tabelas como `PLAN_TABLE`, `DSN_STATEMNT_TABLE`, entre outras.

### Principais colunas da PLAN_TABLE:

| Coluna           | Significado                                                                 |
|------------------|------------------------------------------------------------------------------|
| QUERYNO          | Número da query analisada                                                    |
| QBLOCKNO         | Número do bloco da query (subselects)                                        |
| PLANNO           | Ordem do passo no plano de execução                                          |
| METHOD           | Estratégia de join utilizada (NLJOIN, MERGE, HYBRID)                         |
| ACCESSNAME       | Nome do índice utilizado, se houver                                          |
| TABNAME          | Nome da tabela acessada                                                      |
| ACCESSTYPE       | Tipo de acesso (I = índice, R = scan, N = none)                              |
| MATCHCOLS        | Número de colunas do índice que foram utilizadas para busca                  |
| TIMESTAMP        | Momento em que o EXPLAIN foi executado                                       |

### Exemplo de uso do EXPLAIN:

```sql
EXPLAIN PLAN SET QUERYNO = 100 FOR
SELECT NOME, CPF FROM CLIENTES WHERE CPF = '12345678900';
```

Depois:

```sql
SELECT * FROM PLAN_TABLE WHERE QUERYNO = 100;
```

> ✅ Analisar `ACCESSTYPE`, `ACCESSNAME`, `MATCHCOLS` e `METHOD` ajuda a entender se o acesso está eficiente.

---

## 🧱 Exemplos de Índices Adequados

Um índice eficiente depende da **cardinalidade**, **distribuição de dados** e **seletividade** das colunas.

### Exemplo 1 – Índice para busca exata

```sql
CREATE INDEX IX_CLIENTES_CPF ON CLIENTES (CPF);
```

> Ideal para: `SELECT * FROM CLIENTES WHERE CPF = ?`

---

### Exemplo 2 – Índice composto com ordenação

```sql
CREATE INDEX IX_PEDIDOS_CLIENTE_DATA ON PEDIDOS (ID_CLIENTE, DATA_PEDIDO DESC);
```

> Ideal para: `SELECT * FROM PEDIDOS WHERE ID_CLIENTE = ? ORDER BY DATA_PEDIDO DESC`

---

### Exemplo 3 – Índice para evitar table scan em JOIN

```sql
CREATE INDEX IX_ITENS_IDPEDIDO ON ITENS_PEDIDO (ID_PEDIDO);
```

> Ajuda em joins: `SELECT ... FROM PEDIDOS P JOIN ITENS_PEDIDO I ON P.ID = I.ID_PEDIDO`

---

## 🧠 Dicas Valiosas de Especialista

- 🔄 **Recompile pacotes após mudanças em estatísticas (RUNSTATS)**.
- 📈 **Execute RUNSTATS** regularmente em tabelas com alta movimentação.
- 🚫 **Evite funções em colunas de filtros**: `WHERE YEAR(DATA_NASC) = 1980` impede uso de índice.
- 🧮 **Verifique cardinalidade com a coluna `CARDF` de SYSTABLES** para validar a seletividade dos índices.
- 🧪 **Compare planos antes e depois de alterações com QUERYNO diferente.**
- ⏱️ **Utilize MONITOR TABLES (IFCID) via DB2 Performance Monitor** para identificar gargalos em tempo real.

---

## ⚠️ Tabela de Problemas Comuns e SQLCODEs Associados

| Sintoma                             | Possível Causa                                     | SQLCODE Comum   |
|------------------------------------|----------------------------------------------------|-----------------|
| Alta latência em SELECT            | Falta de índice / índice ineficiente               | -805 (plano não encontrado), -811 (mais de 1 linha) |
| JOINs demorando muito              | Ordem de join incorreta / falta de estatísticas    | -500, -501      |
| Locks em excesso                   | Falta de COMMIT / concorrência elevada             | -911 (rollback) |
| Application timeout                | Escalonamento de lock, deadlock                    | -913 (timeout)  |
| Erro de plano não encontrado       | Pacote não recompilado após alteração              | -805            |

---

## 📌 Conclusão

Tuning de performance no DB2 for z/OS é um processo contínuo. Envolve análise profunda de planos, índices e estatísticas. A chave está em:

- 📊 Conhecer os dados (volume e distribuição)
- 🛠️ Criar índices estratégicos
- 🔍 Validar o plano com EXPLAIN
- 🔄 Manter estatísticas atualizadas
- ⚙️ Observar comportamentos reais de execução (monitoramento)

A excelência na performance depende da visão combinada entre DBA, desenvolvedor e o comportamento real das aplicações.

---

## 📚 Fontes Oficiais Recomendadas

- [IBM: Tuning Queries with EXPLAIN](https://www.ibm.com/docs/en/db2-for-zos/13?topic=statements-using-explain-tune-queries)
- [IBM: PLAN_TABLE structure](https://www.ibm.com/docs/en/db2-for-zos/13?topic=explain-plan-table)
- [IBM: Indexing Strategy Guide](https://www.ibm.com/docs/en/db2-for-zos/13?topic=indexes-creating-strategies)

---

*Última atualização: 21/05/2025*

