# 📁 Tuning e Performance no DB2 for z/OS (Detalhado)

## 🔍 Identificação de Problemas de Performance

Antes de otimizar, precisamos identificar exatamente onde está o problema. Abaixo estão os principais sintomas, formas de identificação, exemplos de SQLCODEs e como interpretá-los.

### 1. Alto Tempo de Resposta

- **O que é:** Consultas ou transações que demoram muito para retornar resultados, prejudicando o usuário final e o sistema.
- **Como identificar:**
  - Monitoramento do tempo médio de resposta no DB2 PM, SMF ou ferramentas de monitoramento.
  - Logs de aplicações mostrando tempos altos de execução.
  - EXPLAIN indicando Full Table Scan em tabelas grandes.
- **Possíveis causas:**  
  - Falta de índice adequado.  
  - Estatísticas desatualizadas (RUNSTATS não executado).  
  - SQL mal escrito (ex: uso excessivo de funções, SELECT *).  
- **SQLCODE relacionado:**  
  - Normalmente não gera SQLCODE de erro, mas o SQLCODE 0 indica sucesso com tempo potencialmente alto.

### 2. Excesso de CPU ou I/O

- **O que é:** Processos que consomem muito CPU ou I/O, impactando a performance geral do sistema.
- **Como identificar:**  
  - Análise dos gráficos de CPU e I/O via SMF, RMF ou monitoramento DB2 PM.  
  - IFCID 003 mostra lock waits e I/O elevados.  
  - Logs do sistema operacional.  
- **Possíveis causas:**  
  - Scans completos desnecessários (full table scans).  
  - Índices mal configurados ou ausentes.  
  - Consultas com joins ineficientes ou subqueries mal formuladas.  
- **SQLCODE relacionado:**  
  - SQLCODE 200 (row not found) pode indicar queries que varrem toda a tabela buscando algo que não existe.  
  - SQLCODEs negativos indicam erro, não diretamente relacionado a consumo.

### 3. Locking e Deadlocks

- **O que é:** Conflitos entre transações que acessam os mesmos recursos simultaneamente, levando a esperas ou abortos.
- **Como identificar:**  
  - IFCID 003 registra informações detalhadas de locks e deadlocks.  
  - Monitorar eventos de lock wait e deadlock no SMF.  
  - Mensagens de erro no sistema ou logs da aplicação.  
- **Erros/SQLCODE comuns:**  
  - **SQLCODE -911** (Deadlock ou timeout): Transação é terminada para resolver deadlock.  
  - **SQLCODE -913** (Lock timeout): Tempo limite excedido aguardando lock.  
- **Ações recomendadas:**  
  - Revisar lógica de transações para reduzir tempo de locks.  
  - Ajustar nível de isolamento, quando possível.  
  - Utilizar acesso por cursores estáticos para minimizar bloqueios.  

### 4. Wait Times Altos

- **O que é:** Tempos excessivos aguardando recursos, como I/O, CPU, locks, ou recursos do sistema.
- **Como identificar:**  
  - Monitoramento de performance para identificar tempos de espera (wait times).  
  - IFCID 014 e 015 podem ajudar a rastrear esperas.  
- **Possíveis causas:**  
  - Contenção por locks.  
  - Saturação de recursos como buffer pools, sort pools.  
  - Problemas no subsistema de armazenamento.  
- **SQLCODE:**  
  - Normalmente não gera SQLCODE específico, mas os problemas refletem em performance degradada.  

### 5. Planos de Acesso Ineficientes

- **O que é:** O otimizador escolhe um caminho de execução que não é o mais eficiente, resultando em scans desnecessários, muitos I/Os e CPU.
- **Como identificar:**  
  - Utilizar o EXPLAIN para analisar o plano de execução.  
  - Observar se o plano usa índices, ou se faz Full Table Scan.  
- **Indicadores no plano:**  
  - High-cost estimado.  
  - High number of rows acessados.  
- **SQLCODE:**  
  - Normalmente sucesso (SQLCODE 0), mas com performance ruim.

---

## 🛠️ EXPLAIN: Ferramenta Essencial para Análise de Performance

O EXPLAIN é fundamental para entender como o otimizador DB2 planeja executar o SQL. Ele revela o caminho escolhido, índices usados, tipos de acesso e estimativas de custo.

### Como funciona

- O comando EXPLAIN gera informações detalhadas sobre o plano de acesso antes da execução do SQL.  
- Essas informações são armazenadas em tabelas específicas do catálogo DB2 (EXPLAIN_INSTANCE, EXPLAIN_OBJECT, EXPLAIN_STATEMENT, etc).

### Comando básico:

```sql
EXPLAIN PLAN FOR
SELECT NOME, ENDERECO FROM CLIENTES WHERE ID = 123;
