# 🧠 Manual Didático e Técnico: Dimensionamento de Tabelas no DB2 for z/OS

> 📌 Este guia serve como base para criação de uma planilha inteligente de dimensionamento de tabelas, utilizando apenas os atributos da tabela e volume de linhas como entrada.

---

## 📑 Índice

- [1. Objetivo do Cálculo](#1-objetivo-do-cálculo)
- [2. Entradas Necessárias](#2-entradas-necessárias)
- [3. Fórmulas de Dimensionamento](#3-fórmulas-de-dimensionamento)
  - [3.1. Cálculo do Tamanho da Linha](#31-cálculo-do-tamanho-da-linha)
  - [3.2. Cálculo do Tamanho da Página](#32-cálculo-do-tamanho-da-página)
  - [3.3. Cálculo do Número de Páginas](#33-cálculo-do-número-de-páginas)
  - [3.4. Cálculo do Espaço Total](#34-cálculo-do-espaço-total)
- [4. Ajustes com PCTFREE e FREEPAGE](#4-ajustes-com-pctfree-e-freepage)
- [5. Compressão de Dados](#5-compressão-de-dados)
- [6. Particionamento de Tabelas](#6-particionamento-de-tabelas)
- [7. Resultado Final e Conversão de Unidades](#7-resultado-final-e-conversão-de-unidades)
- [8. Aba auxiliar para detalhamento de campos](#8-aba-auxiliar-para-detalhamento-de-campos)
- [9. Recomendações Finais](#9-recomendações-finais)

---

## 1. Objetivo do Cálculo

Antecipar o espaço necessário para uma tabela ainda em fase de modelagem, com base apenas nos atributos (campos, tipos, tamanhos) e volume estimado de dados. Permite decidir previamente se:

- A tabela precisa de compressão
- É necessário particionar o tablespace
- Quantas páginas serão utilizadas
- Qual será o espaço em KB, MB, GB ou TB

---

## 2. Entradas Necessárias

| Campo                                | Tipo        | Descrição |
|-------------------------------------|-------------|-----------|
| Volume de Linhas Estimado           | Numérico    | Quantidade máxima de linhas que a tabela terá |
| Tamanho total da linha              | Numérico    | Soma dos tamanhos dos campos fixos e médios dos variáveis |
| Quantidade de colunas NULL          | Numérico    | Para cálculo do NULL Indicator |
| Quantidade de campos VARCHAR        | Numérico    | Estimativa de campos variáveis |
| FREEPAGE                            | Numérico    | Ex: 5 = 1 página vazia a cada 5 páginas |
| PCTFREE                             | Numérico    | Percentual da página reservada para futuras inserções |
| Tamanho da página (em bytes)        | Fixo ou editável | Normalmente: 4K, 8K, 16K ou 32K |
| Fator de Compressão (se aplicável)  | Decimal     | Exemplo: 0.7 significa compressão de 30% |

---

## 3. Fórmulas de Dimensionamento

### 3.1. Cálculo do Tamanho da Linha

```text
Tamanho da linha = 
  Soma dos tamanhos dos campos FIXOS 
+ (Número de campos VARCHAR × tamanho médio estimado)
+ NULL Indicator (INT((nº colunas + 7) / 8)) bytes
```

> 🎯 Dica: DB2 reserva 1 byte de NULL Indicator para cada 8 colunas.

---

### 3.2. Cálculo do Tamanho da Página

```text
Tamanho útil da página = 
  Tamanho da página física - overhead DB2 (cabeçalhos, etc.)

Tamanho disponível = 
  Tamanho útil da página × (1 - PCTFREE%)
```

> Exemplo com 8K e PCTFREE 10%:
> Tamanho disponível = 8192 × 0.9 = 7372 bytes

---

### 3.3. Cálculo do Número de Linhas por Página

```text
Linhas por página = 
  Floor(Tamanho disponível da página / Tamanho da linha)
```

---

### 3.4. Cálculo do Número de Páginas

```text
Páginas estimadas = 
  Volume de Linhas / Linhas por página

Aplicar FREEPAGE:
  Total com freepages = 
    Páginas estimadas × (1 + (1 / FREEPAGE))
```

---

## 4. Ajustes com PCTFREE e FREEPAGE

- **PCTFREE** reserva espaço nas páginas para futuras inserções (update, crescimento).
- **FREEPAGE** define o intervalo de páginas após o qual uma página vazia é deixada (ideal para insert em blocos).

> Exemplo: FREEPAGE = 5 → a cada 5 páginas, 1 vazia → aumenta em 20% o total de páginas.

---

## 5. Compressão de Dados

Compressão é aplicada **após o cálculo bruto**.

```text
Espaço final estimado = 
  Espaço total (em bytes) × Fator de Compressão
```

> Compressão média em instituições financeiras gira entre 30% e 50%, dependendo do tipo de dados.

---

## 6. Particionamento de Tabelas

Sugestão de **critérios simples**:

- Tabelas maiores que **2GB** → avaliar particionamento
- Acima de **1 milhão de páginas estimadas** → particionar
- Quando o crescimento for contínuo e previsível (ex: logs, transações, históricos)

---

## 7. Resultado Final e Conversão de Unidades

Conversão para facilitar leitura do resultado final:

```text
Bytes → KB = / 1024  
KB → MB = / 1024  
MB → GB = / 1024  
GB → TB = / 1024
```

### Exemplo Final:

| Métrica                     | Valor calculado |
|----------------------------|-----------------|
| Tamanho da linha           | 200 bytes       |
| Linhas por página (8K)     | 36              |
| Total de páginas           | 27.778          |
| Espaço estimado (sem comp.)| 222 MB          |
| Espaço após compressão     | 133 MB (compressão 40%) |

---

## 8. Aba auxiliar para detalhamento de campos

Para obter o **tamanho da linha**, é recomendável uma aba de apoio onde o modelador informe:

| Campo     | Tipo     | Tamanho | Nullable | Comentários              |
|-----------|----------|---------|----------|---------------------------|
| CPF       | CHAR     | 11      | NÃO      | Fixo                     |
| Nome      | VARCHAR  | 100     | SIM      | Variável                 |
| Endereço  | VARCHAR  | 200     | SIM      | Variável                 |
| DataNasc  | DATE     | 4       | NÃO      | Fixo                     |
| ...       | ...      | ...     | ...      | ...                      |

E a planilha já somaria os tamanhos automaticamente.

---

## 9. Recomendações Finais

- Adote valores default para PCTFREE (10%) e FREEPAGE (5) inicialmente, mas permita edição
- Calcule antes da criação física da tabela
- Evite surpresas em ambiente produtivo
- Gere cenários com compressão ON e OFF
- Documente os parâmetros utilizados

---

📌 **Nota**: Este modelo é baseado em boas práticas IBM + experiência de campo. O objetivo é ter um **modelo prático e auditável** para apoiar decisões técnicas de estruturação de banco.

