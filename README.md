# Manual Básico DB2 for z/OS

🗂️ **Manual de Boas Práticas para DBAs de Desenvolvimento – DB2 for z/OS**

Este repositório é um guia técnico e prático para profissionais que atuam como Administradores de Banco de Dados no ambiente **IBM DB2 for z/OS**. Estruturado em módulos, ele abrange do básico ao avançado, com foco em produtividade, organização e performance no dia a dia de um DBA.

---

## 📚 Índice de Tópicos

| Módulo | Descrição |
|--------|-----------|
| [📁 Análise de Abend](abend/abend.md) | Análise de Abend em programas DB2 |
| [📁 TSO](tso/tso.md) | Comandos básicos do TSO |
| [📁 Dimensionamento Tabelas DB2](dimensionamento/dimensionamento.md) | Dimensionamento do tamanho das Tabelas no DB2 for z/OS |
| [📁 Dimensionamento Tabelas DB2 - 1](dimensionamento-compl/dimensionamento-compl.md) | Dimensionamento do tamanho das Tabelas no DB2 for z/OS |
| [📁 Dimensionamento Tabelas DB2 - 2](dimensionamento-compl2/dimensionamento-2.md) | Dimensionamento do tamanho das Tabelas no DB2 for z/OS |
| [📁 Catálogo DB2](catalogo/catalogo-db2.md) | Estrutura, tabelas e queries úteis do catálogo do sistema |
| [📁 DEFAULT](valores-default/valores-default.md) | Valores DEFAULT em colunas |
| [📁 Tablespaces](tablespaces/tablespaces.md) | Organização física dos dados e segmentação |
| [📁 Tables](tables/tables.md) | Organização dentro do tablespace |
| [📁 SEQUENCE](sequence/sequence.md) | SEQUENCE |
| [📁 SEQUENCE - Complementar1](sequence-compl1/sequence-compl1.md) | SEQUENCE Informações Complementares |
| [📁 Check Constraint](check-constraint/check-constraint.md) | CHECK CONSTRAINT |
| [📁 Estatísticas e Otimizador](estatisticas/estatisticas.md) | Uso de RUNSTATS, estatísticas e tuning do otimizador |
| [📁 Escopo Inicial de Performance](escopo-inicial_performance/escopo-inicial-performance.md) | ESCOPO INICIAL – Diagnóstico Performance de Query ambiente produtivo |
| [📁 Diagnóstico com base em RUNSTATS](runstats/runstats.md) | Diagnóstico com foco em RUNSTATS |
| [📁 Diagnóstico com base em Indices](indices/indices.md) | Diagnóstico com foco em Indices |
| [📁 Diagnóstico com base em Access Path](access-path/access-path.md) | Diagnóstico com foco em Access Path |
| [📁 Tuning e Performance](desempenho/tunning-consultas.md) | Diagnóstico e otimização de queries |
| [📁 Performance](performance/performance.md) | Visão Crítica: O que torna uma query eficiente no DB2 for z/OS? |
| [📁 Utilitários DB2](utilitarios/utilities.md) | Comandos como REORG, CHECK, COPY, LOAD etc. |
| [📁 DEFUTIL](defutil/defutil.md) | DEFUTIL - Utility Definition |
| [📁 LOAD](load/load.md) | Processos de LOAD |
| [📁 Tipos de LOAD](tipos/tipos-de-load.md) | Os tipos de LOAD |
| [📁 Backup e Recuperação](backup-recuperacao/backup-recovery.md) | Estratégias de segurança e restore |
| [📁 Syscopy](syscopy/syscopy.md) | Detalhes do catálogo  - seção especial |
| [📁 Queries SQL](sql-avancado/sql-exemplos.md) | Consultas avançadas aplicáveis ao dia a dia |
| [📁 BIND](bind/bind.md) | BIND no DB2 for z/OS |
| [📁 BIND2](bind2/bind2.md) | BIND no DB2 for z/OS |
| [📁 Dicas](dicas/dicas.md) | Dicas aplicáveis ao dia a dia para queries e performance |
| [📁 Comandos](mandos/comandos.md) | Comandos DB2 para visualizar situação de tabelas |
| [📁 JCL](jcl/jcl.md) | JCL ( JOB CONTROL LANGUAGE ) |
| [📁 CONTROL-M ](controlm/controlm.md) | Tabelas do CONTROL-M |
| [📁 STROBE ](strobe/strobe.md) | STROBE - ferramenta de análise de desempenho em ambiente z/OS |
| [📁 IBM OMEGAMON ](omegamon/omegamon.md) | IBM OMEGAMON - suíte de ferramentas de monitoramento para ambientes z/OS |
| [📁 MainView ](mainview/mainview.md) | MainView - suíte de ferramentas de monitoramento para ambientes z/OS |
| [📁 SMF Analyzers ](smf-analyzers/smf-analyzers.md) | SMF Analyzers - suíte de ferramentas de monitoramento para ambientes z/OS |
| [📁 HPU – High Performance Unload](hpu/hpu.md) | Descarregamento rápido de dados com HPU, exemplos e tuning |
| [📁 HPU2 – High Performance Unload](hpu2/hpu2.md) | Informações Gerais HPU |
| [📁 Tabelas Temporais](tabelas-temporais/tabelas-temporais.md) | Tabelas Temporais (Temporal Tables) no DB2 for z/OS |
| [📁 DFSORT](dfsort/dfsort.md) | Manual Dfsort |
| [📁 PowerDesigner](powerdesigner/powerdesigner.md) | Manual PowerDesigner para DBAs |
| [📁 Lista de IA's](ia/ia-para-dba.md) | Lista de IA's para consultas pelos DBA's | 
| [📁 CAIXA - ORDENACAO](caixa-ordenacao/caixa-ordenacao.md) | Esboço para uma proposta de Ordenação | 



---

## 🔍 Objetivo

> Este manual foi criado para ser um **repositório vivo de conhecimento**, organizado por temas, de modo que possa ser consultado, atualizado e ampliado com o tempo por profissionais que atuam com DB2 for z/OS.

---

## 📌 Licença

Este projeto é de uso pessoal/profissional aberto. Utilize, adapte e compartilhe com responsabilidade.

---

## 📌 Créditos e Contato

> Criado por **SILVIA GUIMARÃES** como parte de portfólio de projetos.

Para dúvidas ou sugestões, entre em contato comigo:
- **E-mail:** (sguimaraes1004@gmail.com)
- **Redes Sociais: [LinkedIn](https://www.linkedin.com/in/silvia-maria-guimar%C3%A3es-costa-3a01b423b)**
  
---
