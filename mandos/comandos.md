
# Manual de Boas Práticas para DB2 for z/OS

## Introdução

Este manual fornece um guia prático para DBAs de desenvolvimento que trabalham com DB2 for z/OS. Inclui dicas úteis e relevantes focadas em manter a produtividade do dia a dia e a performance de aplicações e tabelas.

## Comandos Úteis para DB2 for z/OS

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
