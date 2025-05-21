# 📊 Estatísticas e Otimizador no DB2 for z/OS

## 🧠 Visão Geral

O **otimizador** do DB2 for z/OS é responsável por determinar o caminho de acesso mais eficiente para executar uma instrução SQL. Para tomar decisões informadas, o otimizador depende de **estatísticas atualizadas** sobre os dados e estruturas do banco. Essas estatísticas são armazenadas em tabelas de catálogo e são coletadas principalmente por meio do utilitário `RUNSTATS`.

---

## 🛠️ Utilitário RUNSTATS

### 📌 O que é?

O `RUNSTATS` é um utilitário do DB2 que coleta estatísticas sobre tabelas, índices e tablespaces. Essas informações incluem:

- Número de linhas
- Número de páginas
- Distribuição de valores em colunas
- Eficiência de índices

Essas estatísticas são armazenadas em tabelas do catálogo e são essenciais para o otimizador escolher planos de acesso eficientes.

### 🔧 Sintaxe Básica

```sql
RUNSTATS TABLESPACE DBNAME.TSNAME
  TABLE(ALL)
  INDEX(ALL)
  FREQVAL NUMCOLS 10 COUNT 10
  HISTOGRAM
  REPORT YES;
```

### 🧩 Explicação dos Parâmetros

- `TABLESPACE DBNAME.TSNAME`: Especifica o tablespace alvo.
- `TABLE(ALL)`: Coleta estatísticas para todas as tabelas no tablespace.
- `INDEX(ALL)`: Coleta estatísticas para todos os índices associados.
- `FREQVAL NUMCOLS 10 COUNT 10`: Coleta os 10 valores mais frequentes para até 10 colunas.
- `HISTOGRAM`: Coleta estatísticas de histograma para colunas especificadas.
- `REPORT YES`: Gera um relatório com as estatísticas coletadas.

---

## 🗂️ Tabelas de Catálogo Relacionadas

As estatísticas coletadas pelo `RUNSTATS` são armazenadas nas seguintes tabelas de catálogo:

- `SYSIBM.SYSTABLES`: Informações sobre tabelas.
- `SYSIBM.SYSCOLUMNS`: Detalhes sobre colunas.
- `SYSIBM.SYSINDEXES`: Informações sobre índices.
- `SYSIBM.SYSCOLDIST`: Estatísticas de distribuição de valores.
- `SYSIBM.SYSCOLSTATS`: Estatísticas detalhadas de colunas.
- `SYSIBM.SYSTABLESPACE`: Estatísticas de tablespaces.
- `SYSIBM.SYSTABLEPART`: Estatísticas de partições de tablespaces.
- `SYSIBM.SYSINDEXPART`: Estatísticas de partições de índices.

---

## 📈 Importância das Estatísticas Atualizadas

Estatísticas desatualizadas podem levar o otimizador a escolher planos de acesso ineficientes, resultando em:

- Aumento no tempo de resposta das consultas.
- Uso excessivo de recursos do sistema.
- Bloqueios e contenções desnecessárias.

É recomendável executar o `RUNSTATS` após:

- Grandes volumes de inserções, atualizações ou exclusões.
- Criação ou alteração de índices.
- Reorganizações de tabelas ou índices (`REORG`).

---

## 🧪 Coleta de Estatísticas Avançadas

### 🎯 Estatísticas de Distribuição

Coletar estatísticas de distribuição ajuda o otimizador a entender a dispersão dos dados, especialmente em colunas com distribuição desigual.

```sql
RUNSTATS TABLESPACE DBNAME.TSNAME
  TABLE(ALL)
  INDEX(ALL)
  FREQVAL NUMCOLS 10 COUNT 10
  HISTOGRAM;
```

### 🔄 Perfis de Estatísticas

Os perfis permitem definir um conjunto padrão de opções para o `RUNSTATS`, facilitando execuções consistentes.

```sql
RUNSTATS TABLESPACE DBNAME.TSNAME
  TABLE(ALL)
  INDEX(ALL)
  SET PROFILE;
```

Para utilizar um perfil existente:

```sql
RUNSTATS TABLESPACE DBNAME.TSNAME
  USE PROFILE;
```

---

## ✅ Boas Práticas

- **Automatize** a execução do `RUNSTATS` em horários de baixa atividade.
- **Monitore** regularmente as estatísticas e atualize-as conforme necessário.
- **Evite** executar o `RUNSTATS` durante operações críticas para minimizar impacto no desempenho.
- **Utilize** perfis para padronizar a coleta de estatísticas em ambientes complexos.

---

## 📚 Para mais informações técnicas

Consulte os seguintes tópicos na documentação oficial da IBM:

- 🔹 [Db2 RUNSTATS utility](https://www.ibm.com/docs/en/db2-for-zos/12?topic=utilities-runstats)
  > Detalhes sobre o utilitário RUNSTATS, incluindo opções e considerações de uso.

- 🔹 [RUNSTATS TABLESPACE syntax and options](https://www.ibm.com/docs/en/db2-for-zos/12?topic=runstats-tablespace-syntax-options)
  > Sintaxe completa e opções disponíveis para o comando RUNSTATS em tablespaces.

- 🔹 [SYSTABLES catalog table](https://www.ibm.com/docs/en/db2-for-zos/12?topic=tables-systables)
  > Informações sobre a tabela de catálogo SYSTABLES.

- 🔹 [SYSINDEXES catalog table](https://www.ibm.com/docs/en/db2-for-zos/12.0.0?topic=tables-sysindexes)
  > Detalhes sobre a tabela de catálogo SYSINDEXES.

---

*Última atualização: 20/05/2025*
