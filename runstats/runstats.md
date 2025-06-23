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

Esta seção visa orientar a análise técnica de performance focando no uso correto do comando `RUNSTATS`. Muitas vezes, a lentidão de uma query está associada a estatísticas incompletas ou imprecisas, que levam o otimizador do DB2 a escolher planos de acesso inadequados.

---

## ✅ Perguntas Relevantes sobre o RUNSTATS

### 1. Qual foi o comando exato utilizado no RUNSTATS?

```sql
RUNSTATS TABLESPACE DBX.TSX 
  TABLE(ALL) 
  INDEX(ALL) 
  KEYCARD 
  FREQVAL NUMCOLS 1 
  HISTOGRAM 
  REPORT YES
```

🔎 Importância: muitos problemas ocorrem por estatísticas coletadas com parâmetros padrões, sem detalhamento suficiente para o otimizador.

---

### 2. Foi utilizado FREQVAL, HISTOGRAM ou KEYCARD?

- `FREQVAL`: identifica valores muito comuns ou raros (ex: 'ATIVO', 'CANCELADO').
- `HISTOGRAM`: distribuições não uniformes.
- `KEYCARD`: cardinalidade real das chaves do índice.

🔎 Sem essas opções, o otimizador assume distribuição uniforme e pode tomar decisões ruins.

---

### 3. As estatísticas foram coletadas para todas as colunas de filtro?

Se a query usa colunas **fora dos índices** nos filtros, é essencial declarar explicitamente:

```sql
FREQVAL NUMCOLS 1 ON COLUMNS(COL1, COL2, COL3)
```

🔎 Sem isso, a seletividade será baseada em suposições.

---

### 4. Foi feita coleta para COLGROUPs (colunas combinadas)?

Quando a query filtra por mais de uma coluna, o otimizador assume independência estatística, o que pode gerar erro de estimativa. Exemplo:

```sql
COLGROUP(COL1, COL2) FREQVAL NUMCOLS 2
```

🔎 Essencial para casos com filtros compostos ou joins com múltiplas colunas.

---

### 5. Foi feito REBIND com EXPLAIN após o RUNSTATS?

```sql
REBIND PACKAGE(...) EXPLAIN(YES) APREUSE(WARN)
```

🔎 O plano só será recalculado após o REBIND. Sem isso, mesmo boas estatísticas não são aproveitadas.

---

### 6. O plano de acesso mudou após RUNSTATS + REBIND?

Pergunte se:
- Deixou de usar índice?
- Passou a fazer Table Space Scan?
- Houve regressão?

🔎 Importante para avaliar se o `RUNSTATS` teve **efeito benéfico ou negativo**.

---

## ❗ Se o RUNSTATS Não Resolver

Se o problema persistir mesmo após um `RUNSTATS` completo, os próximos passos são:

1. Verificar o **novo plano de acesso** (EXPLAIN).
2. Avaliar a **reescrita da query** para evitar funções e filtros em Stage 2.
3. Criar/ajustar **índices** baseados no padrão de uso da query.
4. Coletar **COLGROUPs relevantes** com novo RUNSTATS.
5. Considerar o uso de `REOPT(ALWAYS)` ou `OPTHINT`.

---

## 📌 Exemplo de RUNSTATS Ideal

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

🔎 Esse comando fornece ao otimizador todas as estatísticas essenciais para montar um plano eficiente, com conhecimento real sobre os dados.

---
