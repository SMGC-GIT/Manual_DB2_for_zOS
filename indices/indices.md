# 📘 Diagnóstico com Foco em Índices no DB2 for z/OS

---

## 📑 Índice

- [🎯 Objetivo](#-objetivo)
- [🔍 1. Avaliação Inicial dos Índices](#-1-avaliação-inicial-dos-índices)
- [🔎 2. Diagnóstico de Eficiência do Índice](#-2-diagnóstico-de-eficiência-do-índice)
  - [2.1 Matching Columns](#21-matching-columns)
  - [2.2 Index Screening](#22-index-screening)
  - [2.3 Index Only Access](#23-index-only-access)
- [🛑 3. Más Práticas em Índices](#-3-más-práticas-em-índices)
- [🧠 4. Estratégias Avançadas](#-4-estratégias-avançadas)
  - [4.1 Uso do INCLUDE](#41-uso-do-include)
  - [4.2 Índices Compostos com Seletividade Alta](#42-índices-compostos-com-seletividade-alta)
  - [4.3 Índices para Ordenação (ORDER BY)](#43-índices-para-ordenação-order-by)
  - [4.4 Índices para Joins](#44-índices-para-joins)
- [🧾 5. Como Validar a Utilização do Índice](#-5-como-validar-a-utilização-do-índice)
- [🧠 6. Exemplos Didáticos de Criação de Índices Eficientes](#-6-exemplos-didáticos-de-criação-de-índices-eficientes)
- [📎 Referências Técnicas IBM](#-referências-técnicas-ibm)

---

## 🎯 Objetivo

Índices são o principal instrumento de performance em bancos relacionais. Um índice mal projetado, excessivo ou ausente pode ser a causa central de degradação de performance. Esta seção cobre os critérios técnicos e estratégicos para **avaliar, criar e ajustar índices em DB2 for z/OS**, com foco em ambientes de alta criticidade.

---

## 🔍 1. Avaliação Inicial dos Índices

Antes de qualquer ação, pergunte:

- Existe algum índice que **cobre todas as colunas** dos filtros da query?
- O índice **tem a ordenação correta** (posição das colunas)?
- Há **colunas com baixa seletividade** no início do índice?
- A query precisa de colunas **não indexadas no SELECT**?
- Existem índices que **nunca são utilizados** (ver MONGETINDEX)?

---

## 🔎 2. Diagnóstico de Eficiência do Índice

### 2.1 Matching Columns

Refere-se ao número de colunas de um índice que correspondem exatamente aos filtros usados na query.

> Quanto maior o número de **matching columns**, menor a quantidade de páginas acessadas.

---

### 2.2 Index Screening

Quando a filtragem é feita parcialmente via índice. Apenas a **primeira coluna do índice** casa com o filtro; as demais são filtradas após a leitura inicial.

- Menos eficiente que matching, mas melhor que table scan.
- Evite funções nas colunas que participam de índices.

---

### 2.3 Index Only Access

Ocorre quando **todas as colunas da query** (filtros + retornos) estão no índice.

- O DB2 resolve tudo sem acessar a tabela base.
- Reduz drasticamente o I/O.
- Pode ser viabilizado com `INCLUDE`.

---

## 🛑 3. Más Práticas em Índices

- Colocar `STATUS`, `TIPO_REGISTRO` como primeira coluna (baixa seletividade).
- Índices com muitas colunas → degradam `INSERT`, `UPDATE`.
- Índices redundantes ou criados por tentativa e erro.
- Ausência de `INCLUDE` para viabilizar **Index Only Access**.
- Criar índice apenas para `ORDER BY`, sem avaliar o custo real.

---

## 🧠 4. Estratégias Avançadas

### 4.1 Uso do INCLUDE

Permite adicionar colunas não-chave ao índice. Essas colunas:
- Não participam da ordenação;
- Permitem **responder à query sem acessar a tabela base**;
- Ideal para queries que **filtram por uma coluna, mas retornam outras**.

---

### 4.2 Índices Compostos com Seletividade Alta

- A ordem das colunas **deve respeitar a seletividade**.
- Coloque as colunas **mais seletivas primeiro**.
- Isso reduz o conjunto de dados acessado no índice.

---

### 4.3 Índices para Ordenação (ORDER BY)

- Se a query usar `ORDER BY`, criar índice com **mesma ordem das colunas** permite **eliminar sort**.
- Importante em tabelas grandes, onde sort consome workfile e I/O.

---

### 4.4 Índices para Joins

- O campo usado no `JOIN` deve estar indexado nas tabelas envolvidas.
- Otimiza o método de acesso (Nested Loop, Hybrid, etc.).
- Avalie o tipo e seletividade da relação.

---

## 🧾 5. Como Validar a Utilização do Índice

- Use `EXPLAIN(YES)` no `REBIND PACKAGE` ou via ferramenta de análise.
- Verifique colunas como:
  - `ACCESSNAME`: nome do índice utilizado.
  - `MATCHCOLS`: colunas que casaram com o filtro.
  - `METHOD`: tipo de acesso (1 = index, 0 = TS Scan, etc).

---

## 🧠 6. Exemplos Didáticos de Criação de Índices Eficientes

### ✅ Exemplo 1: Índice para filtros + ordenação + Index Only Access

**Query:**
```sql
SELECT DATA_VENDA, VALOR_TOTAL, STATUS
FROM TB_VENDA
WHERE COD_LOJA = '0051'
ORDER BY DATA_VENDA DESC
```

**Índice sugerido:**
```sql
CREATE INDEX IDX_VENDA_LOJA_DATA 
  ON TB_VENDA (COD_LOJA, DATA_VENDA DESC)
  INCLUDE (VALOR_TOTAL, STATUS)
```

📌 *Por que?*  
- `COD_LOJA` + `DATA_VENDA` = filtro + ordenação → elimina sort.  
- `VALOR_TOTAL`, `STATUS` = retornadas no SELECT → viabilizam **Index Only Access**.

---

### ✅ Exemplo 2: Índice para JOIN e filtro

**Query:**
```sql
SELECT P.ID_PEDIDO, P.DATA_PEDIDO
FROM TB_PEDIDO P
JOIN TB_CLIENTE C ON P.ID_CLIENTE = C.ID_CLIENTE
WHERE C.TIPO_CLIENTE = 'PJ'
```

**Índices sugeridos:**
```sql
-- Suporte ao JOIN
CREATE INDEX IDX_PEDIDO_CLIENTE 
  ON TB_PEDIDO (ID_CLIENTE);

-- Suporte ao filtro + JOIN
CREATE INDEX IDX_CLIENTE_TIPO 
  ON TB_CLIENTE (TIPO_CLIENTE, ID_CLIENTE);
```

📌 *Por que?*  
- Indexa campo de join e melhora cardinalidade.  
- `TIPO_CLIENTE` tem alta seletividade, otimizando o acesso inicial.

---

### ✅ Exemplo 3: Evitando função em Stage 2

**Antes (ineficiente):**
```sql
WHERE MONTH(DATA_PAGAMENTO) = 6
```

**Otimizado (Stage 1):**
```sql
WHERE DATA_PAGAMENTO BETWEEN '2024-06-01' AND '2024-06-30'
```

**Índice:**
```sql
CREATE INDEX IDX_PAGTO_DATA 
  ON TB_PAGAMENTOS (DATA_PAGAMENTO);
```

📌 *Por que?*  
- Permite filtro direto no índice → **Stage 1 access**.  
- Melhora significativamente queries com range de datas.

---

### ✅ Exemplo 4: Índice para evitar sort com múltiplos ORDER BY

**Query:**
```sql
SELECT NOME, UF, CIDADE
FROM TB_CLIENTES
WHERE UF = 'SP'
ORDER BY UF, CIDADE, NOME
```

**Índice:**
```sql
CREATE INDEX IDX_CLIENTE_ORD 
  ON TB_CLIENTES (UF, CIDADE, NOME);
```

📌 *Por que?*  
- Cobertura total da ordenação → elimina sort.  
- Se SELECT usar apenas colunas do índice, permite **Index Only Access**.

---

### ✅ Exemplo 5: Índice para filtros combinados com alta seletividade

**Query:**
```sql
SELECT *
FROM TB_TRANSACAO
WHERE TIPO_TRANSACAO = 'C' AND COD_CANAL = 'APP'
```

**Índice:**
```sql
CREATE INDEX IDX_TRANSACAO_TIPO_CANAL 
  ON TB_TRANSACAO (TIPO_TRANSACAO, COD_CANAL);
```

📌 *Por que?*  
- A combinação é seletiva; índice composto permite **filtragem mais precisa**.  
- Estatísticas com `COLGROUP` ajudam o otimizador a estimar corretamente.

---

## 📎 Referências Técnicas IBM

- [Db2 for z/OS Indexing Overview](https://www.ibm.com/docs/en/db2-for-zos/13?topic=design-indexing-overview)
- [Db2 13 Index Design - Redbook](https://www.redbooks.ibm.com/abstracts/sg248551.html)
- [IBM Documentation – CREATE INDEX](https://www.ibm.com/docs/en/db2-for-zos/13?topic=statements-create-index)

---

