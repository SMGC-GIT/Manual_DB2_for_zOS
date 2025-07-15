# 📚 Parametrização da Ordenação de Famílias Habilitadas ao PBF

## 🎯 Objetivo

Permitir que a ordenação das famílias habilitadas ao Programa Bolsa Família (PBF) seja definida por parâmetros dinâmicos em tabela, conforme diretrizes do MDS, sem necessidade de alterações no código ou JCL.

---

## 🧠 Justificativa

- O gestor poderá mudar os critérios de ordenação sempre que necessário;
- Cada família pode atender a múltiplos critérios;
- A solução deve aplicar a ordenação vigente **no momento da habilitação**;
- Toda a ordenação deve ser **rastreável** e auditável;
- Critérios devem poder ser **priorizados e ativados/desativados**;
- A aplicação da ordenação será feita com **DFSORT**.

---

## 🧾 Estrutura da Tabela: `SCHEDULE_ORDENACAO`

```sql
CREATE TABLE SCHEDULE_ORDENACAO (
    NOME_EXECUCAO      VARCHAR(100),  -- Nome identificador do conjunto de regras
    PRIORIDADE         SMALLINT,      -- Ordem de aplicação dos critérios (1 = mais prioritário)
    CODIGO_REGRA       CHAR(1),       -- Identificação do critério (A a L)
    DESCRICAO_REGRA    VARCHAR(255),  -- Explicação funcional do critério
    POSICAO_INICIAL    SMALLINT,      -- Byte inicial do campo no arquivo de dados
    TAMANHO_CAMPO      SMALLINT,      -- Tamanho em bytes do campo
    TIPO               CHAR(2),       -- CH = Alfanumérico, ZD = Numérico zonado, PD = Packed decimal
    ORDEM              CHAR(1),       -- A = Ascendente, D = Descendente
    ATIVO              CHAR(1),       -- S = Ativo, N = Inativo
    DATA_ATIVACAO      TIMESTAMP      -- Data/hora em que a regra foi ativada
);
```

---

## 📋 Explicação de cada campo

| Campo             | Para que serve                                                                                   |
|------------------|--------------------------------------------------------------------------------------------------|
| `NOME_EXECUCAO`   | Nome da regra vigente (ex: ORDENA_PBF_VIGENTE)                                                  |
| `PRIORIDADE`      | Define a ordem de aplicação entre os critérios                                                  |
| `CODIGO_REGRA`    | Identificador único do critério (letra A-L)                                                     |
| `DESCRICAO_REGRA` | Descrição funcional do critério                                                                 |
| `POSICAO_INICIAL` | Em qual byte do arquivo fixo começa esse campo                                                  |
| `TAMANHO_CAMPO`   | Quantos bytes o campo ocupa                                                                     |
| `TIPO`            | Tipo de dado esperado: texto (CH), número zonado (ZD), packed (PD)                             |
| `ORDEM`           | Direção da ordenação: A = crescente, D = decrescente                                            |
| `ATIVO`           | Se o critério está ativo (S) ou não (N)                                                         |
| `DATA_ATIVACAO`   | Quando essa regra passou a valer                                                                |

---

## 📥 Inserts dos 12 critérios definidos

```sql
INSERT INTO SCHEDULE_ORDENACAO VALUES
('ORDENA_PBF_VIGENTE',  1, 'A', 'Famílias indicadas por decisão judicial',                  1,  1, 'CH', 'D', 'S', CURRENT_TIMESTAMP),
('ORDENA_PBF_VIGENTE',  2, 'B', 'Famílias indicadas por erro operacional',                  2,  1, 'CH', 'D', 'S', CURRENT_TIMESTAMP),
('ORDENA_PBF_VIGENTE',  3, 'C', 'Grupos prioritários',                                      3,  1, 'CH', 'D', 'S', CURRENT_TIMESTAMP),
('ORDENA_PBF_VIGENTE',  4, 'D', 'Menor renda',                                              4,  5, 'ZD', 'A', 'S', CURRENT_TIMESTAMP),
('ORDENA_PBF_VIGENTE',  5, 'E', 'Data de atualização cadastral mais recente',              9,  8, 'CH', 'D', 'S', CURRENT_TIMESTAMP),
('ORDENA_PBF_VIGENTE',  6, 'F', 'Família habilitada há mais tempo',                       17,  8, 'CH', 'A', 'S', CURRENT_TIMESTAMP),
('ORDENA_PBF_VIGENTE',  7, 'G', 'Qtd. de pessoas com menos de 7 anos',                    25,  2, 'ZD', 'D', 'S', CURRENT_TIMESTAMP),
('ORDENA_PBF_VIGENTE',  8, 'H', 'Qtd. de pessoas com menos de 18 anos',                   27,  2, 'ZD', 'D', 'S', CURRENT_TIMESTAMP),
('ORDENA_PBF_VIGENTE',  9, 'I', 'Qtd. de mulheres gestantes vigentes',                    29,  2, 'ZD', 'D', 'S', CURRENT_TIMESTAMP),
('ORDENA_PBF_VIGENTE', 10, 'J', 'Responsável familiar do sexo feminino',                  31,  1, 'CH', 'D', 'S', CURRENT_TIMESTAMP),
('ORDENA_PBF_VIGENTE', 11, 'K', 'Responsável familiar com mais de 60 anos',               32,  1, 'CH', 'D', 'S', CURRENT_TIMESTAMP),
('ORDENA_PBF_VIGENTE', 12, 'L', 'Código familiar crescente',                              33, 10, 'CH', 'A', 'S', CURRENT_TIMESTAMP);
```

---

## 📄 Exemplo de Arquivo de Entrada (layout fixo de 43 bytes)

```
Linha 1: S N 3 00500 20240701 20210101 02 05 01 F S 1234567890
Linha 2: N N 1 00200 20240615 20210312 00 03 00 M N 2234567890
```

---

## 📌 Interpretação da linha

| Campo                             | Valor Linha 1 | Byte inicial | Tamanho |
|----------------------------------|---------------|---------------|----------|
| Decisão judicial                 | S             | 1             | 1        |
| Erro operacional                 | N             | 2             | 1        |
| Grupo prioritário               | 3             | 3             | 1        |
| Renda per capita                 | 00500         | 4             | 5        |
| Data de atualização cadastral   | 20240701      | 9             | 8        |
| Data habilitação                | 20210101      | 17            | 8        |
| Crianças < 7 anos               | 02            | 25            | 2        |
| Pessoas < 18 anos               | 05            | 27            | 2        |
| Gestantes                       | 01            | 29            | 2        |
| Sexo responsável                | F             | 31            | 1        |
| Responsável > 60 anos           | S             | 32            | 1        |
| Código da família               | 1234567890    | 33            | 10       |

---

## 🧠 Como o DFSORT usa esses dados

Consulta que gera o comando `SORT FIELDS` dinamicamente:

```sql
SELECT
  'SORT FIELDS=(' ||
  LISTAGG(
    POSICAO_INICIAL || ',' || TAMANHO_CAMPO || ',' || TIPO || ',' || ORDEM,
    ','
  ) WITHIN GROUP (ORDER BY PRIORIDADE) || ')'
AS COMANDO_SORT
FROM SCHEDULE_ORDENACAO
WHERE NOME_EXECUCAO = 'ORDENA_PBF_VIGENTE'
  AND ATIVO = 'S';
```

### Exemplo de saída:

```
SORT FIELDS=(1,1,CH,D,2,1,CH,D,3,1,CH,D,4,5,ZD,A,9,8,CH,D,...,33,10,CH,A)
```

---

## ✅ Conclusão

A tabela `SCHEDULE_ORDENACAO` é a base para:

- Definir a ordenação das famílias
- Controlar quais critérios são usados e em que ordem
- Permitir alterações sem impacto técnico
- Rastrear qual regra foi aplicada em cada habilitação

---
