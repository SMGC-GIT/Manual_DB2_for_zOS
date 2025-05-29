# Comandos Úteis para DB2 for z/OS

### Comandos Básicos

| Comando | Descrição |
|---------|-----------|
| `-DISPLAY DB(SILVD000) SP(*) LIMIT(*) USE` | Exibe o status de todas as tabelas no banco de dados `SILVD000`, incluindo informações de uso e limites. |
| `-DISPLAY THREAD(*)` | Exibe informações sobre todas as threads ativas no sistema DB2. |
| `-DISPLAY DATABASE(SILVD000) SP(*)` | Exibe o status de todas as tabelas no banco de dados `SILVD000`. |
| `-DISPLAY UTILITY(*)` | Exibe o status de todos os utilitários em execução. |
| `-DISPLAY BUFFERPOOL(*)` | Exibe informações sobre todos os buffer pools. |
| `-DISPLAY LOG` | Exibe informações sobre os logs do DB2. |
| `-DISPLAY STATS` | Exibe estatísticas de desempenho do DB2. |
| `-DISPLAY DDF` | Exibe o status do Distributed Data Facility (DDF). |
| `-DISPLAY GROUP` | Exibe informações sobre o grupo de dados compartilhados. |
| `-DISPLAY PROCEDURE(*)` | Exibe informações sobre todos os procedimentos armazenados. |
| `-DISPLAY FUNCTION(*)` | Exibe informações sobre todas as funções definidas pelo usuário. |
| `-DISPLAY PACKAGE(*)` | Exibe informações sobre todos os pacotes de aplicação. |

### Comandos Intermediários

| Comando | Descrição |
|---------|-----------|
| `-START DB(SILVD000) SP(*) ACCESS(UT)` | Inicia todas as tabelas no banco de dados `SILVD000` com acesso de utilitário. |
| `-STOP DB(SILVD000) SP(*)` | Para todas as tabelas no banco de dados `SILVD000`. |
| `-RECOVER DB(SILVD000) SP(*)` | Recupera todas as tabelas no banco de dados `SILVD000` para um ponto de recuperação específico. |
| `-BIND PACKAGE` | Cria um pacote de aplicação. |
| `-REBIND PACKAGE` | Recompila um pacote de aplicação existente. |
| `-FREE PACKAGE` | Exclui um pacote de aplicação específico. |
| `-BIND PLAN` | Cria um plano de aplicação. |
| `-REBIND PLAN` | Recompila um plano de aplicação existente. |
| `-FREE PLAN` | Exclui um plano de aplicação específico. |

### Comandos Avançados

| Comando | Descrição |
|---------|-----------|
| `-RUNSTATS` | Coleta estatísticas de desempenho para tabelas e índices. |
| `-REORG` | Reorganiza tabelas e índices para melhorar a performance. |
| `-COPY` | Cria uma cópia de backup de tabelas e índices. |
| `-QUIESCE` | Coloca tabelas e índices em um estado de quiescência para manutenção. |

## Explicações dos Componentes dos Comandos

- `DB`: Database (Banco de Dados)
- `SP`: Storage Pool (Pool de Armazenamento)
- `LIMIT`: Limite de recursos ou operações
- `USE`: Uso atual dos recursos
- `ACCESS`: Tipo de acesso (por exemplo, `UT` para utilitário)
- `THREAD`: Thread de execução no DB2
- `UTILITY`: Utilitário de manutenção ou operação
- `BUFFERPOOL`: Pool de buffers de memória
- `LOG`: Logs de transações
- `STATS`: Estatísticas de desempenho
- `DDF`: Distributed Data Facility (Facilidade de Dados Distribuídos)
- `GROUP`: Grupo de dados compartilhados
- `PROCEDURE`: Procedimento armazenado
- `FUNCTION`: Função definida pelo usuário
- `PACKAGE`: Pacote de aplicação
- `PLAN`: Plano de aplicação
- `RUNSTATS`: Coleta de estatísticas
- `REORG`: Reorganização de tabelas e índices
- `COPY`: Cópia de backup
- `QUIESCE`: Estado de quiescência para manutenção

## Como Executar os Comandos

### TSO (Time Sharing Option)

1. Inicie uma sessão TSO e use o comando `DSN` para entrar no ambiente do DB2:
   ```plaintext
   TSO DSN SYSTEM(DB2SSID)
   ```

2. Depois de entrar no ambiente do DB2, execute os comandos desejados:

```plaintext
-DISPLAY DB(SILVD000) SP(*) LIMIT(*) USE
```

 
### ISPF (Interactive System Productivity Facility)

1. No ISPF, acesse o menu de comandos do DB2 usando a opção DB2I e selecione a opção 7 para comandos do DB2.
2. No menu de comandos do DB2, insira os comandos diretamente:
 
```plaintext
-DISPLAY DB(SILVD000) SP(*) LIMIT(*) USE
```


### Execução em Lote (Batch)
1. Crie um job JCL para executar comandos do DB2 em lote:

```plaintext
//JOBNAME  JOB ...
//STEP01   EXEC PGM=IKJEFT01
//SYSTSPRT DD SYSOUT=*
//SYSTSIN  DD *
DSN SYSTEM(DB2SSID)
-DISPLAY DB(SILVD000) SP(*) LIMIT(*) USE
-DISPLAY THREAD(*)
```

---

## 🛠️ Como Executar Comandos `-DISPLAY` no Db2 for z/OS

Os comandos `-DISPLAY` devem ser executados no ambiente z/OS através de:

- **Console SDSF** (linha de comando no TSO)
- **Tool como SPUFI, QMF ou JCL** (em algumas situações)
- **Painéis de administração do Db2I (DSN command interface)**

### ✅ Exemplo via console:

```plaintext
-DIS DB(SILVD000) SP(*) LIMIT(*) USE RESTRICT
```

> 🔹 O comando inicia com `-DIS`, pode usar tanto `-DISPLAY` quanto `-DIS` como forma abreviada.  
> 🔹 O prefixo `-` é necessário para indicar comandos de sistema.

---

## 🚦 Significados de Status Retornados pelos Comandos `-DISPLAY`

Abaixo estão os principais **status que podem aparecer** ao executar comandos `-DISPLAY`, especialmente com `-DISPLAY DATABASE`, e o que fazer em cada caso:

| **Status**     | **Significado**                                                                 | **Ação Recomendada**                                                                 |
|----------------|----------------------------------------------------------------------------------|---------------------------------------------------------------------------------------|
| `RO`           | Read Only – tablespace ou tabela está disponível somente para leitura           | Verifique se o recurso foi posto em `STOP ACCESS(RO)`. Usar `-START` se necessário.  |
| `UT`           | Utilities – objeto está sendo usado por algum utilitário                        | Espere o término do utilitário ou investigue com `-DIS UTIL(*)`.                    |
| `COPY`         | COPY Pending – requer backup via COPY                                            | Execute um `COPY` com utilitário para liberar o uso.                                |
| `CHKP`         | Check Pending – falha na integridade referencial ou carga não validada          | Use `CHECK DATA` ou `LOAD REPLACE ENFORCE` para validar os dados.                   |
| `LPL`          | Logical Page List – páginas inconsistentes, requer recuperação manual           | Use `START DATABASE ... SP ... ACCESS(FORCE)` para tentar limpar o LPL.             |
| `RBDP`         | Rebuild Pending – índice precisa ser reconstruído                               | Use utilitário `REBUILD INDEX` para recriar os índices afetados.                    |
| `STOP`         | Objeto está parado (`STOP` manual ou automático)                                | Use `-START DATABASE(...)` ou `-START INDEX(...)`.                                   |
| `ADB`          | Advisory Reorg – objeto recomenda reorganização por degradação de performance    | Planeje um `REORG TABLESPACE` ou `REORG INDEX` conforme impacto.                    |
| `AREO*`        | After Reorg – objeto precisa de reorganização adicional após alterações          | Execute `REORG` com `REPAIR SET CURRENT`.                                            |

---

## 💡 Dica Avançada: Interpretando o Output

Quando executar:

```plaintext
-DIS DB(SILVD000) SP(*) LIMIT(*) USE RESTRICT
```

Você verá algo como:

```plaintext
DATABASE = SILVD000  STATUS = RW
   SPACENAM = TBCLI01   STATUS = RW
   SPACENAM = TBPED01   STATUS = CHKP COPY
```

> 🔍 Isso indica que `TBPED01` está com dois problemas: pendente de `CHECK` e requer backup com `COPY`.

---

## 🔄 Ações Comuns com Base nos Status

### ✅ Liberar CHECK PENDING:

```sql
CHECK DATA TABLESPACE DBNAME.TBSPACE_NAME;
```

### ✅ Resolver COPY PENDING:

Execute utilitário:

```plaintext
COPY TABLESPACE DBNAME.TBSPACE_NAME FULL YES SHRLEVEL REFERENCE
```

### ✅ Retirar objeto de LPL:

```plaintext
-START DATABASE(DBNAME) SP(TBSPACE_NAME) ACCESS(FORCE)
```

---

## 📚 Referência IBM Oficial

- [Db2 for z/OS -DISPLAY Commands (IBM)](https://www.ibm.com/docs/en/db2-for-zos/12.0.0?topic=commands-display-database-db2)
- [Db2 Command Reference Book (PDF)](https://www.ibm.com/docs/SSEPEK/pdf/db2z_12_comrefbook.pdf)

---

## 🧯 Comandos Corretivos no Db2 for z/OS

Esta seção reúne os principais comandos utilizados para **corrigir situações operacionais** em objetos Db2 como *tablespaces*, *índices* e *databases*, comuns na rotina do DBA de desenvolvimento.

---

### 🔄 1. Remover `CHECK PENDING` (CHKP)

**📌 Situação:** Após um `LOAD`, a integridade referencial pode estar comprometida.

**✅ Comando corretivo:**

```sql
CHECK DATA TABLESPACE DBNAME.TBSPACE_NAME;
```

**💡 Dica:** Use apenas se os dados carregados estão corretos, ou ocorrerá falha.

**🔗 Referência IBM:**  
[CHECK DATA - IBM Db2 for z/OS](https://www.ibm.com/docs/en/db2-for-zos/13.1?topic=statements-check-data)

---

### 💾 2. Resolver `COPY PENDING` (COPY)

**📌 Situação:** O objeto requer backup após um `LOAD`, `REORG` ou `RECOVER`.

**✅ Comando corretivo:**

```plaintext
COPY TABLESPACE DBNAME.TBSPACE_NAME FULL YES SHRLEVEL REFERENCE
```

**💡 Dica:** Use `FULL YES` para garantir cobertura completa e liberar o status.

**🔗 Referência IBM:**  
[COPY Utility - IBM Db2 for z/OS](https://www.ibm.com/docs/en/db2-for-zos/13.1?topic=utilities-copy)

---

### 🧹 3. Limpar `LPL` (Logical Page List)

**📌 Situação:** Páginas corrompidas ou inconsistentes após falha.

**✅ Comando corretivo:**

```plaintext
-START DATABASE(DBNAME) SP(TBSPACE_NAME) ACCESS(FORCE)
```

**💡 Dica:** Verifique com `-DIS DB(...) SP(...) LIMIT(*)` antes.

**🔗 Referência IBM:**  
[START DATABASE - IBM Db2 for z/OS](https://www.ibm.com/docs/en/db2-for-zos/13.1?topic=commands-start-database)

---

### 🛠️ 4. Corrigir `RBDP` (Rebuild Pending)

**📌 Situação:** Índice está pendente de reconstrução.

**✅ Comando corretivo:**

```plaintext
REBUILD INDEX(IXOWNER.IXNAME)
```

**💡 Dica:** Verifique se o índice está inutilizável via `-DISPLAY INDEX(...)`.

**🔗 Referência IBM:**  
[REBUILD INDEX Utility - IBM Db2 for z/OS](https://www.ibm.com/docs/en/db2-for-zos/13.1?topic=utilities-rebuild-index)

---

### 🚀 5. Iniciar objetos parados (`STOP`)

**📌 Situação:** Objeto foi manualmente ou automaticamente parado.

**✅ Comando corretivo:**

```plaintext
-START DATABASE(DBNAME) SP(TBSPACE_NAME)
```

**💡 Dica:** Sempre verifique o status antes com `-DISPLAY DATABASE(...)`.

**🔗 Referência IBM:**  
[START DATABASE Command - IBM Docs](https://www.ibm.com/docs/en/db2-for-zos/13.1?topic=commands-start-database)

---

### 🔧 6. Retirar objetos de `AREO*` (After Reorg Pending)

**📌 Situação:** Após um `REORG`, o objeto precisa de reorganização adicional.

**✅ Comando corretivo:**

```plaintext
REORG TABLESPACE DBNAME.TBSPACE_NAME REPAIR SET CURRENT
```

**💡 Dica:** Pode ser necessário após DROP COLUMN, alterar VARCHAR etc.

**🔗 Referência IBM:**  
[REORG Utility - IBM Db2 for z/OS](https://www.ibm.com/docs/en/db2-for-zos/13.1?topic=utilities-reorg)

---

### 🧩 7. Finalizar Utilitários em Conflito

**📌 Situação:** Um utilitário travado ou inativo permanece registrado.

**✅ Comando corretivo:**

```plaintext
-TERM UTIL(UTILID)
```

**💡 Dica:** Liste utilitários ativos com:

```plaintext
-DIS UTIL(*)
```

**🔗 Referência IBM:**  
[TERM UTIL Command - IBM Db2 for z/OS](https://www.ibm.com/docs/en/db2-for-zos/13.1?topic=commands-term-util)

---

> 🧠 **Dica importante:** Sempre avalie o impacto dos comandos corretivos com base em:
> - Volume de dados
> - Ambiente (produção x teste)
> - Workload concorrente
> - Backup disponível

---

## 📘 Referência Geral

- [Comandos do Db2 for z/OS - IBM Documentation](https://www.ibm.com/docs/en/db2-for-zos/13.1?topic=commands-db2)

---


