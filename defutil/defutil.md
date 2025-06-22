# 🛠️ DEFUTIL - Utility Definition (DB2 for z/OS)

O `DEFUTIL` (DEFINE UTILITY) é um recurso poderoso do DB2 for z/OS utilizado para registrar manualmente entradas na tabela de catálogo `SYSIBM.SYSUTIL` ou para corrigir e controlar situações relacionadas a jobs de utilitários interrompidos ou inconsistentes.

Essa funcionalidade é essencial para gerenciar o estado e execução dos utilitários DB2, principalmente em cenários de falha ou quando é necessário reativar ou limpar uma entrada de utilitário que ficou pendente.

---

## 📚 Índice

- [🔹 1. O que é o DEFUTIL](#1-o-que-é-o-defutil)
- [🔹 2. Quando Utilizar o DEFUTIL](#2-quando-utilizar-o-defutil)
- [🔹 3. Sintaxe Geral do DEFUTIL](#3-sintaxe-geral-do-defutil)
- [🔹 4. Parâmetros do DEFUTIL](#4-parâmetros-do-defutil)
- [🔹 5. Exemplos Práticos](#5-exemplos-práticos)
- [🔹 6. Tratamento de Falhas de Utilitários com DEFUTIL](#6-tratamento-de-falhas-de-utilitários-com-defutil)
- [🔹 7. JCL de Execução do DEFUTIL](#7-jcl-de-execução-do-defutil)
- [🔹 8. Cuidados e Riscos na Utilização](#8-cuidados-e-riscos-na-utilização)
- [🔹 9. Referências Oficiais IBM](#9-referências-oficiais-ibm)

---

## 🔹 1. O que é o DEFUTIL

O `DEFUTIL` é um *service aid* do DB2 for z/OS que permite inserir ou modificar manualmente registros da tabela `SYSUTIL`, normalmente manipulada automaticamente pelo DB2 durante execução de utilitários como `LOAD`, `REORG`, `RUNSTATS`, etc.

---

## 🔹 2. Quando Utilizar o DEFUTIL

- Quando um utilitário DB2 falha e deixa registros “travando” sua reexecução.
- Para reativar ou limpar registros obsoletos da `SYSUTIL`.
- Para diagnosticar e simular execuções de utilitários em ambiente controlado.
- Em ambientes de teste ou homologação para fins de simulação ou investigação.

---

## 🔹 3. Sintaxe Geral do DEFUTIL

A execução do DEFUTIL se dá por meio de parâmetros fornecidos via SYSIN em um JCL utilizando o módulo DSNUTILB.

```plaintext
DEFUTIL  
  UTILID(utilitário_nome)
  DBID(database_id)
  PSID(tablespace_id)
  UTPROC(nome_da_rotina_utilitário)
  STATUS(status_do_utilitário)
