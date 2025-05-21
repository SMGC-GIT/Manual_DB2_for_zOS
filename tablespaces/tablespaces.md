# 📁 Tablespaces no DB2 for z/OS

## 🧠 O que é um Tablespace?

No DB2 for z/OS, um **tablespace** é a estrutura física que armazena os dados das tabelas. Ele funciona como um container lógico que organiza os dados em páginas, usando arquivos LDS (Linear Data Sets) no z/OS. Cada tablespace pertence a um database (`DATABASE`) e é definido com parâmetros que controlam seu tamanho, tipo de alocação, compressão, codificação, entre outros.

---

## 🛠️ Exemplo Completo: Comando `CREATE TABLESPACE`

```sql
CREATE TABLESPACE TS_CLIENTES
  IN DATABASE DB_CLIENTES
  USING STOGROUP STG_DADOS
  PRIQTY 1000
  SECQTY 100
  BUFFERPOOL BP0
  MAXPARTITIONS 10
  SEGSIZE 32
  LOCKSIZE ROW
  CCSID EBCDIC;
```

---

## 🧩 Explicação Detalhada dos Parâmetros

### 🔹 `IN DATABASE DB_CLIENTES`

Define o **database lógico** ao qual o tablespace pertencerá. No DB2 for z/OS, um database agrupa um ou mais tablespaces. O database não armazena dados diretamente, mas organiza o catálogo.

---

### 🔹 `USING STOGROUP STG_DADOS`

Indica o **Storage Group** responsável pela alocação física do espaço em disco (VSAM LDS). O storage group é uma coleção de volumes definidos pelo DBA. O DB2 utiliza o STOGROUP para decidir onde armazenar os conjuntos de dados associados ao tablespace.

📌 *Para mais informações: consultar documentação oficial sobre* `STOGROUP`.

---

### 🔹 `PRIQTY 1000`

Define a quantidade de **páginas de espaço primário** (em unidades de página do buffer pool) que será alocada inicialmente. Neste caso, 1000 páginas.

📌 *Para mais informações: consultar documentação oficial sobre* `PRIQTY`.

---

### 🔹 `SECQTY 100`

Define a quantidade de **páginas de espaço secundário** que será alocada quando o espaço primário estiver cheio. É usada para crescer o tablespace dinamicamente.

📌 *Para mais informações: consultar documentação oficial sobre* `SECQTY`.

---

### 🔹 `BUFFERPOOL BP0`

Associa o tablespace a um **buffer pool**, que é uma área de memória utilizada para caching de páginas de dados e índices. 

- Exemplo: `BP0`, `BP8K0`, `BP16K0`, `BP32K`.
- O número indica o tamanho da página (em KB).
- Buffer pools podem ser ajustados para otimizar o desempenho.

📌 *Para mais informações: consultar documentação oficial sobre* `BUFFERPOOL`.

---

### 🔹 `MAXPARTITIONS 10`

Define o número máximo de partições permitidas para o tablespace. Este parâmetro é obrigatório para tablespaces **Partition-by-Growth (PBG)**.

- O DB2 cria novas partições automaticamente conforme os dados crescem.
- Cada partição é armazenada como um LDS separado.

📌 *Para mais informações: consultar documentação oficial sobre* `MAXPARTITIONS`.

---

### 🔹 `SEGSIZE 32`

Define o tamanho do **segmento** em páginas. Um segmento é a menor unidade de alocação de espaço para tabelas.

- Tabelas segmentadas armazenam dados de uma única tabela por segmento.
- Valor típico: 4, 8, 16, 32, 64.

📌 *Para mais informações: consultar documentação oficial sobre* `SEGSIZE`.

---

### 🔹 `LOCKSIZE ROW`

Controla o **nível de granularidade de bloqueio**. Neste exemplo, os locks são feitos a nível de linha.

Outros valores possíveis:
- `PAGE`
- `TABLESPACE`
- `TABLE`
- `ANY`

📌 *Para mais informações: consultar documentação oficial sobre* `LOCKSIZE`.

---

### 🔹 `CCSID EBCDIC`

Define o **conjunto de caracteres** (Coded Character Set ID) utilizado para armazenar os dados de caractere.

- Valores comuns: `EBCDIC`, `UNICODE`, `ASCII`
- Afeta a forma como os dados são codificados fisicamente.

📌 *Para mais informações: consultar documentação oficial sobre* `CCSID`.

---

## 📍 Observações Importantes

- A partir da versão **V12R1M504**, todos os novos tablespaces devem ser do tipo **Universal Table Space (UTS)**.
- O tipo PBG (Partition-by-Growth) é o mais comum em novas aplicações.
- Tablespaces antigos (simple e segmented) são considerados obsoletos.
- Alterações em parâmetros como `BUFFERPOOL` e `SEGSIZE` exigem **REORG** para aplicar.

---

## ✅ Boas Práticas com Tablespaces

- Utilize **UTS PBG** como padrão em novas tabelas.
- Defina **PRIQTY/SECQTY** com base no volume esperado de dados.
- Associe o tablespace a um **buffer pool otimizado** para o tipo de acesso esperado.
- Mantenha a granularidade de locks em `ROW` ou `PAGE` conforme o uso transacional.
- Use segmentação adequada (`SEGSIZE`) para facilitar a reorganização de dados.

---

## 📚 Para mais informações técnicas

Consulte os seguintes tópicos na documentação oficial da IBM:

- *Tablespace Structure:* https://www.ibm.com/docs/en/db2-for-zos/12.0.0?topic=structures-db2-table-spaces
- *Table space types and characteristics:* https://www.ibm.com/docs/en/db2-for-zos/12?topic=spaces-table-space-types-characteristics-in-db2-zos
- *Creating table spaces explicitly:* https://www.ibm.com/docs/en/db2-for-zos/12?topic=spaces-creating-table-explicitly
- *SYSTABLESPACE catalog table:* https://www.ibm.com/docs/en/db2-for-zos/13.0.0?topic=tables-systablespace


- 🔹 [Tablespaces - Conceitos e Tipos](https://www.ibm.com/docs/en/db2-for-zos/13?topic=objects-table-spaces)
  > Visão geral dos tipos de tablespaces e suas características principais (segmented, partitioned, universal).

- 🔹 [CREATE TABLESPACE - Sintaxe completa](https://www.ibm.com/docs/en/db2-for-zos/13?topic=statements-create-tablespace)
  > Descrição detalhada de todos os parâmetros do comando `CREATE TABLESPACE`.

- 🔹 [BUFFERPOOL - Como funciona e boas práticas](https://www.ibm.com/docs/en/db2-for-zos/13?topic=spaces-buffer-pool-selection)
  > Explica a seleção de bufferpool, critérios de desempenho e configuração.

- 🔹 [LOCKSIZE e LOCKMAX](https://www.ibm.com/docs/en/db2-for-zos/13?topic=statements-create-tablespace#sthref596)
  > Controle de concorrência e bloqueio de dados em tablespaces.

- 🔹 [SEGSIZE, DSSIZE e MAXROWS](https://www.ibm.com/docs/en/db2-for-zos/13?topic=statements-create-tablespace#sthref589)
  > Explicações sobre tamanhos de segmentação, tamanhos máximos e restrições físicas de armazenamento.

---

*Última atualização: 20/05/2025*
