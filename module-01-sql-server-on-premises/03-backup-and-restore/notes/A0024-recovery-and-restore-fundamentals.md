# A0024 – Fundamentos de Recovery e Restore

> **Author:** Rafael Binda  
> **Created:** 2026-04-09  
> **Version:** 2.0  

---

## Descrição

Este material apresenta os conceitos fundamentais de recuperação de dados no SQL Server, incluindo recovery models, funcionamento do processo de restore, operações de REDO e UNDO, opções de execução (RECOVERY, NORECOVERY e STANDBY) e estratégias de recuperação como point-in-time

---

## Hands-on

[Q0020 - Restore with NORECOVERY / RECOVERY](../scripts/Q0020-sql-restore-norecovery-recovery.sql)  
[Q0021 - Restore with STANDBY](../scripts/Q0021-sql-restore-standby.sql)  
[INST-Q0012 - Backup Chain and Restore Sequence Inspection](../../../dba-scripts/SQL-instance-information/INST-Q0012-backup-chain-and-restore-sequence-inspection.sql)  
[INST-Q0013 - Tail Log and Recovery Investigation](../../../dba-scripts/SQL-instance-information/INST-Q0013-tail-log-and-recovery-investigation.sql)  
[INST-Q0014 - Tail Log and Recovery Readiness](../../../dba-scripts/SQL-instance-information/INST-Q0014-tail-log-and-recovery-readiness.sql)

---

## 1 - Introdução ao restore

- Não é necessário criar previamente um banco vazio para executar o restore
- Permissões necessárias:
  - Banco já existente: `sysadmin` ou `db_owner`
  - Criação de novo banco: `sysadmin` ou `db_creator`

---

## 2 - Fases do processo de restore

O processo de restore pode ser dividido em duas fases principais:

#### 1ª fase - Cópia
Consiste na transferência dos dados do backup para os arquivos de dados (MDF/NDF) e log (LDF)

#### 2ª fase - Recovery
Consiste na aplicação das operações de REDO e UNDO utilizando o transaction log, garantindo a consistência do banco de dados

---

## 3 - REDO e UNDO

Durante a fase de recovery, o SQL Server utiliza o transaction log para garantir a consistência do banco de dados através das operações de REDO e UNDO

### REDO (Refazer)

- Consiste em reaplicar todas as transações que já estavam confirmadas (committed) no momento do backup, mas que ainda não tinham sido gravadas fisicamente nos arquivos de dados (MDF/NDF)
- Garante que todas as transações confirmadas sejam aplicadas no banco

Exemplo:  
Uma transação de INSERT já havia recebido COMMIT, porém seus dados ainda não tinham sido persistidos no arquivo MDF  
Durante o recovery, o SQL Server reaplica essa transação para garantir que os dados estejam corretamente no banco

### UNDO (Desfazer)

- Consiste em desfazer (rollback) todas as transações que não estavam confirmadas (uncommitted) no momento do backup
- Garante que nenhuma transação não confirmada permaneça no banco

Exemplo:  
Um UPDATE foi iniciado, mas não recebeu COMMIT antes do backup  
Durante o recovery, o SQL Server desfaz essa transação para evitar inconsistência nos dados

---

## 4 - Opções de restore

### RECOVERY (default)

- Executa as duas fases do restore:
  - Cópia dos dados
  - Recovery (REDO e UNDO)
- Ao final do processo, o banco é colocado online e liberado para uso
- Encerra a sequência de restore, não permitindo a aplicação de backups adicionais (FULL, diferencial ou log)

### NORECOVERY

- Executa apenas a fase de cópia dos dados, sem aplicar o recovery (REDO e UNDO)
- O banco permanece indisponível, com status: `Restoring`
- Permite a aplicação de backups adicionais na sequência (diferencial ou log)
- Deve ser utilizado em todos os passos intermediários do processo de restore

### STANDBY

- Executa as fases de cópia e recovery (REDO e UNDO), porém mantém o banco disponível apenas para leitura
- O banco fica acessível em modo read-only entre os restores
- Permite a aplicação de backups adicionais na sequência (como no NORECOVERY)
- Utiliza um arquivo auxiliar (undo file) para armazenar temporariamente as transações desfeitas (UNDO)
- Esse arquivo é utilizado nos restores seguintes para reaplicar ou descartar corretamente essas transações

---

## 5 - Cenários de exemplo usando RECOVERY e NORECOVERY

### Tempo total para um processo completo de restore

1 FULL NORECOVERY     2 horas  
2 DIFF NORECOVERY     30 minutos  
3 LOG  NORECOVERY     30 minutos  
4 LOG  NORECOVERY     30 minutos  
5 LOG  RECOVERY       30 minutos

### Cenário 1

- Foi utilizado NORECOVERY no último passo (5)
- O banco permaneceu fora do ar (status: Restoring)

#### Solução 1

Reexecutar o último comando utilizando RECOVERY

- O SQL Server identifica que a fase de cópia já foi concluída
- Executa apenas a fase de recovery (REDO e UNDO)

#### Solução 2

Executar o comando:

`RESTORE LOG banco WITH RECOVERY`

- Não é necessário utilizar mídia de backup
- O SQL Server executa somente a fase de recovery

### Cenário 2

- Foi utilizado RECOVERY no passo 3
- Ainda existiam backups adicionais (LOG 4 e LOG 5) para serem restaurados

#### Problema

Ao executar RECOVERY antes do fim da sequência, o SQL Server:

- Finaliza o processo de restore
- Impede a aplicação de novos backups

#### Solução

Será necessário reiniciar todo o processo desde o início:

1 FULL NORECOVERY     2 horas  
2 DIFF NORECOVERY     30 minutos  
3 LOG  NORECOVERY     30 minutos  
4 LOG  NORECOVERY     30 minutos  
5 LOG  RECOVERY       30 minutos

### Resumo

- NORECOVERY → Utilizar enquanto ainda há backups a serem aplicados
- RECOVERY → Utilizar apenas no último passo

---

## 6 - Recovery Models

Define como o transaction log será gerenciado e quais estratégias de recuperação estarão disponíveis no SQL Server

O recovery model determina:

- Como o transaction log registra as operações
- Quando o log pode ser truncado
- Se é possível realizar backup de log
- O nível de granularidade na recuperação dos dados (ex: point-in-time)

Em outras palavras, o recovery model impacta diretamente:

- A capacidade de recuperação do banco
- O risco de perda de dados
- A estratégia de backup adotada

### RECOVERY MODEL SIMPLE

- No recovery model SIMPLE, o SQL Server realiza o truncamento automático do transaction log a cada CHECKPOINT
- Isso significa que o log não mantém histórico completo das transações, apenas o necessário para garantir a consistência do banco

#### Características

- Truncamento automático do transaction log
- Não permite backup de log
- Estrutura de log mais simples e com menor crescimento
- Menor complexidade de gerenciamento

#### Funcionamento

- O SQL Server reutiliza o espaço do transaction log automaticamente
- Após um CHECKPOINT, a parte inativa do log é liberada
- Não há retenção contínua das transações

#### Impacto na recuperação

- Não permite Point-in-Time Restore
- A recuperação é limitada ao último backup FULL ou diferencial
- Todas as transações após o último backup serão perdidas em caso de falha

#### Exemplo prático

Cenário:

- Backup FULL às 00:00
- Falha ocorre às 10:00

Resultado:

- Todos os dados entre 00:00 e 10:00 serão perdidos

#### Quando utilizar

- Ambientes de desenvolvimento
- Ambientes de teste
- Sistemas onde a perda de dados recente é aceitável
- Bancos com baixa criticidade

#### Vantagens

- Menor uso de espaço em disco
- Menor necessidade de gerenciamento de backups de log
- Configuração simples

#### Desvantagens

- Alto risco de perda de dados
- Sem recuperação ponto no tempo
- Não atende ambientes críticos

#### Observações importantes

- Alterar de SIMPLE para FULL exige um novo backup FULL para iniciar a cadeia de log
- Mesmo no modelo SIMPLE, backups FULL e diferencial continuam sendo necessários

#### Resumo

O modelo SIMPLE prioriza simplicidade e baixo gerenciamento, porém com limitação significativa na recuperação de dados, sendo indicado apenas para ambientes onde a perda de dados é aceitável

### RECOVERY MODEL FULL

- No recovery model FULL, o SQL Server mantém o histórico completo das transações no transaction log até que seja realizado um backup de log
- Isso permite a recuperação do banco de dados até qualquer ponto específico no tempo (point-in-time)

#### Características

- Mantém histórico completo do transaction log
- Permite backup de log
- Suporte a Point-in-Time Restore
- Controle total sobre recuperação

#### Funcionamento

- Todas as transações são registradas no transaction log
- O log não é truncado automaticamente
- O truncamento ocorre somente após backup de log
- Enquanto não houver backup de log, o arquivo de log continuará crescendo

#### Impacto na recuperação

- Permite recuperação até qualquer momento específico
- Possibilita restaurar o banco imediatamente antes de uma falha
- Minimiza perda de dados

#### Exemplo prático

Cenário:

- Backup FULL às 00:00
- Backups de log a cada 15 minutos
- Falha ocorre às 10:07

Resultado:

- É possível restaurar o banco até 10:06:59
- Perda de dados praticamente zero

#### Quando utilizar

- Ambientes de produção
- Sistemas críticos
- Aplicações que não toleram perda de dados
- Cenários que exigem recuperação granular

#### Vantagens

- Alta capacidade de recuperação
- Minimiza perda de dados
- Permite recuperação ponto no tempo
- Maior controle sobre o ambiente

#### Desvantagens

- Maior uso de espaço em disco (log)
- Necessidade de gerenciamento de backups de log
- Maior complexidade operacional

#### Observações importantes

- É obrigatório realizar backups de log regularmente
- Se não houver backup de log, o arquivo de log crescerá continuamente
- A cadeia de backups deve ser mantida íntegra (LSN)
- Alterar de SIMPLE para FULL exige um novo backup FULL para iniciar a cadeia

#### Resumo

O modelo FULL oferece o maior nível de proteção de dados, sendo o padrão recomendado para ambientes críticos que exigem recuperação precisa e mínima perda de dados

### RECOVERY MODEL BULK_LOGGED

- Modelo intermediário entre SIMPLE e FULL

#### Quando utilizar

Deve ser utilizado temporariamente em cenários específicos como:
- Carga massiva de dados (ETL)
- Importação de grandes volumes
- Rebuild de índices grandes

#### Estratégia recomendada
1.  Alterar para BULK_LOGGED
2.  Executar operação pesada
3.  Realizar backup de log
4.  Voltar para FULL

#### Minimal Logging

No modelo BULK_LOGGED, algumas operações utilizam logging mínimo

Isso significa:
- Nem todas as alterações são registradas detalhadamente no log
- Apenas informações essenciais são armazenadas

#### Impacto

- Redução significativa do uso de log
- Melhor performance em operações massivas

#### Consequência

- Não é possível restaurar para um ponto exato dentro dessas operações
- O restore só pode ocorrer até o final do backup de log

### Impacto dos Recovery Models na recuperação

O recovery model define diretamente quais tipos de restore são possíveis

| Recovery Model | Backup de Log | Point-in-Time | Risco de perda |
|---------------|--------------|--------------|----------------|
| SIMPLE        | Não          | Não          | Alto           |
| FULL          | Sim          | Sim          | Baixo          |
| BULK_LOGGED   | Sim          | Parcial      | Médio          |

- SIMPLE: perda de dados desde o último backup
- FULL: recuperação até ponto exato no tempo
- BULK_LOGGED: recuperação limitada durante operações bulk

### Troca de recovery model

#### SIMPLE → FULL

- Exige um novo backup FULL para iniciar a cadeia de log
- Antes disso, não é possível fazer backup de log

#### FULL → SIMPLE

- Quebra a cadeia de backup de log
- Todos os backups de log anteriores deixam de ser úteis

#### FULL → BULK_LOGGED → FULL

- Não quebra a cadeia
- Porém afeta granularidade de recuperação durante operações bulk

---

## 7 - Point-in-Time Restore

Permite restaurar o banco de dados para um momento específico no tempo, geralmente utilizado em cenários de erro humano, como exclusões ou atualizações indevidas  
O Point-in-Time Restore permite recuperar o banco com alta precisão, reduzindo perda de dados e sendo essencial em ambientes críticos

### Requisitos

- Recovery model FULL ou BULK_LOGGED
- Backup FULL
- Backups de LOG em sequência

Sem esses elementos, não é possível realizar recuperação ponto no tempo

### Funcionamento

O Point-in-Time Restore utiliza a cadeia de backups de log para avançar o banco até um momento exato

O processo ocorre da seguinte forma:
1. Restaurar o backup FULL com NORECOVERY
2. Restaurar backups diferenciais (se houver) com NORECOVERY
3. Restaurar backups de LOG em sequência
4. No último backup de log, aplicar a cláusula STOPAT

Isso permite interromper a recuperação exatamente no momento desejado

### Exemplo prático

Cenário:
- DELETE acidental às 22:37
- Último backup de log às 22:30
- Próximo backup de log às 22:45

Mesmo sem backup exatamente às 22:37, é possível recuperar:
- Restaurando o log das 22:45
- Utilizando STOPAT = '22:36:59'

### Observações importantes

- O restore sempre segue a ordem cronológica dos backups
- A cláusula STOPAT é aplicada apenas no último backup de log
- Não é possível aplicar STOPAT em backups FULL ou diferencial

### Limitações

- Não funciona no recovery model SIMPLE
- Em BULK_LOGGED, pode haver limitação durante operações bulk
- Depende da integridade completa da cadeia de backups

### Uso comum

- Recuperação de erro humano (DELETE/UPDATE incorreto)
- Reversão de alterações indevidas
- Recuperação de dados específicos sem necessidade de restore completo para último estado

---

## 8 - Observações finais

Backup não garante recuperação  
Somente restore testado garante recuperação

---

## Referências

- [RESTORE Statements (Transact-SQL)](https://learn.microsoft.com/pt-br/sql/t-sql/statements/restore-statements-transact-sql?view=sql-server-ver16)
- [RESTORE Statements - Arguments (Transact-SQL)](https://learn.microsoft.com/pt-br/sql/t-sql/statements/restore-statements-arguments-transact-sql?view=sql-server-ver16)
- [Modelos de recuperação (SQL Server)](https://learn.microsoft.com/pt-br/sql/relational-databases/backup-restore/recovery-models-sql-server?view=sql-server-ver16)
- [Restaurações completas de banco de dados (modelo de recuperação completo)](https://learn.microsoft.com/pt-br/sql/relational-databases/backup-restore/complete-database-restores-full-recovery-model?view=sql-server-ver16)
- [Restaurar um banco de dados do SQL Server até um ponto no tempo](https://learn.microsoft.com/pt-br/sql/relational-databases/backup-restore/restore-a-sql-server-database-to-a-point-in-time-full-recovery-model?view=sql-server-ver16)
