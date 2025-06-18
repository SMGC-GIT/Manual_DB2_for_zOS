## 🚀 Performance de Queries no DB2 for z/OS

## 📚 Índice

- [🔍 O que torna uma query eficiente](#🔍-o-que-torna-uma-query-eficiente)
- [📊 Estatísticas precisas para o Otimizador](#📊-estatísticas-precisas-para-o-otimizador)
- [🏗️ Modelagem e Desenho Físico](#🏗️-modelagem-e-desenho-físico)
- [✍️ Escrita Eficiente do SQL](#✍️-escrita-eficiente-do-sql)
- [🧰 Ferramentas de Diagnóstico e Otimização](#🧰-ferramentas-de-diagnóstico-e-otimização)
- [🧠 Comportamento do Otimizador](#🧠-comportamento-do-otimizador)
- [🔧 Diagnóstico e Tuning](#🔧-diagnóstico-e-tuning)
- [✅ Resumo das Boas Práticas](#✅-resumo-das-boas-práticas)

---

### 🔍 O que torna uma query eficiente

Uma **query com boa performance** no DB2 for z/OS:

- Retorna o resultado correto, no menor tempo possível
- Utiliza eficientemente CPU, I/O e buffer pools
- Escala bem com aumento de volume
- É estável mesmo com mudanças nas estatísticas

---

### 📊 Estatísticas precisas para o Otimizador

O otimizador é cost-based e depende das estatísticas para gerar planos eficientes.

**✅ Boas práticas**:

- Executar `RUNSTATS` regularmente
- Usar `FREQVAL`, `COLGROUP`, `HISTOGRAM` para dados com skew
- Incluir `STATS PROFILE` para reprodutibilidade
- Estatísticas compostas para joins complexos

**⚠️ Riscos**:

- Estatísticas antigas ou incompletas geram planos ruins
- Uso inadequado do `AUTO_STMT_STATS`

🔗 [RUNSTATS - IBM Documentation](https://www.ibm.com/docs/en/db2-for-zos/13?topic=statistics-collecting-data)

---

### 🏗️ Modelagem e Desenho Físico

O layout físico afeta diretamente a eficiência de acesso.

**✅ Boas práticas**:

- Utilizar `UTS` (Universal Tablespaces)
- Índices com `INCLUDE`, compostos e seletivos
- Avaliar uso de particionamento
- Normalização consciente

**⚠️ Riscos**:

- Joins custosos por modelagem fraca
- Índices obsoletos ou em excesso

🔗 [Database Design - IBM Documentation](https://www.ibm.com/docs/en/db2-for-zos/13?topic=guide-database-design-considerations)

---

### ✍️ Escrita Eficiente do SQL

A forma como o SQL é escrito impacta diretamente a geração do plano.

**✅ Boas práticas**:

- Preferir `EXISTS` em vez de `IN` para subqueries
- Usar `JOIN` explícito, evitar `WHERE` implícito
- Evitar funções sobre colunas indexadas
- Utilizar `FETCH FIRST n ROWS ONLY` quando aplicável
- Modularizar usando CTEs (`WITH`)

**⚠️ Riscos**:

- Uso indiscriminado de `DISTINCT`, `UNION`
- Subqueries mal planejadas
- Falta de `OPTIMIZE FOR`

🔗 [SQL Performance Guidelines - IBM](https://www.ibm.com/docs/en/db2-for-zos/13?topic=performance-improving-sql)

---

### 🧰 Ferramentas de Diagnóstico e Otimização

Ferramentas recomendadas pela IBM:

- `EXPLAIN`, `DSN_STATEMNT_TABLE`
- `PLAN_TABLE`, `ACCESSPATH`
- `IBM Data Studio`, `Visual Explain`
- `OMPE` (Omegamon Performance Expert)
- `SYSQUERY`, `SYSQUERYPLAN` (Query Caching)

---

### 🧠 Comportamento do Otimizador

O otimizador do DB2:

- É sensível a `FETCH FIRST`, `ORDER BY`, `GROUP BY`
- Prefere filtros seletivos com índices
- Reage ao uso de estatísticas incompletas
- Pode gerar planos diferentes com variações mínimas

---

### 🔧 Diagnóstico e Tuning

Para análise e tuning avançado:

- Utilizar `EXPLAIN` detalhado com `DSN_QUERYINFO_TABLE`
- Avaliar acessos com `LIST PREFETCH`, `MULTI-INDEX`, `MATCH SCAN`
- Verificar cache e uso de índice em `PLAN_TABLE`
- Acompanhar estatísticas em tempo real via `MONITOR` ou `OMPE`

---

### ✅ Resumo das Boas Práticas

| Área                         | Prática Recomendada                                        |
|------------------------------|-------------------------------------------------------------|
| Estatísticas                 | `RUNSTATS` com colgroups e histogramas                     |
| SQL                          | Uso explícito de JOINs, EXISTS, filtros eficientes         |
| Índices                      | Índices compostos, com INCLUDE, atualizados                |
| Otimizador                   | Validar plano com `EXPLAIN`, `OPTIMIZE FOR`, `FETCH FIRST` |
| Análise de performance       | Usar `Visual Explain`, `OMPE`, `DSN_STATEMNT_TABLE`        |
| Escalabilidade               | Avaliar particionamento e paralelismo                      |

---
