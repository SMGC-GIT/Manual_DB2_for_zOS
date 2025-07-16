# 📊 STROBE para z/OS – Guia Técnico e Didático

## 📚 Índice

- [1. O que é STROBE?](#1-o-que-é-strobe)
- [2. Para que serve o STROBE?](#2-para-que-serve-o-strobe)
- [3. Como o STROBE funciona](#3-como-o-strobe-funciona)
- [4. Principais componentes](#4-principais-componentes)
- [5. Como interpretar os relatórios](#5-como-interpretar-os-relatórios)
- [6. Uso do STROBE com DB2 for z/OS](#6-uso-do-strobe-com-db2-for-zos)
- [7. Melhores práticas](#7-melhores-práticas)
- [8. Comandos úteis e exemplos](#8-comandos-úteis-e-exemplos)
- [9. Links úteis e documentação oficial](#9-links-úteis-e-documentação-oficial)

---

## 1. O que é STROBE?

**STROBE** é uma ferramenta de análise de desempenho em ambiente z/OS desenvolvida pela **Compuware** (atualmente parte da **BMC Software**). Ela permite **identificar gargalos de performance** em programas que rodam em ambientes batch, CICS, DB2, IMS, entre outros.

> 📌 Também chamado de "STROBE for z/OS", é amplamente utilizado por **DBAs**, **desenvolvedores**, **analistas de sistemas** e **especialistas em performance** para identificar código ineficiente ou recursos excessivamente consumidos.

---

## 2. Para que serve o STROBE?

STROBE é utilizado para:

- Identificar **rotinas ineficientes**
- Determinar **tempo de CPU** consumido por módulos
- Analisar **esperas de I/O**
- Medir **esperas por locks**, recursos, serviços
- Otimizar **SQLs pesadas no DB2**
- Localizar **acessos excessivos a VSAM ou DB2**
- Determinar **hotspots** em código Assembler, COBOL, PL/I, etc.

---

## 3. Como o STROBE funciona

STROBE funciona por **amostragem estatística**, capturando o estado da aplicação em execução em **intervalos regulares** (snapshots). Ao final da coleta, o STROBE monta um relatório com:

- Percentual de tempo em cada módulo/linha
- Chamadas a serviços do sistema
- Eventos de espera (I/O, lock, etc.)
- Utilização de CPU por módulo

> 🧠 Como é baseado em estatística, não afeta significativamente o desempenho do sistema.

---

## 4. Principais componentes

### ✔️ Measurement Request
- Requisição de medição, definindo qual job/task será analisado.

### ✔️ Measurement Session
- Sessão de coleta dos dados, realizada no tempo real ou por job batch.

### ✔️ STROBE Interface
- Interfaces ISPF e batch (JCL).

### ✔️ Performance Profile (Relatório)
- Saída gerada após a medição com todas as estatísticas e diagnósticos.

---

## 5. Como interpretar os relatórios

Os **Performance Profiles** do STROBE são detalhados e podem conter:

| Seção                | Conteúdo                                                                 |
|----------------------|--------------------------------------------------------------------------|
| **Module Usage**     | Quais módulos consomem mais CPU                                          |
| **Procedure Usage**  | Procedimentos mais utilizados                                             |
| **Service Time**     | Tempo de serviço por chamada (ex: Getmain, VSAM, SQL CALL, etc.)         |
| **Wait States**      | Onde há esperas (I/O, locks, recursos)                                   |
| **DB2 Statements**   | SQLs que causaram maiores esperas ou uso de CPU                          |
| **Line Attribution** | Linhas específicas do código mais executadas ou com maior uso de CPU     |

---

## 6. Uso do STROBE com DB2 for z/OS

O STROBE é extremamente eficaz na análise de **programas COBOL/DB2**, auxiliando na identificação de:

- **SQLs com alto custo**
- **Índices mal utilizados**
- **Excessos de fetch ou calls**
- **Acessos repetitivos à mesma tabela**
- **Lock waits** e **deadlocks**

### 🔍 Exemplo de indicadores no relatório:
- High CPU time on SQL
- Consecutive fetches
- Locking time > 10%
- Wait time on RID Pool

> ✅ STROBE também permite correlacionar chamadas SQL com o plano de execução (EXPLAIN) do DB2.

---

## 7. Melhores práticas

| Prática                                      | Descrição                                                                 |
|----------------------------------------------|--------------------------------------------------------------------------|
| 🕐 Executar STROBE em ambiente controlado     | Use ambientes de homologação ou produções controladas.                   |
| 📌 Capturar em horário de pico                | Mede com mais precisão os gargalos reais.                               |
| 🧩 Fazer o mapping correto do load module     | Necessário para identificar linha por linha no relatório.               |
| 🔎 Comparar antes e depois de otimizações     | Ajuda a validar ganhos de performance.                                  |
| 📄 Usar junto com o EXPLAIN do DB2            | Ajuda a correlacionar SQLs ineficientes com planos mal otimizados.      |

---

## 8. Comandos úteis e exemplos

### ✅ Exemplo de JCL para iniciar STROBE:

```jcl
//STROBJOB JOB (ACCT),'STROBE TEST',CLASS=A,MSGCLASS=X
//STEP01   EXEC PGM=STRB,PARM='JOBNAME=JOBXYZ,TIME=10'
//STEPLIB  DD DSN=STROBE.LOADLIB,DISP=SHR
//PROFILE  DD SYSOUT=*
//SYSOUT   DD SYSOUT=*
