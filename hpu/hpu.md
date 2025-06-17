---

## 🚀 HPU – High Performance Unload for Db2 for z/OS

### 🧬 O que é

HPU (High Performance Unload) é uma ferramenta da IBM projetada para realizar **unloads de dados** do Db2 for z/OS **de forma extremamente rápida e eficiente**, sem passar pelo gerenciador de buffer do Db2.

---

### 🎯 Para que serve

- Realizar **exportações massivas** de dados diretamente dos datasets VSAM subjacentes
- Evitar overhead do SQL ou UNLOAD padrão
- Utilizar formatos de saída flexíveis para processamento externo ou carga em outro ambiente

---

### ✅ Vantagens do HPU

| Vantagem                     | Descrição |
|-----------------------------|-----------|
| 🔥 **Alta performance**     | Bypass total do bufferpool e do otimizador SQL |
| 💾 **Baixo uso de CPU**     | Acesso direto ao VSAM reduz uso de recursos |
| 🧩 **Flexibilidade de saída** | Saída em diversos formatos: FIXED, CSV, DELIMITED |
| 📦 **Desacoplamento**       | Pode funcionar com o Db2 desligado |
| 🔐 **Consistência**         | Pode ser executado em modo consistent (a partir de image copy) |

---

### 🛠️ Sintaxe básica – JCL de exemplo

```jcl
//UNLOAD   JOB ...
//STEP1    EXEC PGM=INZUTILB,REGION=0M,
//             PARM='DB2A'
//STEPLIB  DD DSN=IBM.DB2.HPU.LOADLIB,DISP=SHR
//SYSPRINT DD SYSOUT=*
//SYSIN    DD *
  UNLOAD DATA
     FROM TABLE DB2DB01.CLIENTES
     TO DDNAME(SAIDA1)
     FORMAT DELIMITED
     WITH COLUMN NAMES
     FIELD DELIMITER ',' 
     NULL INDICATOR 'NULL'
     SHRLEVEL REFERENCE
/*
//SAIDA1   DD DSN=HPU.UNLOAD.CLIENTES,DISP=(NEW,CATLG,DELETE),
//            SPACE=(CYL,(100,50)),UNIT=SYSDA
```

---

### 🔍 Explicação do exemplo

| Componente | Significado |
|------------|-------------|
| `INZUTILB` | Programa que executa o HPU |
| `STEPLIB` | Biblioteca de carga do HPU (customizada por ambiente) |
| `SYSIN` | Bloco de instruções HPU |
| `SHRLEVEL REFERENCE` | Permite leitura consistente dos dados online |
| `FORMAT DELIMITED` | Define formato da saída (CSV-like) |
| `FIELD DELIMITER ','` | Usa vírgula como delimitador |
| `NULL INDICATOR 'NULL'` | Representa valores nulos no output |

---

### 🔄 Outros formatos de saída

- `FORMAT FIXED` – Saída com largura fixa (padronizada para batch)
- `FORMAT INTERNAL` – Compatível com LOAD ou RELOAD
- `FORMAT DELIMITED` – Para CSV e integração com sistemas externos

---

### 🧠 Dicas avançadas de uso

| Situação | Dica |
|---------|------|
| Exportar apenas algumas colunas | Use `SELECT (col1, col2)` no `FROM TABLE` |
| Melhorar performance | Use `DISABLE LOGGING` e `PARALLEL READS` se permitido |
| Extração por partição | Use `PARTITION n` para unload de parte específica |
| Ambientes com segurança | Pode ser necessário `AUTHID`, `TRUSTED CONTEXT`, etc. |
| Consistência | Prefira `SHRLEVEL REFERENCE` ou `SHRLEVEL CONSISTENT` |

---

### 🛡️ Cuidados importantes

- 🔒 HPU acessa diretamente os datasets VSAM → **Não respeita as regras SQL ou triggers**
- 📌 Não atualiza o catálogo DB2 → Não registra atividades em logs ou SYSCOPY
- 🚫 Não deve ser usado em substituição ao UNLOAD padrão sem análise técnica

---

### 📚 Documentação oficial IBM

- [High Performance Unload (HPU) for Db2 for z/OS – IBM Docs](https://www.ibm.com/docs/en/db2-hpu/5.1)
- [HPU Syntax Reference](https://www.ibm.com/docs/en/db2-hpu/5.1?topic=reference-syntax)

---

### 💻 Exemplo mais avançado – unload com WHERE, delimitadores e várias saídas

```jcl
//UNLDJOB  JOB ...
//STEP1    EXEC PGM=INZUTILB,PARM='DB2A'
//STEPLIB  DD DSN=IBM.DB2.HPU.LOADLIB,DISP=SHR
//SYSIN    DD *
  UNLOAD DATA
     FROM TABLE DB2DB01.CLIENTES
     SELECT (ID_CLIENTE, NOME, CPF)
     WHERE ESTADO = 'SP'
     TO DDNAME(OUT1)
     FORMAT DELIMITED
     FIELD DELIMITER '|'
     RECORD DELIMITER NEWLINE
     NULL INDICATOR ''
     SHRLEVEL REFERENCE
/*
//OUT1     DD DSN=HPU.SAIDA.CLIENTES.SP,
//            DISP=(NEW,CATLG,DELETE),
//            SPACE=(CYL,(100,50)),UNIT=SYSDA
```

---

### ⚙️ HPU e Performance

| Boa prática | Benefício |
|-------------|-----------|
| Usar `PARALLEL READS` | Melhora performance em tabelas grandes |
| Dividir por partição | Distribui carga e acelera execução |
| Selecionar colunas específicas | Reduz I/O e tamanho de output |
| Evitar WHERE complexas | Elas são aplicadas após a leitura do VSAM (custo adicional) |

---

### 🧰 Outros recursos úteis

- `TRACE` – Gera saída de debug para analisar performance
- `LOGGING` – Controla o que será registrado
- `LOAD COMPATIBLE` – Garante compatibilidade com formato de carga para uso posterior no `LOAD`

---

### 📝 Dica prática para DBAs

> Sempre valide os unloads gerados, especialmente ao utilizar `FORMAT DELIMITED` com campos que podem conter delimitadores internos (ex: nomes com vírgulas).  
> Considere também gerar um arquivo de layout para posterior importação em sistemas externos.

---

### 🧩 Limitações
- Não substitui o UNLOAD com SQL onde há necessidade de **JOINs ou lógica de transformação**.
- Respeita somente os dados físicos do tablespace — **não interpreta views ou security policies**.
- Precisa de permissões específicas para acessar os datasets do DB2.

---

### 📚 Referência IBM
[High Performance Unload for Db2 for z/OS – IBM Documentation](https://www.ibm.com/docs/en/db2-hpu/latest)

---

---

## 🧯 Erros Comuns no HPU (High Performance Unload) e Como Resolver

| Código / Mensagem     | Descrição                                                       | Causa Provável                                       | Como Resolver                                                                 |
|------------------------|------------------------------------------------------------------|------------------------------------------------------|--------------------------------------------------------------------------------|
| **DSNU050I**           | Tabela não encontrada ou nome incorreto                         | Nome do objeto especificado incorretamente           | Verifique o nome da tabela, tablespace ou view usado na declaração `TABLE`.   |
| **DSNUGUTC - RC=08**   | Terminação por erro de execução                                 | Parâmetros inconsistentes ou erros de sintaxe        | Revise a sintaxe do SYSIN e os parâmetros obrigatórios.                       |
| **INSUFF_PRIVILEGES**  | Falta de autorização para acessar o objeto                      | Usuário não possui permissão para acessar a tabela   | Garanta as permissões necessárias com `GRANT SELECT` ou acesso `SYSADM`.      |
| **FILE NOT FOUND**     | Arquivo especificado em DD não localizado                       | Nome do dataset incorreto ou não alocado             | Verifique o nome do dataset no JCL e se ele está disponível.                  |
| **RC=12 / RC=16**      | Retorno indicando falha grave no unload                         | Sintaxe incorreta, dataset cheio ou erro lógico      | Refaça a validação da sintaxe e verifique espaço em disco.                   |
| **HPUUSAGE001E**       | O HPU foi executado fora da política de licenciamento           | Licença inválida ou ambiente fora do escopo          | Verifique a validade da licença do produto HPU com o administrador do sistema.|
| **DSNU056I**           | Unload falhou devido a erro de I/O em arquivo temporário        | Problemas de alocação de espaço                      | Verifique parâmetros de alocação de espaço nos DDs temporários.               |
| **HPUOPT001W**         | Parâmetro não reconhecido ou obsoleto                           | Uso de parâmetros descontinuados                     | Revise o manual e substitua os parâmetros por equivalentes atualizados.       |
| **SQLCODE -904**       | Recurso indisponível                                            | Tablespace em `RESTRICT` ou `CHECK PENDING`          | Verifique com `-DISPLAY DATABASE` e resolva com `REPAIR` ou `CHECK DATA`.     |
| **ABEND S0C4 / S0C7**  | Erro de proteção de armazenamento / erro numérico               | Dados inconsistentes ou instrução inválida           | Valide os dados de origem e o mapeamento nos `FIELD` / `LAYOUT`.              |
| **HPUXMSG002I**        | HPU não conseguiu gerar estatísticas de performance             | Faltam parâmetros ou buffers insuficientes           | Habilite corretamente os parâmetros de performance e buffers no `SYSIN`.      |

---

### 🧪 Exemplo de Correção – Falta de Permissão

```sql
-- Erro:
-- INSUFF_PRIVILEGES: usuário sem permissão para acessar a tabela CLIENTES

-- Correção (executar com ID autorizado):
GRANT SELECT ON DB2DB01.CLIENTES TO USER DBA123;
```

---

### 📚 Referências Oficiais IBM

- [IBM Db2 High Performance Unload Reference](https://www.ibm.com/docs/en/db2-hpu)
- [IBM Db2 Utilities Messages and Codes](https://www.ibm.com/docs/en/db2-for-zos/13?topic=messages-db2-utilities)
- [IBM - SQL Error Codes](https://www.ibm.com/docs/en/db2-for-zos/13?topic=sql-sqlcode-sqlstate)

---


