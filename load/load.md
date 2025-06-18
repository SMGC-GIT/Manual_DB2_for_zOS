# 🚀 LOAD – Boas Práticas, Decisões Técnicas e JCL Avançado

## 📚 Índice

- [🔍 Quando usar o LOAD](#🔍-quando-usar-o-load)
- [💡 Dicas Técnicas de DBA](#💡-dicas-técnicas-de-dba)
- [❓ Questionário Técnico: Escolha do Tipo de LOAD](#❓-questionário-técnico-escolha-do-tipo-de-load)
- [📈 Volumes Ideais por Modo](#📈-volumes-ideais-por-modo)
- [⚙️ Tipos de LOAD e Cenários](#⚙️-tipos-de-load-e-cenários)
- [🛠️ Boas práticas de performance](#🛠️-boas-práticas-de-performance)
- [🚨 Erros comuns e soluções](#🚨-erros-comuns-e-soluções)
- [📝 Exemplos de JCL por tipo](#📝-exemplos-de-jcl-por-tipo)
- [📚 Referências IBM](#📚-referências-ibm)

---

### 🔍 Quando usar o LOAD

O **LOAD** é indicado para:
- Montantes altos de dados (milhares a milhões de linhas)
- Necessidade de velocidade e eficiência
- Recriação total ou parcial da tabela com gathers inline
- Uso de opções como `PRESORTED`, `PARALLEL` e `INLINE STATISTICS`

Não use para updates marginais — prefira `UPDATE`, `INSERT` ou `MERGE`.

---

### 💡 Dicas Técnicas de DBA

1. **ALTER TABELA → REFRESH LOGIC**  
   Após alterar colunas (ex: adicionar coluna, mudar tipo), rodar `DSN1COPY` inline ou `INLINE STATISTICS YES` evita inconsistências no LOAD. :contentReference[oaicite:1]{index=1}  
2. **PRESORTED** apenas se dados estiverem pré-ordenados por clustering index, evitando REORG pós-load. :contentReference[oaicite:2]{index=2}  
3. **COPYDDN/RECOVERYDDN + LOG NO** cria image copy inline e acelera o processo sem gerar logs redundantes. :contentReference[oaicite:3]{index=3}  
4. **FORMAT INTERNAL** recomendado se executar UNLOAD previamente, eliminando overhead de conversão. :contentReference[oaicite:4]{index=4}  
5. **PARALLEL** melhora uso de CPU/I/O — mas use em tabelas distribuídas para evitar contenções. :contentReference[oaicite:5]{index=5}  
6. Utilize **INLINE STATISTICS YES** para evitar post-RUNSTATS e garantir consistência de plano. :contentReference[oaicite:6]{index=6}  
7. Em tabelas com LOB/XML, indique `LOBFILE ddname` para evitar overhead de memória. :contentReference[oaicite:7]{index=7}  

---

### ❓ Questionário Técnico: Escolha do Tipo de LOAD

1. Os dados vão substituir completamente a tabela?  
   - Sim → siga para **REPLACE**  
   - Não → use **INSERT** ou **RESUME**

2. Os dados estão pré-ordenados por clustering index?  
   - Sim → use `PRESORTED YES`  
   - Não → deixe sem PRESORTED

3. Deseja evitar logging e criar image copy?  
   - Sim → inclua `COPYDDN` + `LOG NO`

4. É crítico atualizar estatísticas inline?  
   - Sim → use `INLINE STATISTICS YES`

5. Volume alto (> 5M linhas) e hardware disponível?  
   - Sim → adicione `PARALLEL(0)` ou `PARALLEL(n)`

---

### 📈 Volumes Ideais por Modo

- **Batch simples** (`INSERT`, `REPLACE`): >= 1M linhas  
- **Online SHRLEVEL CHANGE**: 100k–500k linhas, acesso concorrente  
- **PRESORTED + PARALLEL**: ideal para >5M linhas  

---

### ⚙️ Tipos de LOAD e Cenarios

| Tipo         | Cenário de uso                                             |
|--------------|------------------------------------------------------------|
| INSERT       | Inclusão incremental de dados                             |
| REPLACE      | Substituição completa de dados                            |
| RESUME       | Continuação após falha, mantendo checkpoints              |
| TERMINATE    | Finaliza carga em andamento quando preciso abortar        |
| COPYDDN      | Cria image copy inline da tabela após carga               |
| PRESORTED    | Quando dados já vêm ordenados por index de cluster        |
| PARALLEL     | Uso de múltiplas threads para acelerar a carga            |

---

### 🛠️ Boas práticas de performance

- Evite conversões CCSID e tipos durante o LOAD :contentReference[oaicite:8]{index=8}  
- Impeça REORG pós-load com `PRESORTED` quando possível :contentReference[oaicite:9]{index=9}  
- Evite fragmentação com `COPYDDN + LOG NO`  
- Acelere carga com `PRESORTED + PARALLEL` :contentReference[oaicite:10]{index=10}  
- Sempre use `INLINE STATISTICS YES` :contentReference[oaicite:11]{index=11}  
- Ao editar colo XML/LOB, use `LOBFILE` para armazenamento externo

---

### 🚨 Erros comuns e soluções

| Código/Mensagem          | Justificativa                       | Solução                                       |
|--------------------------|-------------------------------------|------------------------------------------------|
| SQL1103N                 | Tipo de dado incompatível           | Ajustar formato ou usar `FILETYPE MODIFIED`   |
| SQL3116W / SQL3185W      | Campo esperado vazio                | Rever `NULL INDICATOR`, incluir colunas       |
| DSNT408I                 | Viola integrity constraints         | Usar `ENFORCE NO`, depois `SET INTEGRITY ALL` |
| IEFC020I                 | Dataset não encontrável             | Revisar DD e altações de acesso               |
| IEC090I                  | Espaço insuficiente                 | Ajustar espaço DSNALLOC                       |

---

### 📝 Exemplos de JCL por tipo

#### INSERT com estatísticas
```jcl
//LOADINS JOB …
…
 LOAD DATA
  FROM INDD(INFILE)
  INSERT INTO TABLESPACE DB2LIB.TS1
  INLINE STATISTICS YES
  PARALLEL
/*
//INFILE DD DSN=MY.INPUT.FILE,DISP=SHR
```

#### REPLACE + COPYDDN
```jcl
//LOADREP JOB …
…
 LOAD DATA
  FROM INDD(INFILE)
  REPLACE
  COPYDDN(LAST_IC)
  INLINE STATISTICS YES
  FORMAT INTERNAL
/*
//INFILE DD DSN=MY.INPUT.INT,DISP=SHR
```

#### RESUME + PRESORTED
```jcl
//LOADRES JOB …
…
 LOAD DATA
  RESUME YES
  INSERT INTO DB2LIB.TS1
  PRESORTED YES
  SHRLEVEL CHANGE
  PARALLEL(0)
/*
```

---

### 📚 Referências IBM

- [Improving LOAD Performance](https://www.ibm.com/docs/en/db2-for-zos/12?topic=load-improving-performance) :contentReference[oaicite:12]{index=12}  
- [LOAD Utility Syntax](https://www.ibm.com/docs/en/db2-for-zos/12?topic=utilities-load) :contentReference[oaicite:13]{index=13}  
- [IBMid’s DB2 Utilities in Practice (Redbooks)](https://www.redbooks.ibm.com/redpapers/pdfs/redp5503.pdf) :contentReference[oaicite:14]{index=14}  
- [INDEXING MODE and PARALLEL details](https://www.ibm.com/docs/en/db2/11.1?topic=commands-load-using-admin-cmd) :contentReference[oaicite:15]{index=15}  

---
