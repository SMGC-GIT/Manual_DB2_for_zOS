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
---

# 💼 Solução Técnica: Parametrização de Ordenação de Famílias PBF (MDS)

## 🧠 Contexto

A ordenação das famílias habilitadas ao PBF deve ser dinâmica, conforme regras do MDS. Isso será feito através de uma tabela (`SCHEDULE_ORDENACAO`) que define os critérios ativos, sua ordem de aplicação e configuração técnica.

A ordenação deve refletir **imediatamente** após qualquer alteração.  
**Apenas o perfil “Caixa Master” poderá fazer alterações**.

---

## 🧩 Etapas e Componentes da Solução

### 1. 🔧 População da Tabela `SCHEDULE_ORDENACAO`

#### 🟦 A) Via tela online (recomendado)

- **Tela de manutenção** será desenvolvida com segurança baseada em perfil.
- Acesso restrito ao **perfil Caixa Master**.
- Tela terá os campos:
  - Código da regra (A a L)
  - Descrição (pré-preenchida)
  - Ordem de prioridade (1 a 12)
  - Ativo (Sim/Não)
  - Tipo, tamanho e posição já fixos (apenas visualização)
- Alterações gravam diretamente na tabela via **programa CICS**.
- O sistema atualiza a `DATA_ATIVACAO` automaticamente ao salvar.

> ✅ Alterações feitas na tela refletirão **imediatamente** na próxima execução do processo de ordenação.

#### 🟨 B) Via job batch (opcional)

- Em casos emergenciais ou técnicos, será possível alimentar a tabela via **programa COBOL** ou **REXX** batch.
- Pode usar **SYSIN** com estrutura CSV ou linha por linha.
- Exemplo de entrada em SYSIN:

```
ORDENA_PBF_VIGENTE,1,A,Famílias indicadas por decisão judicial,1,1,CH,D,S
```

- Ou LOAD DB2 se houver grande volume.

---

### 2. 🧑‍💼 Como o Gestor altera os dados?

| Forma         | Tecnologias        | Permissão           | Reflexo |
|---------------|--------------------|----------------------|---------|
| Tela online   | CICS + DB2         | Caixa Master         | Imediato |
| Batch/manual  | JCL + COBOL/REXX   | Suporte técnico      | Imediato |

- A alteração de prioridade (campo `PRIORIDADE`) muda a **ordem de aplicação dos critérios**.
- O campo `ATIVO = 'S'` ou `'N'` determina se o critério está ativo ou não.
- As alterações são lidas diretamente na próxima execução.

---

### 3. 📦 É necessário LOAD/UNLOAD?

| Situação                                         | Ação sugerida            |
|--------------------------------------------------|--------------------------|
| Inicial (popular as 12 regras pela 1ª vez)       | LOAD ou INSERT manual    |
| Alterações pontuais (mudar ordem, ativar)        | Via tela ou UPDATE       |
| Reconfiguração completa (trocar regras)          | UNLOAD → alteração → LOAD|

> ✅ Em produção, **preferimos usar UPDATE/INSERT via tela**. LOAD só em casos controlados.

---

### 4. 🧬 Ordem dos eventos no processo

#### Fluxo padrão da ordenação das famílias:

1. 🔁 O gestor altera a prioridade ou ativa/desativa um critério via tela
2. ✅ A tabela `SCHEDULE_ORDENACAO` é atualizada com nova `DATA_ATIVACAO`
3. 🛠 Um programa (REXX ou COBOL) consulta a tabela e **gera dinamicamente** a instrução `SORT FIELDS=(...)`
4. 📄 A instrução é gravada num dataset `&&SORTSYSIN` (em disco)
5. 📑 Um **job JCL com DFSORT** usa esse dataset como `SYSIN` e aplica a ordenação ao arquivo de entrada de famílias
6. ✅ O arquivo de saída já estará ordenado conforme os critérios definidos
7. 📝 Opcional: gravar na `HISTORICO_ORDENACAO` qual conjunto de regras foi usado para aquela execução

---

### 5. 🛠 Tecnologias envolvidas

| Etapa                     | Tecnologia recomendada |
|---------------------------|------------------------|
| Alteração de critérios    | Tela CICS + DB2        |
| Consulta à tabela         | SQL com ORDER BY       |
| Geração do SYSIN DFSORT   | COBOL ou REXX          |
| Execução da ordenação     | DFSORT via JCL         |
| Controle de histórico     | Tabela `HIST_ORDENACAO` (opcional) |

---

## 🧾 Exemplo técnico da alteração

> Exemplo: o gestor decide que “menor renda” agora será prioridade 1.

1. Ele acessa a tela, localiza o critério “D” e muda a prioridade para 1.
2. O sistema atualiza o campo `PRIORIDADE = 1`, e `DATA_ATIVACAO = current timestamp`.
3. O processo batch noturno executa:
   - Gera o novo `SORT FIELDS`
   - Ordena o arquivo com DFSORT
   - Habilita famílias na nova ordem
4. A família com menor renda aparece antes de outra que era prioritária anteriormente.

---

## 🧪 Exemplo prático com dados reais

### 📄 Arquivo de entrada simulado (layout fixo com 43 posições)

```
S N 3 00500 20240701 20210101 02 05 01 F S 1234567890
N N 1 00200 20240615 20210312 00 03 00 M N 2234567890
```

### 🔍 Interpretação das colunas

| Campo                             | Posição | Tamanho | Tipo | Exemplo     |
|----------------------------------|---------|---------|------|-------------|
| Indic. Decisão Judicial          | 1       | 1       | CH   | 'S' ou 'N'  |
| Indic. Erro Operacional          | 3       | 1       | CH   | 'N'         |
| Grupo Prioritário                | 5       | 1       | CH   | '3'         |
| Renda Per Capita                 | 7       | 5       | ZD   | '00500'     |
| Data Atualização Cadastral       | 13      | 8       | CH   | '20240701'  |
| Data Habilitação Família         | 22      | 8       | CH   | '20210101'  |
| Qtde Menores de 7 Anos           | 31      | 2       | ZD   | '02'        |
| Qtde Menores de 18 Anos          | 34      | 2       | ZD   | '05'        |
| Qtde Gestantes (sexo fem)        | 37      | 2       | ZD   | '01'        |
| Sexo do Responsável Familiar     | 40      | 1       | CH   | 'F'         |
| Indicador de RF > 60 anos        | 42      | 1       | CH   | 'S'         |
| Código Familiar                  | 43      | 10      | CH   | '1234567890'|

---

### ✅ Tabela `SCHEDULE_ORDENACAO` com critérios ativos

```sql
INSERT INTO SCHEDULE_ORDENACAO VALUES ('ORDENA_PBF_VIGENTE', 1, 'RENDA_PERCAPITA', 7, 5, 'ZD', 'A', 'S');
INSERT INTO SCHEDULE_ORDENACAO VALUES ('ORDENA_PBF_VIGENTE', 2, 'DATA_ATUALIZACAO', 13, 8, 'CH', 'D', 'S');
INSERT INTO SCHEDULE_ORDENACAO VALUES ('ORDENA_PBF_VIGENTE', 3, 'CODIGO_FAMILIAR', 43, 10, 'CH', 'A', 'S');
```

---

### 🧾 Comando DFSORT gerado automaticamente

```jcl
SORT FIELDS=(7,5,ZD,A,13,8,CH,D,43,10,CH,A)
```

---

### 📊 Resultado ordenado do arquivo de entrada

| Linha | Renda  | Data Atualização | Código Familiar |
|-------|--------|------------------|-----------------|
| 2     | 00200  | 20240615         | 2234567890      |
| 1     | 00500  | 20240701         | 1234567890      |

---

## 📘 Histórico da Ordenação Aplicada

Tabela opcional para auditoria e rastreabilidade:

```sql
CREATE TABLE HIST_ORDENACAO_APLICADA (
    ID_EXECUCAO      INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    NOME_EXECUCAO    VARCHAR(100),
    DATA_APLICACAO   TIMESTAMP,
    USUARIO_APLICOU  VARCHAR(50),
    COMANDO_SORT     VARCHAR(1000),
    QUANTIDADE_REGS  INTEGER
);
```

> Essa tabela será atualizada pelo programa REXX ou COBOL logo após a execução do SORT, registrando os critérios usados e a quantidade de registros processados.

---

## 🛡️ Segurança e Perfis

| Ação                             | Perfil necessário |
|----------------------------------|-------------------|
| Alterar critério na tela online  | Caixa Master      |
| Executar ordenação               | Técnico (Batch)   |
| Consultar histórico              | Auditoria / Master|

---

## ✅ Conclusão

Esta solução garante:

- Flexibilidade total para o gestor decidir a ordenação;
- Controle e auditoria por meio de tabelas e histórico;
- Integração transparente com o batch existente via DFSORT;
- Facilidade de manutenção futura com novos critérios.

---
---

# 💼 Solução Técnica — Parametrização da Ordenação de Famílias Habilitadas ao PBF (MDS)

## 🧠 Objetivo

Atender à necessidade do MDS de ajustar, a qualquer momento, os critérios de ordenação utilizados para habilitação de famílias ao Programa Bolsa Família, com base em parâmetros controlados por gestor autorizado.

A ordenação será:

- Dinâmica e modificável;
- Executada por **DFSORT** com base nos critérios definidos;
- Controlada por tabela de configuração em **DB2**;
- Aplicada automaticamente no momento da habilitação.

---

## 🎯 Requisitos Funcionais

- O gestor poderá **alterar os critérios de ordenação** sempre que necessário;
- Cada família pode **atender a múltiplos critérios simultaneamente**;
- A ordenação vigente no **momento da habilitação** deve ser aplicada;
- Toda ordenação deve ser **rastreável e auditável**;
- Critérios devem ter campos de **prioridade, tipo, direção e status ativo**;
- O processo de ordenação será feito com **DFSORT**;
- Apenas perfis com permissão "Caixa Master" poderão alterar a ordenação.

---

## 🗄️ Tabela de Parâmetros: `SCHEDULE_ORDENACAO`

```sql
CREATE TABLE SCHEDULE_ORDENACAO (
    NOME_EXECUCAO      VARCHAR(100),  -- Identificador da configuração
    PRIORIDADE         SMALLINT,      -- Ordem de aplicação (1 = mais prioritário)
    CODIGO_REGRA       CHAR(1),       -- Código do critério (A a L)
    DESCRICAO_REGRA    VARCHAR(255),  -- Descrição funcional
    POSICAO_INICIAL    SMALLINT,      -- Byte inicial do campo no arquivo fixo
    TAMANHO_CAMPO      SMALLINT,      -- Quantidade de bytes
    TIPO               CHAR(2),       -- Tipo DFSORT: CH, ZD, PD
    ORDEM              CHAR(1),       -- Direção: A (asc), D (desc)
    ATIVO              CHAR(1),       -- S = Ativo | N = Inativo
    DATA_ATIVACAO      TIMESTAMP      -- Momento da ativação
);
```

---

## 📝 Explicação dos Campos

| Campo             | Função                                                                 |
|------------------|------------------------------------------------------------------------|
| NOME_EXECUCAO     | Nome da configuração vigente (ex: `ORDENA_PBF_VIGENTE`)               |
| PRIORIDADE        | Define a ordem entre os critérios (menor número = maior prioridade)   |
| CODIGO_REGRA      | Código da regra (A a L)                                                |
| DESCRICAO_REGRA   | Descrição funcional do critério                                        |
| POSICAO_INICIAL   | Byte inicial no arquivo fixo                                           |
| TAMANHO_CAMPO     | Quantidade de bytes lidos a partir da posição inicial                 |
| TIPO              | Tipo do campo no SORT: CH (alfanumérico), ZD (zonado), PD (packed)    |
| ORDEM             | A = ascendente, D = descendente                                       |
| ATIVO             | Se a regra está ativa (S) ou inativa (N)                              |
| DATA_ATIVACAO     | Quando a regra passou a valer                                          |

---

## 🔁 Inserção dos Critérios Iniciais

```sql
INSERT INTO SCHEDULE_ORDENACAO VALUES
('ORDENA_PBF_VIGENTE',  1, 'A', 'Famílias indicadas por decisão judicial',   1,  1, 'CH', 'D', 'S', CURRENT_TIMESTAMP),
('ORDENA_PBF_VIGENTE',  2, 'B', 'Famílias indicadas por erro operacional',   2,  1, 'CH', 'D', 'S', CURRENT_TIMESTAMP),
('ORDENA_PBF_VIGENTE',  3, 'C', 'Grupos prioritários',                       3,  1, 'CH', 'D', 'S', CURRENT_TIMESTAMP),
('ORDENA_PBF_VIGENTE',  4, 'D', 'Menor renda',                               4,  5, 'ZD', 'A', 'S', CURRENT_TIMESTAMP),
('ORDENA_PBF_VIGENTE',  5, 'E', 'Data de atualização cadastral mais recente',9,  8, 'CH', 'D', 'S', CURRENT_TIMESTAMP),
('ORDENA_PBF_VIGENTE',  6, 'F', 'Família habilitada há mais tempo',         17,  8, 'CH', 'A', 'S', CURRENT_TIMESTAMP),
('ORDENA_PBF_VIGENTE',  7, 'G', 'Qtd. de pessoas com menos de 7 anos',      25,  2, 'ZD', 'D', 'S', CURRENT_TIMESTAMP),
('ORDENA_PBF_VIGENTE',  8, 'H', 'Qtd. de pessoas com menos de 18 anos',     27,  2, 'ZD', 'D', 'S', CURRENT_TIMESTAMP),
('ORDENA_PBF_VIGENTE',  9, 'I', 'Qtd. de mulheres gestantes vigentes',      29,  2, 'ZD', 'D', 'S', CURRENT_TIMESTAMP),
('ORDENA_PBF_VIGENTE', 10, 'J', 'Responsável familiar do sexo feminino',    31,  1, 'CH', 'D', 'S', CURRENT_TIMESTAMP),
('ORDENA_PBF_VIGENTE', 11, 'K', 'Responsável familiar com mais de 60 anos', 32,  1, 'CH', 'D', 'S', CURRENT_TIMESTAMP),
('ORDENA_PBF_VIGENTE', 12, 'L', 'Código familiar crescente',                33, 10, 'CH', 'A', 'S', CURRENT_TIMESTAMP);
```

---

## 📂 Exemplo de Arquivo de Entrada (LRECL = 43 bytes)

```
Linha 1: S N 3 00500 20240701 20210101 02 05 01 F S 1234567890
Linha 2: N N 1 00200 20240615 20210312 00 03 00 M N 2234567890
```

---

## 📊 Interpretação dos Campos (linha 1)

| Campo                                 | Valor | Byte inicial | Tamanho |
|--------------------------------------|--------|---------------|----------|
| Decisão judicial                     | S      | 1             | 1        |
| Erro operacional                     | N      | 2             | 1        |
| Grupo prioritário                    | 3      | 3             | 1        |
| Renda per capita                     | 00500  | 4             | 5        |
| Data atualização cadastral           | 20240701 | 9           | 8        |
| Data habilitação                     | 20210101 | 17          | 8        |
| Crianças < 7                         | 02     | 25            | 2        |
| Pessoas < 18                         | 05     | 27            | 2        |
| Gestantes                            | 01     | 29            | 2        |
| Sexo responsável                     | F      | 31            | 1        |
| Responsável > 60 anos                | S      | 32            | 1        |
| Código familiar                      | 1234567890 | 33        | 10       |

---

## 🧠 Geração dinâmica da cláusula `SORT FIELDS=`

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

---

## 🧾 Resultado esperado do SQL

```text
SORT FIELDS=(1,1,CH,D,2,1,CH,D,3,1,CH,D,4,5,ZD,A,9,8,CH,D,17,8,CH,A,25,2,ZD,D,27,2,ZD,D,29,2,ZD,D,31,1,CH,D,32,1,CH,D,33,10,CH,A)
```

Essa saída é salva em um dataset e utilizada como **SYSIN do DFSORT**.

---

## 📌 Esse SELECT é usado para:

- Gerar automaticamente os parâmetros `SORT FIELDS=(...)`;
- Controlar a lógica da ordenação sem alterar JCL;
- Atender dinamicamente o que foi definido na tabela pelo gestor.

---




