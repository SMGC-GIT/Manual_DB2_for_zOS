# 📘 Diagnóstico com Foco em RUNSTATS no DB2 for z/OS

> **Seção do Manual**: `04.03.1 – Diagnóstico com foco em RUNSTATS`

---

## 📑 Índice

- [🎯 Objetivo](#-objetivo)
- [✅ Perguntas Relevantes sobre o RUNSTATS](#-perguntas-relevantes-sobre-o-runstats)
  - [1. Qual foi o comando exato utilizado no RUNSTATS?](#1-qual-foi-o-comando-exato-utilizado-no-runstats)
  - [2. Foi utilizado FREQVAL, HISTOGRAM ou KEYCARD?](#2-foi-utilizado-freqval-histogram-ou-keycard)
  - [3. As estatísticas foram coletadas para todas as colunas de filtro?](#3-as-estatísticas-foram-coletadas-para-todas-as-colunas-de-filtro)
  - [4. Foi feita coleta para COLGROUPs (colunas combinadas)?](#4-foi-feita-coleta-para-colgroups-colunas-combinadas)
  - [5. Foi feito REBIND com EXPLAIN após o RUNSTATS?](#5-foi-feito-rebind-com-explain-após-o-runstats)
  - [6. O plano de acesso mudou após RUNSTATS + REBIND?](#6-o-plano-de-acesso-mudou-após-runstats--rebind)
- [❗ Se o RUNSTATS Não Resolver](#-se-o-runstats-não-resolver)
- [📌 Exemplo de RUNSTATS Ideal](#-exemplo-de-runstats-ideal)

---

## 🎯 Objetivo

O comando `RUNSTATS` é uma das ferramentas mais importantes para performance no DB2 for z/OS, pois alimenta o otimizador com informações reais sobre os dados (distribuição, cardinalidade, frequência etc.). Esta seção oferece um roteiro detalhado para diagnóstico de problemas de performance causados por estatísticas desatualizadas, mal coletadas ou ausentes — com foco em **eficiência de plano de acesso** e impacto direto em **sistemas críticos em produção**.

---

## ✅ Perguntas Relevantes sobre o RUNSTATS

---

### 1. Qual foi o comando exato utilizado no RUNSTATS?

**O que perguntar:**
> Qual foi o comando completo executado no RUNSTATS? Usaram `FREQVAL`, `KEYCARD`, `HISTOGRAM`?

**Por que isso importa:**
- Um comando simplificado como `RUNSTATS TABLE(ALL)` é insuficiente.
- Sem opções adicionais, o otimizador assume distribuição uniforme dos dados, o que raramente é verdade em produção.
- A falta de opções detalhadas **prejudica a seletividade calculada**, levando a planos ruins, como *table space scan* desnecessário.

**Consequências técnicas:**
- Filtros mal interpretados;
- Uso de índices inadequados;
- Substituição de *Nested Loop* por *Merge Join* ou *Hybrid Join* sem necessidade.

---

### 2. Foi utilizado `FREQVAL`, `HISTOGRAM` ou `KEYCARD`?

**Significado técnico:**
- `FREQVAL`: coleta os valores **mais frequentes** de colunas, útil quando há concentração em poucos valores (ex: status 'A').
- `HISTOGRAM`: divide os dados em **intervalos de frequência**, essencial quando há **distribuição desigual**.
- `KEYCARD`: atualiza a cardinalidade das chaves de índice, usada para avaliar seletividade e custo de index access.

**Impacto real:**
- Sem `FREQVAL` e `HISTOGRAM`, o otimizador **não reconhece valores dominantes ou raros**.
- Sem `KEYCARD`, ele pode subestimar ou superestimar o custo de usar o índice — levando ao **uso indevido de full scan**.

---

### 3. As estatísticas foram coletadas para todas as colunas de filtro?

**Exemplo de má prática comum:**
> A query filtra por `DATA_VENCIMENTO`, que **não está em nenhum índice** e não foi incluída no RUNSTATS.

**Resultado:**
- O otimizador assume uma distribuição uniforme;
- A estimativa de linhas retornadas fica incorreta;
- A performance se degrada.

**Como resolver:**
> Usar:
```sql
FREQVAL NUMCOLS 1 ON COLUMNS(DATA_VENCIMENTO)
```

**Observação:**  
Mesmo colunas fora dos índices **devem ter estatísticas**, se participarem de filtros, joins ou condições.

---

### 4. Foi feita coleta para COLGROUPs (colunas combinadas)?

**O que é:**
- `COLGROUP` permite que o DB2 colete estatísticas sobre **combinações de colunas**, essencial quando a query filtra por múltiplas colunas simultaneamente.

**Problema comum:**
- O otimizador assume que os filtros são **estatisticamente independentes**, o que gera erros grosseiros de estimativa.

**Exemplo prático:**
```sql
COLGROUP(COD_AGENCIA, COD_PRODUTO) FREQVAL NUMCOLS 2
```

**Sem isso, consequências possíveis:**
- Plano de acesso ineficiente;
- *Nested Loop* com cardinalidade errada;
- Joins com tabela errada como "outer".

---

### 5. Foi feito REBIND com EXPLAIN após o RUNSTATS?

**Por que isso é fundamental:**
- O otimizador **só reavalia o plano de acesso** depois de um REBIND.
- Mesmo com estatísticas atualizadas, se não houver REBIND, o plano antigo será mantido.

**O que é esperado:**
```sql
REBIND PACKAGE(CONTROLE.ROTINAXX) EXPLAIN(YES) APREUSE(WARN)
```

**Parâmetros importantes:**
- `EXPLAIN(YES)`: grava o novo plano na `DSN_STATEMNT_TABLE` ou `PLAN_TABLE`.
- `APREUSE(WARN)`: tenta reaproveitar o plano antigo, mas permite substituição caso não seja mais eficiente.

**Consequências da ausência de REBIND:**
- O sistema continua usando um **plano obsoleto**, mesmo com estatísticas novas;
- Diagnóstico pode parecer “ineficaz”, quando na verdade o REBIND foi omitido.

---

### 6. O plano de acesso mudou após RUNSTATS + REBIND?

**Por que perguntar:**
- Se o plano **não mudou**, as estatísticas podem ter sido inócuas (mal coletadas ou irrelevantes).
- Se o plano **mudou e piorou**, pode indicar:
  - Falta de `COLGROUP`;
  - Falta de valores frequentes;
  - Cardinalidade incorreta;
  - Preferência indevida por *table scan* ou *merge join*.

**Ação imediata:**
- Solicitar novo EXPLAIN da query após o REBIND;
- Comparar com plano anterior.

---

## ❗ Se o RUNSTATS Não Resolver

### Próximos passos recomendados:

1. **Analisar EXPLAIN com cautela:**
   - Matching Index Access?
   - Stage 1 ou Stage 2?
   - Tipo de join?

2. **Reescrita da query:**
   - Evitar funções e cálculos no WHERE;
   - Reduzir uso de `DISTINCT`, `ORDER BY` se não forem essenciais.

3. **Criação ou ajuste de índices:**
   - Adição de colunas com `INCLUDE`;
   - Novo índice composto.

4. **Novo RUNSTATS com COLGROUP e HISTOGRAM:**
   - Reforça o conhecimento do otimizador sobre dados combinados.

5. **Uso de REOPT(ALWAYS):**
   - Caso a query use parâmetros dinâmicos com variação alta.

---

## 📌 Exemplo de RUNSTATS Ideal

Este comando simula uma coleta de estatísticas adequada para casos em que há:
- Filtros com colunas isoladas e combinadas;
- Presença de colunas fora dos índices;
- Distribuição desigual dos dados;
- Necessidade de decisão baseada em valores mais frequentes.

```sql
RUNSTATS TABLESPACE DBX.TSX 
  TABLE(ALL) 
  INDEX(ALL) 
  KEYCARD 
  FREQVAL NUMCOLS 1 
  FREQVAL NUMCOLS 2 ON COLUMNS(COL1, COL2) 
  HISTOGRAM ON COLUMNS(COL1, COL2)
  REPORT YES
```

### Explicação de cada item:

- `TABLESPACE DBX.TSX`: define a tablespace de destino da coleta;
- `TABLE(ALL)`: coleta estatísticas para todas as tabelas do espaço;
- `INDEX(ALL)`: inclui todos os índices das tabelas;
- `KEYCARD`: atualiza a contagem de chaves distintas dos índices;
- `FREQVAL NUMCOLS 1`: coleta os valores mais frequentes de colunas individuais (útil para filtros simples);
- `FREQVAL NUMCOLS 2 ON COLUMNS(COL1, COL2)`: coleta frequência combinada de valores em pares de colunas (essencial para filtros múltiplos);
- `HISTOGRAM ON COLUMNS(COL1, COL2)`: coleta dados de distribuição para identificar valores dominantes e outliers;
- `REPORT YES`: gera um relatório detalhado para revisão das estatísticas geradas.

---

### ⚠️ Dicas importantes:

- Evite usar apenas `RUNSTATS TABLE(ALL) INDEX(ALL)` sem nenhuma personalização: isso não fornece informações suficientes para o otimizador tomar decisões precisas.
- Sempre **faça REBIND após o RUNSTATS**, com `EXPLAIN(YES)`, para garantir que o novo plano seja gerado com base nas estatísticas atualizadas.
- Analise o **novo plano de acesso** com base no `EXPLAIN`, antes de considerar reescrita da query ou criação de novos índices.
- Use `REPORT YES` para comparar a **cardinalidade estimada vs real** e verificar se a coleta foi suficiente.

---

📎 **Referência complementar**:  
- [IBM Docs – RUNSTATS Utility (Db2 13 for z/OS)](https://www.ibm.com/docs/en/db2-for-zos/13?topic=utilities-runstats-utility)
- [IBM Redbook – Db2 13 Performance Topics](https://www.redbooks.ibm.com/abstracts/sg248551.html)

---

🔚 **Conclusão da Seção**:  
Compreender e aplicar corretamente o `RUNSTATS` é essencial para garantir a eficiência das queries em ambientes críticos. Uma coleta mal feita pode comprometer todo o plano de acesso. Esta seção deve ser sempre revisitada antes de partir para alterações estruturais no banco ou reescrita de lógica SQL.

---

