## BACKUP E RECUPERAÇÃO NO DB2 FOR Z/OS

O processo de backup e recuperação é essencial para garantir a integridade e disponibilidade dos dados em ambientes críticos, como o DB2 for z/OS. Diferente de SGBDs mais simples, o DB2 for z/OS oferece um conjunto robusto de utilitários nativos para realizar essas tarefas com confiabilidade em ambientes de grande porte.

Neste módulo, vamos abordar os principais recursos, estratégias e utilitários relacionados ao **Backup e Recuperação**, incluindo:

- COPY
- MERGECOPY
- RECOVER
- QUIESCE
- MODIFY RECOVERY
- REPORT RECOVERY
- DSN1COPY (em alguns cenários avançados)
- DSN1LOGP (para análise de logs em recuperação)
- Estratégias de RTO/RPO
- Considerações sobre BACKUP SYSTEM (z/OS-level)

---

## PRINCIPAIS PONTOS SOBRE BACKUP E RECUPERAÇÃO

### 🔹 Objetivo Geral
Garantir a **disponibilidade dos dados**, mesmo em cenários de falha (física ou lógica), com base em pontos de recuperação consistentes.

### 🔹 Tipos de Backup
- **COPY (FULL ou INCREMENTAL)**: Gera uma imagem física dos datasets do tablespace ou índice.
- **MERGECOPY**: Consolida múltiplos backups incrementais em uma única imagem full.
- **BACKUP SYSTEM** (z/OS): Cópia de volume físico no nível do sistema operacional (em ambientes SMS-managed).
- **Image Copies Automatizadas via SHRLEVEL CHANGE** (sem downtime).

### 🔹 Tipos de Recuperação
- **RECOVER TOCOPY**: Recupera dados usando uma imagem específica.
- **RECOVER TORBA / TOLOGPOINT / TOLRSN**: Recupera até um ponto no tempo ou log.
- **QUIESCE**: Garante um ponto consistente para possíveis recoveries lógicos.
- **DSN1COPY**: Utilitário especializado, usado com cautela para copiar blocos físicos entre datasets.

### 🔹 Gerenciamento de Backups
- **MODIFY RECOVERY**: Apaga entradas antigas do SYSCOPY e datasets de backup.
- **REPORT RECOVERY**: Lista quais backups estão disponíveis para determinado objeto.
- **Automação via JOBs cíclicos e ferramentas de agendamento**.

### 🔹 Boas Práticas
- Realizar **COPY FULL periódica**, com backups incrementais entre elas.
- Usar **QUIESCE** em sistemas com transações críticas, para marcar pontos de recuperação consistentes.
- Testar **RECOVER** em ambientes de homologação.
- Manter o histórico controlado com o **MODIFY RECOVERY**, evitando crescimento desnecessário do catálogo.
- Avaliar políticas de RPO (Recovery Point Objective) e RTO (Recovery Time Objective).

### 🔹 Considerações Avançadas
- **BACKUP SYSTEM** pode ser integrado a soluções de Disaster Recovery (ex: GDPS/PPRC).
- **RECOVER TORBA** exige que todos os logs estejam disponíveis (incluindo archive logs).
- A falha de RECOVER pode ser analisada com auxílio de **DSN1LOGP** e suporte IBM.

---

## EXEMPLOS DE JCL PARA OPERAÇÕES DE BACKUP E RECUPERAÇÃO

### 🧾 COPY FULL
```jcl
//COPYTS  EXEC DSNUPROC,UID='COPYTS',UTPROC='',
//         SYSTEM='DSN',UIDX='001'
//SYSPRINT DD SYSOUT=*
//SYSIN    DD *
  COPY TABLESPACE DB01.TS01 FULL YES SHRLEVEL REFERENCE
/*
//*
//SYSREC00 DD DSN=DB01.TS01.COPY1,
//         DISP=(NEW,CATLG,CATLG),
//         SPACE=(CYL,(50,10),RLSE),
//         UNIT=SYSDA
```

### 🧾 MERGECOPY
```jcl
//MERGE   EXEC DSNUPROC,UID='MERGE',UTPROC='',
//         SYSTEM='DSN',UIDX='002'
//SYSPRINT DD SYSOUT=*
//SYSIN    DD *
  MERGECOPY TABLESPACE DB01.TS01
/*
//*

```

### 🧾 RECOVER TORBA
```jcl
//RCVRBA  EXEC DSNUPROC,UID='RCVRBA',UTPROC='',
//         SYSTEM='DSN',UIDX='003'
//SYSPRINT DD SYSOUT=*
//SYSIN    DD *
  RECOVER TABLESPACE DB01.TS01
          TORBA X'00000000012345'
/*
//*

```

### 🧾 MODIFY RECOVERY
```jcl
//MODRCVR EXEC DSNUPROC,UID='MODRCV',UTPROC='',
//         SYSTEM='DSN',UIDX='004'
//SYSPRINT DD SYSOUT=*
//SYSIN    DD *
  MODIFY RECOVERY TABLESPACE DB01.TS01
         RETAIN LAST 2
/*
//*

```

### 🧾 REPORT RECOVERY
```jcl
//RPTRECOV EXEC DSNUPROC,UID='RPTRECOV',UTPROC='',
//          SYSTEM='DSN',UIDX='005'
//SYSPRINT DD SYSOUT=*
//SYSIN    DD *
  REPORT RECOVERY TABLESPACE DB01.TS01
/*
//*

```

---

## 🔐 DETALHAMENTO  - Tipos de Backup no DB2 for z/OS

O DB2 for z/OS oferece diferentes formas de realizar **backup lógico e físico**, utilizando utilitários específicos ou recursos nativos do sistema. A escolha da abordagem depende do nível de granularidade necessário, do impacto permitido no ambiente e dos objetivos de RTO/RPO definidos.

---

### 1. COPY (FULL ou INCREMENTAL)

O utilitário **COPY** é o principal mecanismo nativo de backup no DB2.

#### ✅ COPY FULL
- Realiza um **backup completo** do *tablespace* ou *índice* especificado.
- Pode ser executado com:
  - `SHRLEVEL REFERENCE`: Garante consistência, exige locking.
  - `SHRLEVEL CHANGE`: Permite acesso concorrente, mas exige logs para tornar a cópia consistente durante o RECOVER.

#### ✅ COPY INCREMENTAL
- Gera backup **somente das páginas modificadas** desde a última cópia full.
- Exige que o objeto tenha sido previamente copiado com COPY FULL.
- Menor volume de dados → útil para janelas curtas ou backups frequentes.

#### 🎯 Notas:
- As cópias são registradas no catálogo (`SYSCOPY`).
- É possível definir múltiplos datasets de saída.
- Pode ser armazenado em fita ou disco.
- Usado diretamente no utilitário **RECOVER**.

---

### 2. MERGECOPY

- Consolida múltiplas **cópias incrementais** em uma nova imagem de backup full.
- Reduz o número de arquivos necessários durante o RECOVER.
- Elimina a necessidade de manter diversas cópias pequenas.
- Opera somente sobre backups existentes.

#### Exemplo prático:
- Após 1 COPY FULL + 3 COPY INCREMENTAL, executar MERGECOPY para gerar uma nova imagem completa reutilizável.

---

### 3. BACKUP SYSTEM (z/OS)

- Recurso do z/OS que permite criar **cópias ponto-a-ponto (point-in-time)** dos volumes (DASD).
- Utiliza o utilitário DFSMShsm (HSM).
- Pode ser integrado a ambientes com **GDPS** e **Disaster Recovery**.
- É controlado por política de SMS.

#### Requisitos:
- O ambiente precisa ser SMS-managed.
- O DB2 deve estar em modo *log suspend* temporário para garantir consistência.

#### Quando usar:
- Recuperações rápidas de múltiplos objetos.
- Proteção em nível de volume para desastres.

---

### 4. DSN1COPY (Backup físico específico)

- Utilitário de **cópia bit a bit** de datasets do DB2.
- Não registra no catálogo (`SYSCOPY`).
- Pode ser usado para:
  - Restaurar imagens fora do DB2.
  - Copiar tabelas individuais entre ambientes.
  - Analisar corrupção de dados.

#### Cuidado:
- Deve ser utilizado por especialistas.
- Requer conhecimento de catálogo e mapeamento de objetos físicos.
- Pode corromper dados se usado incorretamente.

---

### 5. BACKUP por Storage Externo (Snapshot)

- **Snapshots** feitos por ferramentas de storage (IBM FlashCopy, EMC TimeFinder, etc.).
- Costuma ser usado para acelerar cópias e replicações.
- Exige que o DB2 esteja em modo consistente ou que um QUIESCE seja feito antes da cópia.

---

### ✅ Comparativo Rápido

| Tipo de Backup     | Utilitário       | Tipo         | Registrado no SYSCOPY | Recomendado para |
|--------------------|------------------|--------------|------------------------|------------------|
| COPY FULL          | COPY             | Lógico/Físico| Sim                    | Base de recuperação |
| COPY INCREMENTAL   | COPY             | Lógico       | Sim                    | Backups frequentes |
| MERGECOPY          | MERGECOPY        | Lógico       | Sim                    | Consolidação de cópias |
| BACKUP SYSTEM      | DFSMShsm/z/OS    | Físico       | Não                    | DR e snapshot full |
| DSN1COPY           | DSN1COPY         | Físico       | Não                    | Restauração técnica |
| Snapshot Storage   | Externo (Flash)  | Físico       | Não                    | Replicação e backup rápido |

---


## 💾 DETALHAMENTO - Tipos de Recuperação no DB2 for z/OS

O DB2 for z/OS oferece vários métodos de recuperação para restaurar objetos a um estado consistente. A escolha do tipo ideal depende do tipo de falha, da granularidade do objeto afetado, dos tempos de recuperação exigidos (RTO) e da existência de backups válidos e logs disponíveis (RPO).

---

### 1. RECOVER TOCOPY

- Restaura o objeto usando uma cópia registrada (COPY FULL ou MERGECOPY).
- A opção mais direta quando há uma cópia full válida e recente.

#### ✅ Detalhes:
- Aponta para um `COPY` específico registrado na `SYSCOPY`.
- Não exige leitura de log se usado com `RETAIN` ou `REUSE`.
- Pode ser usado para:
  - TABLESPACE
  - INDEX
  - PARTITION
- Pode ser combinada com `REBUILD INDEX` automaticamente.

---

### 2. RECOVER TORBA (Recover to a Log Point)

- Recupera o objeto até uma **log point específica** (RBA ou LRSN).
- Permite retornar a um estado exato anterior a uma falha lógica ou erro humano (ex: delete acidental).

#### ✅ Detalhes:
- Necessário:
  - Cópia válida (COPY FULL ou MERGECOPY)
  - Arquivos de log disponíveis até o ponto desejado.
- `TORBA` é utilizado em sistemas com RBA (modo básico).
- `TOLRSN` é utilizado em sistemas com LRSN (modo data sharing ou extended logging).

---

### 3. RECOVER TOLOGPOINT/TOLRSN

- Similar ao TORBA, mas usando o logpoint hexadecimal.
- Geralmente usado em ambientes com Data Sharing ou sistemas que utilizam LRSN.

---

### 4. RECOVER TOCOPY/TOLOGPOINT + TORBA em conjunto

- É possível restringir a aplicação de log até um limite (`TORBA`) mesmo quando se parte de uma cópia (`TOCOPY`).
- Estratégia comum para **recuperações controladas**, onde se quer restaurar até um ponto anterior a uma falha específica.

---

### 5. RECOVER TOCURRENT

- Aplica toda a sequência de logs desde a cópia mais recente até o momento atual.
- Utilizado para recuperar objetos após falhas físicas, como *media failure*.

#### ✅ Vantagens:
- Automatiza todo o processo.
- Requer apenas a cópia e os logs disponíveis no sistema.

---

### 6. RECOVER TABLESPACESET

- Utilizado para recuperar **múltiplos tablespaces logicamente relacionados**, como:
  - Tabela base e seus índices
  - Tabelas com LOBs e XML associados

#### ✅ Benefícios:
- Garante consistência lógica entre objetos interdependentes.
- Pode ser usado com TORBA, TOCOPY, TOLOGPOINT, etc.

---

### 7. RECOVER POSTPONED

- Aplica operações de recuperação que foram adiadas anteriormente (ex: falhas no LOG APPLY).
- Utilizado para automatizar *pendências de recuperação* identificadas em falhas anteriores.

---

### 8. RECOVER BACKOUT (com suporte ao UNDO)

- Desfaz alterações de transações específicas, quando o suporte de UNDO via log estiver habilitado.
- Aplica-se especialmente a ambientes com suporte a `System Time` (temporal tables).

---

### 9. Manual com DSN1COPY

- Restauração física usando o utilitário DSN1COPY.
- Bit a bit de datasets VSAM.
- **Risco elevado**, pois não atualiza catálogo nem SYSCOPY.
- Exige expertise em estruturas internas e mapeamento de OBIDs.

---

### 🔄 Recuperação de Índices

- O DB2 permite usar `RECOVER INDEX` ou usar `REBUILD INDEX`.
- `REBUILD INDEX` pode ser automático após um RECOVER de TABLESPACE com `INDEXES ALL`.

---

### ⚠️ Considerações Importantes

- A existência de cópias e logs válidos é essencial.
- `SYSCOPY` e `SYSIBM.SYSLGRNX` são usados para identificar as entradas de cópia e logs.
- Utilitários como `REPORT RECOVERY` e `REPORT TABLESPACESET` ajudam a identificar pontos recuperáveis.
- QUIESCE pode ser utilizado para facilitar identificações de consistência e logpoints seguros.

---

## 📦 DETALHAMENTO - Gerenciamento de Backups no DB2 for z/OS

O gerenciamento de backups no DB2 for z/OS não se limita a armazenar cópias — ele envolve todo um ecossistema de controle, retenção, validação e suporte à recuperação de dados com zero perda, dentro dos requisitos de RPO/RTO. A seguir, o detalhamento dos pontos mais relevantes, com foco nas necessidades reais de quem administra ambientes produtivos.

---

### 🎯 Objetivos e Requisitos

- **Garantia de recuperação** segura e íntegra em caso de falha física (disco, memória) ou lógica (erro humano, corrupção).
- Atender a **políticas de retenção**, **auditoria**, **compliance**, e **regulatórias (ex: LGPD, SOX)**.
- **Otimizar espaço em disco e fitas** de backup.
- Minimizar o tempo de recuperação (RTO) e a perda de dados (RPO).
- Permitir **auditoria de cópias** e controle detalhado das operações.

---

### 🛠️ Componentes Fundamentais

#### ✅ 1. `SYSCOPY` (Tabela de Catálogo)

- **Finalidade**: Registra todas as cópias realizadas por utilitários `COPY`, `MERGECOPY`, `LOAD LOG NO`, `REORG LOG NO`, entre outros.
- **Campos importantes**:
  - `DSNAME`: Nome do dataset com a cópia.
  - `ICTYPE`: Tipo de cópia (`F`, `I`, `R`, `S`, `L`, `P`, `B`, `C`...).
  - `TIMESTAMP`: Data/hora da cópia.
  - `DBNAME`, `TSNAME`: Identificação do objeto.
  - `STYPE`: Tipo de espaço (`T` - tablespace, `I` - índice).
- **Uso prático**:
  - Verificação de backups disponíveis para `RECOVER`.
  - Determinação de necessidade de `MERGECOPY` ou `MODIFY`.
  - Base para relatórios (`REPORT RECOVERY`).

#### ✅ 2. `SYSLGRNX` (System Log Range)

- Mapeia a **faixa de logs** (RBA ou LRSN) afetando cada espaço de banco de dados.
- Utilizado pelo `RECOVER` para aplicar logs após a cópia.
- Atualizado por atividades de escrita e checkpoints.

#### ✅ 3. Arquivos de Log (Active e Archive)

- **Obrigatórios para aplicar alterações posteriores ao COPY full.**
- São referenciados automaticamente durante o `RECOVER`.
- Devem ser **mantidos em conformidade com a retenção dos backups**.

---

### 🔄 Estratégias de Backup (para cada tipo de dado)

#### 📌 COPY Full

- Cópia completa e consistente de um tablespace ou índice.
- Recomendado após:
  - `LOAD REPLACE`
  - `REORG`
  - `ALTER TABLE` com impacto estrutural
- Armazenado em disco ou fita.

#### 📌 COPY Incremental

- Copia apenas páginas alteradas desde o último COPY (full ou incremental).
- Exige que o objeto esteja definido com `COPY YES`.
- Reduz custo e tempo de cópia, mas exige mais trabalho durante o `RECOVER`.

#### 📌 COPY SHRLEVEL CHANGE

- Permite copiar dados com a tabela online.
- Pode usar logs para aplicar alterações ocorridas durante a cópia.
- Ideal para ambientes 24x7.

#### 📌 MERGECOPY

- Consolida COPY full e incrementais em um novo dataset full.
- Mantém catálogo (`SYSCOPY`) atualizado.
- Recomendado para performance e limpeza de catálogo.

---

### 🧹 Retenção e Eliminação

#### 🛠️ Utilitário: `MODIFY RECOVERY`

- Elimina entradas antigas de `SYSCOPY` e datasets físicos.
- Suporta:
  - `AGE`
  - `DATE`
  - `GDGLIMIT`
  - `DELETE YES`
- Deve ser **automatizado com critérios bem definidos**, com logs válidos ainda disponíveis para recuperação se necessário.

#### 🧠 Boas práticas:

- **Não eliminar** cópias necessárias para uma recuperação válida.
- **Verificar logs arquivados** antes de remover cópias.
- **Automatizar relatórios** (`REPORT RECOVERY`) para avaliar se cópias ainda são úteis.

---

### 📋 Monitoramento e Auditoria

#### ✅ Utilitário: `REPORT RECOVERY`

- Informa se há um caminho válido de recuperação completo.
- Avalia se `COPY`, `MERGECOPY` e logs ainda garantem a recuperação.
- Suporte para filtros por:
  - DBNAME, TSNAME
  - Período
  - Status

---

### 💡 Dicas Essenciais de DBA

| Situação                                          | Ação Recomendada                                      |
|--------------------------------------------------|--------------------------------------------------------|
| Após `LOAD LOG NO`                               | Realizar `COPY FULL` imediatamente                     |
| Após `REORG LOG NO`                              | Verificar `SYSCOPY` e forçar nova cópia se necessário  |
| LOGs não disponíveis                              | Recuperação até o último COPY possível                 |
| Compressão de dados                              | Utilizar `COPY SHRLEVEL CHANGE` para compatibilidade   |
| Recuperação de testes                             | Utilizar `DSN1COPY` para restauração em ambientes dev  |

---
---

### 📋 Plano Automatizado de Retenção de Backups

A seguir, um modelo completo e profissional de estratégia automatizada de retenção de backups, incluindo:

- Política de retenção (com base em idade e número de cópias)
- Execução automatizada com `MODIFY RECOVERY`
- Auditoria semanal com `REPORT RECOVERY`
- Organização e controle de logs e datasets
- Templates de JCL para agendamento no JES2/3 ou scheduler corporativo (Control-M, OPC, etc.)

---

### 🧠 Política de Retenção Recomendada (Exemplo Prático)

| Tipo de Backup             | Retenção Recomendada     |
|---------------------------|--------------------------|
| COPY FULL                 | 15 dias                  |
| COPY INCREMENTAL          | 7 dias                   |
| MERGECOPY                 | 30 dias (se aplicável)   |
| LOGs Arquivados           | 30 dias (mínimo)         |
| DSN1COPY (ambiente teste) | Retido por ambiente      |

> ⚠️ *A retenção deve respeitar o período mínimo necessário para recuperação segura (RPO) e ser ajustada conforme o SLA da empresa.*

---

### 🛠️ Utilitário: MODIFY RECOVERY

#### Exemplo 1: Excluir cópias com mais de 15 dias

```jcl
//MODRECOV JOB (ACCT),'MODIFY COPY',CLASS=A,MSGCLASS=X
//STEP1 EXEC DSNUPROC,SYSTEM=DB2P,UID='MODRCV01',UTPROC=''
//SYSIN    DD *
  MODIFY RECOVERY TABLESPACE DBX.TSCLIENT
     DELETE YES
     AGE 15
/*
//SYSPRINT DD SYSOUT=*
//SYSUDUMP DD SYSOUT=*
```

#### Exemplo 2: Excluir entradas baseadas em data (backup feito antes de 2024-12-01)

```jcl
//MODRECOV JOB (ACCT),'MODIFY COPY',CLASS=A,MSGCLASS=X
//STEP1 EXEC DSNUPROC,SYSTEM=DB2P,UID='MODRCV02',UTPROC=''
//SYSIN    DD *
  MODIFY RECOVERY TABLESPACE DBX.TSCLIENT
     DELETE YES
     DATE 2024-12-01
/*
//SYSPRINT DD SYSOUT=*
```

---

### 📑 Utilitário: REPORT RECOVERY (Auditoria Semanal)

```jcl
//REPRCOV  JOB (ACCT),'AUDITORIA BACKUP',CLASS=A,MSGCLASS=X
//STEP1 EXEC DSNUPROC,SYSTEM=DB2P,UID='REPRCV01',UTPROC=''
//SYSIN    DD *
  REPORT RECOVERY
    TABLESPACE DBX.*
    LIMIT 30
/*
//SYSPRINT DD SYSOUT=*
```

> 🔎 Gera um relatório dos objetos que **ainda têm caminho de recuperação válido** com base nas cópias disponíveis e nos logs arquivados.

---

### 📊 Relatório de SYSCOPY (Query de Apoio)

```sql
SELECT DBNAME, TSNAME, TIMESTAMP, ICTYPE, DSNAME
FROM SYSIBM.SYSCOPY
WHERE DBNAME = 'DBX'
  AND TSNAME = 'TSCLIENT'
ORDER BY TIMESTAMP DESC;
```

> Use como apoio para verificação manual ou construção de dashboards internos.

---

### 🧰 Outras Recomendações

- 🚨 Sempre rodar `REPORT RECOVERY` antes de `MODIFY RECOVERY`.
- 🎯 Após `REORG LOG NO`, garanta uma nova `COPY`.
- 📁 Verifique se os logs arquivados estão disponíveis (não eliminados precocemente).
- 📦 Utilize **GDG** para backups físicos: `COPY001.GDG`, `COPY002.GDG`, etc.
- 🔄 Use `MERGECOPY` periodicamente para consolidar backups incrementais.


---

## ✅ Conclusão

A automação da retenção de backups e a limpeza de `SYSCOPY` são fundamentais para garantir:

- Catálogo limpo e eficiente
- Redução no uso de disco e fitas
- Cumprimento de SLAs de recuperação
- Maior confiança nas operações de `RECOVER`

> Um DBA bem preparado não só faz o backup — ele **garante que a recuperação aconteça de forma rápida e segura.**


---

### 📚 Referência IBM

```text
https://www.ibm.com/docs/en/db2-for-zos/latest?topic=utilities-copy-utility
https://www.ibm.com/docs/en/db2-for-zos/latest?topic=utilities-modify-recovery-utility
https://www.ibm.com/docs/en/db2-for-zos/latest?topic=utilities-report-recovery-utility
```

---

