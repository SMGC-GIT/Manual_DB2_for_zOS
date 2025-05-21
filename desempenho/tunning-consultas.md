# 📁 Tuning e Performance no DB2 for z/OS (Detalhado)

## 🔍 Identificação de Problemas de Performance

Antes de otimizar, precisamos identificar exatamente onde está o problema. Abaixo estão os principais sintomas, formas de identificação, exemplos de SQLCODEs e como interpretá-los.

### 1. Alto Tempo de Resposta

- **O que é:** Consultas ou transações que demoram muito para retornar resultados, prejudicando o usuário final e o sistema.
- **Como identificar:**
  - Monitoramento do tempo médio de resposta no DB2 PM, SMF ou ferramentas de monitoramento.
  - Logs de aplicações mostrando tempos altos de execução.
  - EXPLAIN indicando Full Table Scan em tabelas grandes.
- **Possíveis causas:**  
  - Falta de índice adequado.  
  - Estatísticas desatualizadas (RUNSTATS não executado).  
  - SQL mal escrito (ex: uso excessivo de funções, SELECT *).  
- **SQLCODE relacionado:**  
  - Normalmente não gera SQLCODE de erro, mas o SQLCODE 0 indica sucesso com tempo potencialmente alto.

### 2. Excesso de CPU ou I/O

- **O que é:** Processos que consomem muito CPU ou I/O, impactando a performance geral do sistema.
- **Como identificar:**  
  - Análise dos gráficos de CPU e I/O via SMF, RMF ou monitoramento DB2 PM.  
  - IFCID 003 mostra lock waits e I/O elevados.  
  - Logs do sistema operacional.  
- **Possíveis causas:**  
  - Scans completos desnecessários (full table scans).  
  - Índices mal configurados ou ausentes.  
  - Consultas com joins ineficientes ou subqueries mal formuladas.  
- **SQLCODE relacionado:**  
  - SQLCODE 200 (row not found) pode indicar queries que varrem toda a tabela buscando algo que não existe.  
  - SQLCODEs negativos indicam erro, não diretamente relacionado a consumo.

### 3. Locking e Deadlocks

- **O que é:** Conflitos entre transações que acessam os mesmos recursos simultaneamente, levando a esperas ou abortos.
- **Como identificar:**  
  - IFCID 003 registra informações detalhadas de locks e deadlocks.  
  - Monitorar eventos de lock wait e deadlock no SMF.  
  - Mensagens de erro no sistema ou logs da aplicação.  
- **Erros/SQLCODE comuns:**  
  - **SQLCODE -911** (Deadlock ou timeout): Transação é terminada para resolver deadlock.  
  - **SQLCODE -913** (Lock timeout): Tempo limite excedido aguardando lock.  
- **Ações recomendadas:**  
  - Revisar lógica de transações para reduzir tempo de locks.  
  - Ajustar nível de isolamento, quando possível.  
  - Utilizar acesso por cursores estáticos para minimizar bloqueios.  

### 4. Wait Times Altos

- **O que é:** Tempos excessivos aguardando recursos, como I/O, CPU, locks, ou recursos do sistema.
- **Como identificar:**  
  - Monitoramento de performance para identificar tempos de espera (wait times).  
  - IFCID 014 e 015 podem ajudar a rastrear esperas.  
- **Possíveis causas:**  
  - Contenção por locks.  
  - Saturação de recursos como buffer pools, sort pools.  
  - Problemas no subsistema de armazenamento.  
- **SQLCODE:**  
  - Normalmente não gera SQLCODE específico, mas os problemas refletem em performance degradada.  

### 5. Planos de Acesso Ineficientes

- **O que é:** O otimizador escolhe um caminho de execução que não é o mais eficiente, resultando em scans desnecessários, muitos I/Os e CPU.
- **Como identificar:**  
  - Utilizar o EXPLAIN para analisar o plano de execução.  
  - Observar se o plano usa índices, ou se faz Full Table Scan.  
- **Indicadores no plano:**  
  - High-cost estimado.  
  - High number of rows acessados.  
- **SQLCODE:**  
  - Normalmente sucesso (SQLCODE 0), mas com performance ruim.

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

