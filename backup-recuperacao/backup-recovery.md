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

## 🔐 Tipos de Backup no DB2 for z/OS

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

### 📚 REFERÊNCIA
```jcl
Documentação Oficial IBM:
https://www.ibm.com/docs/en/db2-for-zos/latest?topic=utilities-copy-utility

```

---



