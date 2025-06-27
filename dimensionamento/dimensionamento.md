# Manual Completo de Dimensionamento de Tabelas no DB2 for z/OS

---

## Índice

1. [Introdução](#introdução)  
2. [Conceitos Fundamentais](#conceitos-fundamentais)  
3. [Como Calcular o Tamanho da Linha](#como-calcular-o-tamanho-da-linha)  
4. [Simulação Prática: Exemplo de Cálculo](#simulação-prática-exemplo-de-cálculo)  
5. [Simulação Avançada: Tabela com LOBs](#simulação-avançada-tabela-com-lobs)  
6. [Explicações Técnicas Importantes](#explicações-técnicas-importantes)  
7. [Boas Práticas de Dimensionamento](#boas-práticas-de-dimensionamento)  
8. [Referências Oficiais IBM](#referências-oficiais-ibm)  

---

# 1. Introdução

Este manual tem como objetivo fornecer um guia técnico e didático para **dimensionar tabelas no ambiente DB2 for z/OS**, focando no cálculo do tamanho da linha e da tabela, baseado em dados confiáveis da IBM e práticas de mercado. É destinado a DBAs e profissionais de dados que atuam em ambientes produtivos, como na Caixa Econômica Federal (CEF).

Dimensionar adequadamente uma tabela é fundamental para:
- Planejar o espaço em tablespaces.
- Decidir pela necessidade de compressão.
- Avaliar a possibilidade de particionamento.
- Otimizar a performance e evitar gargalos de I/O.

---

# 2. Conceitos Fundamentais

- **Linha (Row):** unidade lógica que armazena os dados para um registro da tabela.
- **Coluna (Column):** campo da tabela que contém dados de um tipo específico.
- **Tamanho da linha:** soma do tamanho em bytes de todas as colunas, mais overhead.
- **Tablespace:** espaço físico no DB2 onde as tabelas são armazenadas.
- **LOB (Large Object):** tipos de dados que armazenam grandes volumes (ex: CLOB, BLOB).
- **Bitmap Nullability:** estrutura que controla quais colunas podem conter valores NULL.
- **Overhead:** bytes extras para controle interno do DB2 (null bitmap, locks, etc.).

---

# 3. Como Calcular o Tamanho da Linha

O tamanho da linha é a soma dos tamanhos estimados de cada coluna mais o overhead associado.

### Tamanhos Estimados por Tipo de Dado

| Tipo de Dado     | Tamanho Padrão                 | Observação Técnica                                               |
|------------------|--------------------------------|------------------------------------------------------------------|
| CHAR(n)          | n                              | Tamanho fixo, exatamente n bytes                                 |
| VARCHAR(n)       | n + 2                          | n bytes + 2 bytes para controle do comprimento                   |
| INTEGER          | 4                              | Inteiro padrão, 4 bytes                                          |
| SMALLINT         | 2                              | Inteiro pequeno, 2 bytes                                        |
| BIGINT           | 8                              | Inteiro grande, 8 bytes                                         |
| DECIMAL(p,s)     | (p / 2) + 1                    | Packed decimal; arredondado para cima                            |
| DATE             | 4                              | Formato AAAAMMDD, 4 bytes                                       |
| TIME             | 3                              | Formato HHMMSS, 3 bytes                                         |
| TIMESTAMP        | 10                             | Data e hora, 10 bytes                                           |
| CLOB/BLOB        | 2 ou 4                         | Ponteiro no registro base; dados armazenados fora da linha      |

### Considerações

- Para colunas do tipo VARCHAR, considere o tamanho máximo ou uma média realista para o dimensionamento.
- CLOBs e BLOBs armazenam apenas um ponteiro na linha base, os dados reais ficam em tablespaces auxiliares.
- Overhead típico por linha varia entre 10 e 12 bytes, incluindo bitmap nullability e controles internos.
- Bitmap Nullability consome 1 bit por coluna que permite NULL, agrupados em bytes para até 8 colunas por byte.

---

# 4. Simulação Prática: Exemplo de Cálculo

| Coluna             | Tipo de Dado     | Tamanho (bytes) | Observação                                                   |
|--------------------|------------------|-----------------|--------------------------------------------------------------|
| ID_CLIENTE         | INTEGER          | 4               | Integer padrão                                               |
| CPF                | CHAR(11)         | 11              | Tamanho fixo                                                |
| NOME               | VARCHAR(50)      | 52              | 2 bytes de controle + 50 bytes                               |
| IDADE              | SMALLINT         | 2               | Pequeno inteiro                                             |
| NASCIMENTO         | DATE             | 4               | Formato padrão                                              |
| ATIVO              | CHAR(1)          | 1               | Flag S/N                                                   |
| SALDO              | DECIMAL(9,2)     | 5               | Packed decimal (9 dígitos, 2 decimais)                      |
| HORA_CADASTRO      | TIME             | 3               | Hora no formato HHMMSS                                     |
| ULTIMA_ATUALIZACAO | TIMESTAMP        | 10              | Data e hora completa                                       |
| EMAIL              | VARCHAR(100)     | 102             | 2 bytes controle + 100 bytes                                |
| TELEFONE           | VARCHAR(20)      | 22              | 2 bytes controle + 20 bytes                                 |
| OBSERVACOES        | CLOB(1K)         | Não contabilizado | Dados externos, ponteiro na linha base                      |

### Total estimado da linha (sem CLOB):

`4 + 11 + 52 + 2 + 4 + 1 + 5 + 3 + 10 + 102 + 22 = 216 bytes`

> ⚠️ Os dados CLOB/BLOB são armazenados externamente e só ocupam ponteiro na linha base.

---

# 5. Simulação Avançada: Tabela com LOBs

| Coluna    | Tipo de Dado  | Tamanho Estimado (bytes) | Armazenamento                        | Observação                                |
|-----------|---------------|--------------------------|------------------------------------|------------------------------------------|
| ID_DOC    | INTEGER       | 4                        | Inline (linha base)                 | Chave primária                           |
| TITULO    | VARCHAR(255)  | 257                      | Inline                             | 255 bytes + 2 bytes controle comprimento |
| DESCRICAO | CLOB(10K)     | ~20 (ponteiro na linha)  | Fora da linha (LOBMAP + tablespace) | Dados armazenados fora da linha          |
| ANEXO_PDF | BLOB(5MB)     | ~20 (ponteiro na linha)  | Fora da linha (LOBMAP + tablespace) | Dados binários grandes fora da linha    |

> **Nota:** LOBs armazenam apenas ponteiro no registro; o conteúdo real fica em tablespace auxiliar, gerenciado pelo LOBMAP.

---

# 6. Explicações Técnicas Importantes

### Bitmap Nullability

- DB2 usa bitmap para indicar colunas NULL.
- Cada bit representa uma coluna: 1 = NULL, 0 = valor presente.
- Um byte cobre até 8 colunas.
- Espaço extra por linha, necessário para operações rápidas.

### Overhead da Linha (Row Overhead)

- Cada linha tem um overhead fixo de 10 a 12 bytes.
- Contém dados de controle como tamanho da linha, status de lock, bitmap nullability.

### Directory Pages e Tablespaces

- Dados armazenados em pages (normalmente 4KB) dentro de tablespaces.
- Directory pages guardam metadados para localizar rapidamente essas pages.
- Essencial para performance e escalabilidade do DB2.

### Compressão de Dados

- Compressão de linhas ou páginas reduz o tamanho físico no disco.
- Pode diminuir espaço em 30-70%.
- Altamente recomendada para tabelas maiores que 4GB.
- Melhora performance de I/O e uso de buffer pool.

---

# 7. Boas Práticas de Dimensionamento

- Sempre utilize estimativas realistas para campos VARCHAR.
- Inclua overheads de linha e bitmap na conta.
- Avalie compressão para tabelas grandes ou com dados repetitivos.
- Considere particionamento para tabelas muito volumosas ou com acesso segmentado.
- Monitore crescimento anual da tabela para prever necessidade futura de espaço.

---

# 8. Referências Oficiais IBM

- [DB2 12 for z/OS Performance Topics](https://www.ibm.com/docs/en/db2-for-zos/12?topic=performance)
- [DB2 for z/OS SQL Reference](https://www.ibm.com/docs/en/db2-for-zos/12?topic=sql-reference)
- [IBM Knowledge Center - Data Compression](https://www.ibm.com/docs/en/db2-for-zos/12?topic=performance-data-compression)
- [IBM Redbooks - DB2 for z/OS Database Administration](https://www.redbooks.ibm.com/abstracts/sg247821.html)

---

*Este manual foi elaborado com base em informações oficiais da IBM, práticas de mercado e experiência em ambientes produtivos do DB2 for z/OS.*

---

**Próximos passos sugeridos:**  
Após dominar o dimensionamento, é possível construir uma planilha dinâmica que automatize esses cálculos, integrando tabelas com todos os tipos de dados e simulações de crescimento, compressão e particionamento.

---


