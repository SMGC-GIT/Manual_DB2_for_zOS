# Guia Completo de Análise e Monitoramento de Abend no DB2 for z/OS

---

## Índice

1. [Checklist para Coleta de Informações de Abend](#1-checklist-para-coleta-de-informações-de-abend)  
2. [Principais DDNAMEs para Análise de Abend no JOB](#2-principais-ddnames-para-análise-de-abend-no-job)  
3. [Ferramentas de Monitoramento Usadas em Ambiente](#3-ferramentas-de-monitoramento-usadas-em-ambiente)  
4. [Procedimentos Práticos para Verificação de Abend na](#4-procedimentos-práticos-para-verificação-de-abend-na)  
5. [Análise de Job Demorando Muito em Produção](#5-análise-de-job-demorando-muito-em-produção)  
6. [Explicações Detalhadas das Causas de Job Demorado](#6-explicações-detalhadas-das-causas-de-job-demorado)

---

## 1. Checklist para Coleta de Informações de Abend

| Nº  | Campo                                | Informação a Preencher/Coletar                |
|------|-------------------------------------|----------------------------------------------|
| 1    | Nome do Programa                    |                                              |
| 2    | Nome do JOB / Transação             |                                              |
| 3    | Data e Hora do Abend                |                                              |
| 4    | Código do Abend (ex: S0C7, U4038)  |                                              |
| 5    | Reason Code / RC                   |                                              |
| 6    | Nome do Plano DB2 (PLAN)            |                                              |
| 7    | Collection ID (se for PACKAGE)      |                                              |
| 8    | Logs JES (JESMSGLG, JESJCL, JESYSMSG) | Anexar print ou trecho                      |
| 9    | Último SQL Executado (se possível)  |                                              |
| 10   | Tabela(s) envolvida(s)              |                                              |
| 11   | Banco de Dados / Tablespace         |                                              |
| 12   | Tipo de Operação SQL (SELECT, INSERT, etc.) |                                      |
| 13   | Lock, Timeout ou Deadlock?          |                                              |
| 14   | Alguma alteração recente? (REORG, RUNSTATS, REBIND, etc.) |                             |
| 15   | Dump gerado? (SYSUDUMP, SYSMDUMP) – nome e local |                                     |
| 16   | Offset / Endereço do Abend          |                                              |
| 17   | Módulo/Programa onde ocorreu        |                                              |
| 18   | Ação Tomada até o momento           |                                              |
| 19   | Nome de quem coletou as informações |                                              |

> *Dica:* Use este checklist para coletar informações de forma estruturada e ágil, especialmente quando estiver remoto ou sem acesso direto.

---

## 2. Principais DDNAMEs para Análise de Abend no JOB

| DDNAME      | Descrição                                          | Importância na Análise          |
|-------------|----------------------------------------------------|--------------------------------|
| JESMSGLG    | Log do JES – início/fim do job, mensagens do z/OS | Muito importante               |
| JESYSMSG    | Mensagens de erro do sistema, dumps e info extra  | Muito importante               |
| JESJCL      | Exibe o JCL submetido e mensagens de erro         | Importante                    |
| SYSOUT      | Saída do programa – DISPLAYs, PRINTs, logs        | Essencial                     |
| SYSPRINT    | Mensagens de compiladores, DB2, utilitários       | Essencial                     |
| SYSUDUMP    | Dump gerado automaticamente em abends              | Muito útil                    |
| SYSMDUMP    | Dump binário mais completo para análise de abends | Muito útil                    |
| SYSABEND    | Dump padrão para abends (se configurado)           | Útil                         |
| SYSTSPRT    | Saída de compilação/utilitários/precompiladores    | Depende do contexto           |
| SYSTSIN     | Entrada de comandos TSO/SPUFI                      | Para reexecuções             |
| SYSTSOUT    | Saída de utilitários DB2 como DSNUTILB             | Útil para jobs DB2           |
| SYSIN       | Entrada de controle de utilitários                 | Análise de comandos          |
| SYSERR      | Mensagens de erro de compiladores/precompiladores | Relevante em falhas          |
| DBRMLIB     | Contém os DBRM usados no bind (DB2)                | Verificação de bind          |
| SYSDUMP     | Dump do programa (se configurado)                   | Se presente                  |

---

## 3. Ferramentas de Monitoramento Usadas em Ambiente

| Ferramenta           | Fornecedor       | Função Principal                        | Uso Típico / Observações                          |
|---------------------|------------------|---------------------------------------|--------------------------------------------------|
| OMEGAMON for DB2    | IBM              | Monitoramento em tempo real            | Threads, locks, I/O, buffer pools                 |
| DB2PM / Query Monitor | IBM              | Análise de SQLs e estatísticas         | SQLs pesadas, custo, execução                      |
| DISPLAY Commands    | IBM              | Comandos diretos no DB2                 | -DIS THD, -DIS DDF, -DIS LOCKS, etc.              |
| DSNZPARM Display    | IBM              | Verificação de parâmetros DB2           | Tuning, configuração de ambiente                   |
| DSN1LOGP            | IBM              | Análise de logs de recovery             | Diagnóstico de UNDO/REDO                            |
| SMF (101/102/115/116) | IBM             | Coleta de métricas z/OS                 | Waits, locks, tempo CPU, análise histórica          |
| MainView for DB2    | BMC              | Monitoramento gráfico                   | Alternativa ao Omegamon, amigável                   |
| BMC Apptune         | BMC              | Análise de SQLs e performance           | Top SQL, CPU, buffer hits                            |
| BMC Log Master      | BMC              | Análise de logs e UNDO                   | Auditoria, rollback manual                           |
| Catalog Manager     | BMC              | Gerenciamento de objetos DB2             | Visualização rápida de tabelas, índices, etc.       |
| CA Insight for DB2  | CA/Broadcom      | Monitoramento de sessões/locks          | Semelhante ao Omegamon/MainView                      |
| RC/Query            | CA/Broadcom      | Consulta ao catálogo DB2                 | Visual de estrutura e objetos                        |
| RC/Update           | CA/Broadcom      | Alteração segura de objetos              | Manutenção de estruturas e dados                     |
| CEMT / CEDF / CEEMT | IBM (CICS)       | Diagnóstico de transações online        | Transações DB2 em CICS                               |
| CICS Explorer       | IBM              | Monitoramento e debugging CICS           | Visão geral com Omegamon                             |
| Splunk / Zabbix / Nagios | Vários        | Observabilidade e alertas                 | Dashboards e alertas via integração                  |
| Grafana com Zowe    | Open Mainframe   | Visualização de métricas z/OS             | Painéis remotos modernos                              |
| Zowe CLI            | Open Mainframe   | Acesso CLI ao z/OS                        | Execução remota de comandos z/OS                      |

---

## 4. Procedimentos Práticos para Verificação de Abend 

### 1. Coleta Inicial

- Nome do job/programa/transação  
- Data e hora do abend  
- Código do abend (ex: S0C7, U4038, SQLCODE)  
- Logs JES (JESMSGLG, JESYSMSG, JESJCL)  
- Último SQL executado  
- Dump gerado? (SYSUDUMP, SYSMDUMP)  

Solicite ao operador ou suporte que preencha o checklist [Checklist_Abend_DB2_Produção.xlsx](sandbox:/mnt/data/Checklist_Abend_DB2_Produção.xlsx).

### 2. Diagnóstico Imediato

Execute ou solicite os comandos DB2:

```text
-DIS THREAD(*) TYPE(ACTIVE)
-DIS DDF DETAIL
-DIS LOCKS(S)
-DIS BUFFERPOOL(*)

## 5. Procedimentos Práticos de Monitoramento na CEF

### 5.1 JOB está executando por muitas horas em produção

Passos para investigação:

1. **Verifique no SDSF/MAINVIEW/OMEGAMON se há locks ativos.**
   - Veja se o JOB está em HOLD, LOCKWAIT ou SUSPEND.
   - Identifique o recurso travado.

2. **Verifique plano de acesso.**
   - Um REBIND automático pode ter gerado plano ruim.
   - Use `EXPLAIN` ou ferramentas como DB2PM para ver plano gerado.
   - Verifique se estatísticas estão desatualizadas.

3. **Compare com ambiente de desenvolvimento.**
   - Diferenças no volume de dados?
   - Índices existentes no DEV também estão no PROD?
   - Estatísticas atualizadas em ambos?

4. **Traces ou monitoramento.**
   - Habilitar IFCID 58, 172, 196 para SQLs longas e locks.
   - Pode-se usar comando:
     ```
     -START TRACE(PERFM) CLASS(3) DEST(GTF)
     ```

5. **Uso de ferramentas CEF**
   - Use painel de locking do OMEGAMON (LC panel).
   - Verifique `Current Threads`, `DB2 Statistics`, `Thread Detail`.

---

## 6. Causas Comuns de Execução Prolongada de JOB

| Causa                           | Explicação                                                                                   | Ação Recomendada                                                                 |
|--------------------------------|-----------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------|
| **Falta de índice**            | O otimizador escolhe tabelascan completo, gerando alto custo.                                | Criar índice adequado e/ou atualizar estatísticas; rebind após criação.           |
| **REBIND automático ruim**     | Pode ocorrer após alterações em tabela ou RUNSTATS.                                          | Verifique plano de acesso. Faça REBIND manual com EXPLAIN para avaliar plano.     |
| **Estatísticas desatualizadas**| Otimizador toma decisões ruins sem dados precisos.                                           | Executar RUNSTATS atualizados e REBIND.                                           |
| **Locks e contenção**          | Threads concorrentes esperam liberação de recursos.                                          | Analisar OMEGAMON / Mainview e aplicar timeout ou ISOLATION adequado.             |
| **Volume de dados diferente**  | Ambiente PROD com milhões de linhas, enquanto DEV é pequeno.                                 | Testar com volume similar em QA antes de produção.                                |
| **Parâmetros de JOB diferentes**| JOB em PROD pode ter parâmetros (datas, ranges) muito maiores.                              | Validar se os parâmetros usados são compatíveis com performance esperada.         |
| **Segmentação em partições**   | Tabela particionada pode estar concentrando acesso em poucos partições.                     | Avaliar distribuição dos dados e particionamento.                                 |
| **Problemas de espaço**        | Tabelaspace cheio ou índice corrompido causa degradação.                                     | Usar DSN1COPY, DISPLAY DATABASE para verificar anomalias e REORG, se necessário.  |

---

## 7. Ações Corretivas e Preventivas

### Corretivas
- REBIND com EXPLAIN e análise de plano.
- RUNSTATS atualizado antes de REBIND.
- CANCEL THREAD em casos de deadlock crítico.
- ALTERações de índice.
- Suporte de storage, se espaço for causa.

### Preventivas
- Agendar RUNSTATS periódico em tabelas críticas.
- REBIND planejado após alterações estruturais.
- Simular execução com dados representativos em QA.
- Monitoramento contínuo com alertas de tempo de execução.
- Documentação de dependências do job e versões dos objetos.

---

