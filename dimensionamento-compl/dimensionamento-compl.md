# 🧮 Manual Completo de Dimensionamento de Tabelas no DB2 for z/OS

> Documento técnico consolidado com foco em **cálculo de espaço**, **modelagem eficiente**, **simulações práticas** e **explicações detalhadas** sobre o funcionamento interno do DB2 for z/OS.

---

## 📚 Índice

1. [🔰 Introdução](#🔰-introdução)  
2. [📦 Estrutura e Conceitos Fundamentais](#📦-estrutura-e-conceitos-fundamentais)  
3. [📐 Como Calcular o Tamanho da Linha](#📐-como-calcular-o-tamanho-da-linha)  
4. [🧮 Fórmulas Técnicas de Cálculo](#🧮-fórmulas-técnicas-de-cálculo)  
5. [📝 Simulação Prática de Cálculo](#📝-simulação-prática-de-cálculo)  
6. [🧾 Simulação Avançada com LOBs](#🧾-simulação-avançada-com-lobs)  
7. [🧠 Explicações Técnicas Detalhadas](#🧠-explicações-técnicas-detalhadas)  
8. [✅ Boas Práticas de Dimensionamento](#✅-boas-práticas-de-dimensionamento)  
9. [📊 Abas Sugeridas para Planilha](#📊-abas-sugeridas-para-planilha)  
10. [🔗 Referências Técnicas IBM](#🔗-referências-técnicas-ibm)

---

## 🔰 Introdução

O dimensionamento correto de tabelas no **DB2 for z/OS** é essencial para:

- Planejamento de espaço em disco e buffer pools  
- Eficiência de I/O  
- Escolha de compressão e particionamento  
- Garantia de performance e escalabilidade

Este manual é voltado a DBAs e profissionais de dados em ambientes críticos como o da **Caixa Econômica Federal**.

---

## 📦 Estrutura e Conceitos Fundamentais

| Termo                  | Definição                                                                 |
|------------------------|--------------------------------------------------------------------------|
| 📄 Linha (Row)         | Conjunto de colunas que forma um registro                                |
| 📌 Coluna              | Campo de dados com tipo e restrições                                     |
| 📐 Tamanho da Linha    | Soma dos bytes das colunas + overhead                                    |
| 💽 Tablespace          | Área física onde as páginas são armazenadas                              |
| 🧱 Página (Page)       | Unidade de armazenamento no buffer pool (4K, 8K, 16K, 32K)                |
| 🧷 LOB (Large Object)  | Dados extensos como CLOB ou BLOB                                          |
| 🎯 Bitmap Nullability  | Mapa binário que define colunas NULLABLE                                 |
| 🧩 Overhead            | Espaço adicional para controle do DB2                                    |

---

## 📐 Como Calcular o Tamanho da Linha

### 📊 Tamanhos por Tipo de Dado

| Tipo de Dado     | Tamanho Estimado (bytes) | Observações Técnicas                              |
|------------------|---------------------------|---------------------------------------------------|
| `CHAR(n)`        | n                         | Tamanho fixo                                      |
| `VARCHAR(n)`     | n + 2                     | Inclui 2 bytes extras para controle               |
| `INTEGER`        | 4                         | Valor inteiro                                     |
| `SMALLINT`       | 2                         | Inteiro menor                                     |
| `BIGINT`         | 8                         | Inteiro de 64 bits                                |
| `DECIMAL(p,s)`   | (p/2)+1                   | Formato packed decimal                            |
| `DATE`           | 4                         | AAAAMMDD                                          |
| `TIME`           | 3                         | HHMMSS                                            |
| `TIMESTAMP`      | 10                        | YYYY-MM-DD-HH.MM.SS.MMMMMM                        |
| `CLOB/BLOB`      | ~20 (ponteiro)            | Armazenado externamente, ponteiro na linha base   |

---

## 🧮 Fórmulas Técnicas de Cálculo

```text
TAMANHO DA LINHA = 
  Soma dos tamanhos das colunas fixas
+ (2 * Nº de VARCHARs)
+ Overhead da Linha (8–12 bytes)
+ Null Bitmap (1 byte para até 8 colunas NULLABLE)
+ Alinhamento (Padding: 0–4 bytes, variável)
```

### 📏 Linhas por Página

```text
LINHAS POR PÁGINA =
  (PAGE SIZE - PAGE HEADER) / ROW LENGTH
```

> Exemplo: página de 4KB (4096 bytes), header ~128 bytes, linha = 41 bytes  
> (4096 - 128) / 41 ≈ 96 linhas por página

---

## 📝 Simulação Prática de Cálculo

| Coluna             | Tipo de Dado   | Tamanho Estimado | Observações                          |
|--------------------|----------------|------------------|--------------------------------------|
| ID_CLIENTE         | INTEGER        | 4                |                                      |
| CPF                | CHAR(11)       | 11               |                                      |
| NOME               | VARCHAR(50)    | 52               | 50 + 2 bytes de controle             |
| IDADE              | SMALLINT       | 2                |                                      |
| NASCIMENTO         | DATE           | 4                |                                      |
| ATIVO              | CHAR(1)        | 1                |                                      |
| SALDO              | DECIMAL(9,2)   | 5                | 9 dígitos packed                     |
| HORA_CADASTRO      | TIME           | 3                |                                      |
| ULTIMA_ATUALIZACAO | TIMESTAMP      | 10               |                                      |
| EMAIL              | VARCHAR(100)   | 102              | 100 + 2 bytes                        |
| TELEFONE           | VARCHAR(20)    | 22               | 20 + 2 bytes                         |
| OBSERVACOES        | CLOB(1K)       | ~20              | Armazenado fora da linha             |

### 🔢 Total estimado:
```
Total = 4 + 11 + 52 + 2 + 4 + 1 + 5 + 3 + 10 + 102 + 22 + 20 = 236 bytes
Overhead estimado: 12 bytes
Null Bitmap (4 cols nullable): 1 byte

Total final: 236 + 12 + 1 = **249 bytes**
```

---

## 🧾 Simulação Avançada com LOBs

| Coluna    | Tipo        | Tamanho Estimado | Armazenamento       | Observação                           |
|-----------|-------------|------------------|----------------------|--------------------------------------|
| ID_DOC    | INTEGER     | 4                | Inline               |                                      |
| TITULO    | VARCHAR(255)| 257              | Inline               | 255 + 2 bytes                        |
| DESCRICAO | CLOB(10K)   | ~20              | Fora da linha (LOBMAP)| Ponteiro apenas                      |
| ANEXO_PDF | BLOB(5MB)   | ~20              | Fora da linha (LOBMAP)| Ponteiro apenas                      |

---

## 🧠 Explicações Técnicas Detalhadas

### 🔳 Bitmap Nullability

- Estrutura interna usada pelo DB2 para marcar colunas NULL.
- 1 bit por coluna NULLABLE, agrupados em bytes.
- Evita verificação individual de NULL nas colunas.

### 🧩 Overhead da Linha

- Tamanho médio: 8 a 12 bytes.
- Inclui row header, status de bloqueio, ponteiros internos.
- Aumenta com RRF (Row Format) e compressão.

### 📚 Directory Pages & Tablespaces

- Cada tablespace contém várias páginas de dados.
- Directory pages mantêm metadados para navegação rápida.
- Importante para performance de I/O e busca por RID.

### 🧯 Compressão de Dados

| Tipo de Compressão | Aplicável a | Redução estimada | Observações                        |
|--------------------|-------------|------------------|------------------------------------|
| Row-level          | Linha       | 30–70%           | Boa para dados repetitivos         |
| Page-level         | Página      | Média            | Usa dicionários locais             |
| Huffman            | LOBs        | Alta             | Dados binários                     |

> 💡 Compressão reduz uso de disco e melhora fetches do buffer pool.

---

## ✅ Boas Práticas de Dimensionamento

- [x] Use `SMALLINT` em vez de `INTEGER` quando possível  
- [x] Use `VARCHAR` apenas com estimativa realista  
- [x] Minimize colunas `NULLABLE`  
- [x] Estime overheads: null bitmap + header + padding  
- [x] Considere compressão para tabelas > 1 milhão de linhas  
- [x] Planeje particionamento para consultas massivas ou com range  

---

## 📊 Abas Sugeridas para Planilha

| Aba                    | Descrição                                                    |
|------------------------|--------------------------------------------------------------|
| `01_TiposDeDados`      | Tabela com tamanhos padrão por tipo (CHAR, VARCHAR etc.)     |
| `02_ModeloTabela`      | Nome, tipo, tamanho e nullability de cada coluna             |
| `03_CalculoLinha`      | Soma de tamanhos, overheads, bitmap, alinhamento             |
| `04_LinhasPorPagina`   | Cálculo com base no page size (4K, 8K etc.)                  |
| `05_EspacoTotal`       | Estimativa total da tabela em MB/GB                          |
| `06_AnaliseMelhoria`   | Sugestões automáticas para otimização                        |
| `07_LOBsDetalhados`    | Controle de colunas LOB e seus tamanhos                      |

---

## 🔗 Referências Técnicas IBM

- 📘 [Estimating Row Size and Table Space Usage](https://www.ibm.com/docs/en/db2-for-zos/12?topic=design-estimating-row-size-table-space-usage)  
- 📘 [DB2 for z/OS SQL Reference](https://www.ibm.com/docs/en/db2-for-zos/12?topic=sql-reference)  
- 📘 [IBM Knowledge Center – Data Compression](https://www.ibm.com/docs/en/db2-for-zos/12?topic=performance-data-compression)  
- 📘 [IBM Redbooks – DB2 for z/OS Administration](https://www.redbooks.ibm.com/abstracts/sg247821.html)  

---

> 💡 *Este manual foi consolidado com base em fontes oficiais da IBM, experiência prática e engenharia reversa de ambientes produtivos de missão crítica.*
