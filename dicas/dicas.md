## ⚡ Dicas de Performance para DBAs em DB2 – Parte 1

### 📌 1. **Evite o uso de `SELECT *`**

**❗ Por quê evitar?**  
- Recupera **todas** as colunas, mesmo que você não precise delas.
- Aumenta o volume de dados trafegado.
- Pode prejudicar o uso de índices cobrindo consultas (Index-only access).
- Torna o plano de acesso menos previsível e mais custoso.

**🔍 Exemplo prático:**
```sql
-- ❌ Evite isso
SELECT * FROM CLIENTES;

-- ✅ Use apenas o necessário
SELECT NOME, EMAIL, CIDADE FROM CLIENTES;
```

**💬 Observação:** Se novas colunas forem adicionadas, o `SELECT *` pode quebrar sua aplicação ou gerar resultados inesperados.

**🔗 Referência IBM:**  
[Writing efficient SQL queries – IBM Db2 12](https://www.ibm.com/docs/en/db2-for-zos/12.0.0?topic=performance-writing-efficient-sql-queries)

---

### 📌 2. **Use índices corretamente**

**❗ Por quê?**  
- Consultas com `WHERE`, `JOIN` e `ORDER BY` são aceleradas com bons índices.
- Evita *table scans* desnecessários.
- Permite uso de *Index-only access*, onde o DB2 nem precisa acessar a tabela.

**🔍 Exemplo de problema comum:**
```sql
-- ❌ Index ignorado pois função é aplicada à coluna
SELECT * FROM CLIENTES WHERE UPPER(NOME) = 'JOÃO';

-- ✅ Index utilizado corretamente
SELECT * FROM CLIENTES WHERE NOME = 'João';
```

**💬 Dica:**  
Evite funções nas colunas do `WHERE`, como `UPPER()`, `SUBSTR()`, `CAST()` — elas invalidam o uso de índices.

---

### 📌 3. **Evite subconsultas correlacionadas desnecessárias**

**❗ Por quê?**  
- São executadas para **cada linha** da consulta externa.
- Pode levar a tempo de execução alto em grandes volumes.

**🔍 Exemplo de substituição com JOIN:**
```sql
-- ❌ Subconsulta correlacionada (ineficiente)
SELECT C.NOME
FROM CLIENTES C
WHERE EXISTS (
  SELECT 1 FROM PEDIDOS P
  WHERE P.ID_CLIENTE = C.ID_CLIENTE
);

-- ✅ JOIN (mais eficiente)
SELECT DISTINCT C.NOME
FROM CLIENTES C
JOIN PEDIDOS P ON P.ID_CLIENTE = C.ID_CLIENTE;
```

**🔗 Referência IBM:**  
[Db2 12 - Writing efficient subqueries](https://www.ibm.com/docs/en/db2-for-zos/12.0.0?topic=queries-writing-efficient-subqueries)

---

### 📌 4. **Otimize consultas em tabelas particionadas**

**❗ Por quê?**  
- DB2 pode **eliminar partições** inteiras se você usar filtros corretos (Partition Elimination).
- Reduz drasticamente o volume de dados lido.

**🧠 Entenda:**  
Tabelas particionadas por DATA (por exemplo) exigem que sua query use `WHERE DATA_COLUNA BETWEEN...` para ativar o particionamento.

**🔍 Exemplo correto:**
```sql
-- ✅ Acesso eficiente a partições específicas
SELECT * FROM VENDAS
WHERE DATA_VENDA BETWEEN '2025-01-01' AND '2025-03-31';
```

**🔗 Referência IBM:**  
[Optimization strategies for partitioned tables](https://www.ibm.com/docs/en/db2/11.5.x?topic=strategies-optimization-partitioned-tables)

---

### 📌 5. **Evite conversões implícitas de tipos de dados**

**❗ Por quê?**  
- Impede o uso de índices.
- Aumenta o custo de CPU.
- Pode gerar resultados incorretos se a conversão não for o que você espera.

**🧠 O que é uma conversão implícita?**  
Quando você compara uma coluna `INTEGER` com um valor `VARCHAR`, o DB2 converte **automaticamente** (nem sempre de forma eficiente).

**🔍 Exemplo:**
```sql
-- ❌ Conversão implícita (DB2 converte CPF para string ou string para número)
SELECT * FROM CLIENTES WHERE CPF = '12345678900';

-- ✅ Tipos compatíveis: sem conversão, permite uso de índice
SELECT * FROM CLIENTES WHERE CPF = 12345678900;
```

**💬 Dica:**  
Garanta que seus literais e variáveis estejam com o **mesmo tipo de dado** da coluna, especialmente em `WHERE`, `JOIN`, `GROUP BY`.

**🔗 Referência complementar:**  
[Seven SQL Performance-Killers to Avoid in Db2 (Virtual-DBA)](https://virtual-dba.com/blog/seven-sql-performance-killers-avoid-db2/)

---

### 📌 6. **Otimize inserções em tabelas com grande volume de dados**

**❗ Por quê?**  
- Tabelas com alto volume de `INSERT` podem sofrer contenção e ter queda de performance.

**🧠 Estratégias sugeridas:**
- Use tablespaces com `MEMBER CLUSTER` para paralelismo.
- Utilize `INSERT ALGORITHM 2` (melhor para UTS – Universal Tablespaces).
- Avalie o uso de `APPEND YES`.

**🔍 Exemplo DDL:**
```sql
CREATE TABLESPACE VENDAS_TS
IN DATABASE PRODDB
USING STOGROUP STG1
BUFFERPOOL BP0
MEMBER CLUSTER YES
MAXROWS 255;
```

**🔗 Referência IBM:**  
[Insert algorithm 2 – Db2 for z/OS](https://www.ibm.com/docs/en/db2-for-zos/13?topic=performance-insert-algorithm-2)

---

### 📌 7. **Mantenha estatísticas atualizadas com RUNSTATS**

**❗ Por quê?**  
- O otimizador do DB2 depende dessas estatísticas para montar bons planos de execução.
- Estatísticas desatualizadas = planos ruins = performance ruim.

**🔍 Exemplo recomendado:**
```sql
RUNSTATS TABLESPACE DBNAME.TSNAME
TABLE (ALL)
SAMPLE 10 PERCENT
INDEX(ALL)
WITH DISTRIBUTION
REPORT YES;
```

**💬 Dica:**  
Agende `RUNSTATS` em janelas de baixa carga ou após grandes cargas de dados.

---

### 📌 8. **Evite LOCKs desnecessários e contenção**

**❗ Por que isso impacta a performance?**  
LOCKs mal gerenciados causam *waits*, *deadlocks*, *timeouts* e degradação geral da performance. Quanto mais tempo uma transação mantém os recursos travados, maior o risco de concorrência e gargalos.

**💡 Dicas práticas:**
- Use `WITH UR` (Uncommitted Read) para consultas somente leitura em tabelas estáveis.
- Em aplicações OLTP, mantenha as transações o mais curtas possível.
- Evite SELECTs muito longos ou complexos em tabelas que estão sendo frequentemente atualizadas.

**🔍 Exemplo:**
```sql
-- ✅ Leitura sem bloqueio, ideal para relatórios ou consultas não críticas
SELECT NOME, CPF FROM CLIENTES WITH UR;
```

**🔗 Referência IBM:**  
[Db2 concurrency control – IBM Docs](https://www.ibm.com/docs/en/db2-for-zos/13.0.0?topic=concurrency-controls-locking)

---

> ⚠️ **Importante:**  
> Sempre valide otimizações com base no contexto: volume de dados, tipo de workload, índices existentes e padrões de acesso. Uma estratégia que melhora performance em um cenário pode prejudicar em outro.

---

> 💡 **Dica bônus:**  
> Mantenha estatísticas atualizadas com `RUNSTATS` utilizando opções como `WITH DISTRIBUTION`, `INDEXES ALL`, `FREQVAL`, entre outras. Isso garante que o otimizador escolha os melhores planos de execução possíveis.

**🔗 Referência adicional:**  
[RUNSTATS Utility – IBM Docs](https://www.ibm.com/docs/en/db2-for-zos/13.0.0?topic=utilities-runstats-utility)

---


---

## ⚡ Dicas de Performance para DBAs em DB2 – Parte 1 (Continuação)

### 📌 9. **Evite o uso desnecessário de `DISTINCT`**

**🔎 Por que isso impacta a performance?**  
O uso do `DISTINCT` força o DB2 a processar e eliminar duplicatas. Isso geralmente exige uma ordenação interna e aumenta o uso de CPU e I/O, especialmente em grandes conjuntos de dados. Quando a unicidade já é garantida, seu uso é redundante e prejudica o desempenho.

**💡 Exemplo:**
```sql
-- ❌ Uso desnecessário de DISTINCT
SELECT DISTINCT NOME, CIDADE FROM CLIENTES;

-- ✅ Uso adequado
SELECT NOME, CIDADE FROM CLIENTES;
```

**📌 Dica prática:**  
Evite `DISTINCT` quando os dados retornados já forem únicos, por exemplo, quando há uma cláusula `GROUP BY` ou se as colunas consultadas compõem uma chave primária ou índice único.

---

### 📌 10. **Evite funções em colunas indexadas nas cláusulas `WHERE`**

**🔎 Por que isso impacta a performance?**  
Funções aplicadas a colunas em cláusulas `WHERE` ou `JOIN` impedem que o otimizador utilize índices, forçando varredura completa de tabela (table scan).

**💡 Exemplo:**
```sql
-- ❌ Índice não utilizado devido à função
SELECT * FROM VENDAS WHERE YEAR(DATA_VENDA) = 2025;

-- ✅ Índice utilizado corretamente
SELECT * FROM VENDAS WHERE DATA_VENDA BETWEEN '2025-01-01' AND '2025-12-31';
```

**📌 Dica prática:**  
Reescreva a consulta de forma que a função não afete a coluna indexada. Sempre que possível, compare diretamente os valores da coluna.

---

### 📌 11. **Use `WITH UR` (Uncommitted Read) em consultas de apenas leitura**

**🔎 Por que isso impacta a performance?**  
`WITH UR` permite ler dados ainda não confirmados por outras transações. Isso reduz o bloqueio de registros e melhora a concorrência em consultas analíticas ou de monitoramento, onde pequenas inconsistências são aceitáveis.

**💡 Exemplo:**
```sql
-- ✅ Consulta de leitura com menor bloqueio
SELECT NOME, CPF FROM CLIENTES WITH UR;
```

**📌 Dica prática:**  
Utilize com cautela: `WITH UR` é excelente para relatórios e dashboards, mas deve ser evitado em operações críticas, pois pode retornar dados desatualizados ou não confirmados.

---

### 📌 12. **Evite `ORDER BY` quando a ordenação não for necessária**

**🔎 Por que isso impacta a performance?**  
`ORDER BY` exige ordenação explícita no resultado, o que pode gerar alto custo computacional em grandes volumes de dados, mesmo quando a ordenação não é realmente usada no consumo final da consulta.

**💡 Exemplo:**
```sql
-- ❌ Ordenação desnecessária
SELECT NOME, CIDADE FROM CLIENTES ORDER BY NOME;

-- ✅ Sem ordenação quando não necessário
SELECT NOME, CIDADE FROM CLIENTES;
```

**📌 Dica prática:**  
Remova o `ORDER BY` se o front-end ou processo consumidor da query não depende da ordem dos registros. Isso melhora a performance e reduz uso de recursos.

---

### 📌 13. **Evite `SELECT COUNT(*)` em grandes tabelas sem filtro**

**🔎 Por que isso impacta a performance?**  
`COUNT(*)` em uma tabela sem cláusula `WHERE` varre todas as linhas para contar registros, consumindo tempo e recursos, especialmente em grandes volumes.

**💡 Exemplo:**
```sql
-- ❌ Contagem completa de linhas
SELECT COUNT(*) FROM VENDAS;

-- ✅ Alternativa: usar estatísticas do catálogo
SELECT CARD FROM SYSIBM.SYSTABLES WHERE NAME = 'VENDAS';
```

**📌 Dica prática:**  
Use `SYSIBM.SYSTABLES.CARD` para obter uma estimativa baseada em estatísticas atualizadas com `RUNSTATS`, quando possível. Para contagens precisas, filtre a consulta com `WHERE`.

---

### 📌 14. **Prefira `UNION ALL` quando a eliminação de duplicatas não for necessária**

**🔎 Por que isso impacta a performance?**  
`UNION` realiza uma ordenação e eliminação de duplicatas entre os conjuntos retornados. Já `UNION ALL` apenas concatena os resultados, sem esse processamento adicional.

**💡 Exemplo:**
```sql
-- ❌ Eliminação de duplicatas desnecessária
SELECT NOME FROM CLIENTES_BR
UNION
SELECT NOME FROM CLIENTES_AR;

-- ✅ Combinação mais eficiente
SELECT NOME FROM CLIENTES_BR
UNION ALL
SELECT NOME FROM CLIENTES_AR;
```

**📌 Dica prática:**  
Se você sabe que os conjuntos não possuem dados duplicados, ou se duplicatas são aceitáveis, use `UNION ALL` e ganhe em performance.

---

### 📌 15. **Evite `LIKE '%...%'` em grandes tabelas**

**🔎 Por que isso impacta a performance?**  
Consultas com `LIKE '%termo%'` desabilitam o uso de índices, forçando varreduras completas da tabela, o que é muito custoso em bases grandes.

**💡 Exemplo:**
```sql
-- ❌ Índice não é usado
SELECT * FROM PRODUTOS WHERE NOME LIKE '%caneta%';

-- ✅ Índice pode ser usado
SELECT * FROM PRODUTOS WHERE NOME LIKE 'caneta%';
```

**📌 Dica prática:**  
Evite o uso de `%` no início do padrão. Quando necessário buscar por partes de palavras, considere técnicas como:
- Índices de texto completo (text search)
- Criação de colunas auxiliares com palavras-chave
- Ferramentas como DB2 Text Search ou soluções externas (ElasticSearch)

---

> 📘 **Importante:**  
> Cada uma dessas dicas foi selecionada para facilitar melhorias rápidas de performance sem grandes mudanças estruturais. Elas funcionam como "ajustes finos", ideais para o dia a dia de um DBA que busca ganhar eficiência com pequenas modificações.

---
