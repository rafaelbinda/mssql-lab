# A0033 – Recovering the msdb Database

> **Author:** Rafael Binda  
> **Created:** 2026-04-27  
> **Version:** 1.0  

---

## Descrição

Este documento apresenta os principais conceitos relacionados à recuperação do banco de sistema `msdb` no SQL Server

O banco `msdb` armazena metadados operacionais importantes da instância, como jobs do SQL Server Agent, histórico de backups, histórico de restores, operadores, alertas, planos de manutenção, Database Mail e outras informações usadas por recursos administrativos

A recuperação da `msdb` exige atenção porque diversos serviços e funcionalidades do SQL Server dependem dela

---

## 1 - O que é o banco msdb

O banco `msdb` é um banco de sistema usado pelo SQL Server para armazenar informações operacionais da instância  
Ele não é o catálogo principal da instância, como a `master`, mas é essencial para diversas funcionalidades administrativas  

Exemplos de informações armazenadas na `msdb`:  
- Jobs do SQL Server Agent
- Schedules
- Operadores
- Alertas
- Histórico de execução de jobs
- Histórico de backups
- Histórico de restores
- Planos de manutenção
- Database Mail
- Log shipping
- Políticas
- SSIS packages armazenados no SQL Server, quando aplicável

---

## 2 - Por que a msdb é importante

A `msdb` concentra muitos metadados usados no dia a dia operacional do DBA  
Se ela for perdida ou corrompida, vários recursos administrativos podem ser afetados  

Exemplos de impactos:  
- Jobs podem desaparecer
- Históricos de backup podem ser perdidos
- Históricos de restore podem ser perdidos
- Alertas podem deixar de existir
- Operadores podem precisar ser recriados
- Planos de manutenção podem ser perdidos
- Configurações de Database Mail podem ser afetadas
- Rotinas automatizadas podem parar de funcionar

Mesmo que os bancos de usuário continuem existindo, a perda da `msdb` pode comprometer a operação e a administração da instância  

---

## 3 - Diferença entre master e msdb

A `master` é essencial para a inicialização e identificação da instância  
A `msdb` é essencial para funcionalidades operacionais e administrativas  
Comparação conceitual:

```text
master = metadados estruturais da instância
msdb   = metadados operacionais da instância
```

Se a `master` estiver corrompida, a instância pode nem inicializar  
Se a `msdb` estiver corrompida, a instância pode até inicializar, mas recursos como SQL Server Agent, jobs, histórico e Database Mail podem falhar  

---

## 4 - Recovery model da msdb

Por padrão, a `msdb` utiliza recovery model `SIMPLE`  
Isso significa que normalmente não existe backup de log da `msdb`  

A estratégia de backup da `msdb` costuma se basear em:  
- Backup full
- Backup diferencial, quando necessário

Exemplo de consulta:  
```sql
SELECT
    name,
    recovery_model_desc
FROM sys.databases
WHERE name = 'msdb';
GO
```

Resultado esperado:  
```text
msdb    SIMPLE
```

Como a `msdb` normalmente está em recovery model `SIMPLE`, o restore tende a ser feito até o ponto do último backup full ou differential disponível  

---

## 5 - Backup da msdb

A `msdb` permite backup full e backup diferencial  
Isso é importante porque, em alguns ambientes, a `msdb` pode crescer bastante  

Exemplo de backup full:  
```sql
BACKUP DATABASE msdb
TO DISK = 'C:\Backups\msdb_FULL.bak'
WITH CHECKSUM, COMPRESSION, STATS = 10;
GO
```

Exemplo de backup diferencial:  
```sql
BACKUP DATABASE msdb
TO DISK = 'C:\Backups\msdb_DIFF.bak'
WITH DIFFERENTIAL, CHECKSUM, COMPRESSION, STATS = 10;
GO
```

O backup da `msdb` deve fazer parte da estratégia de backup dos bancos de sistema  

---

## 6 - Quando fazer backup da msdb

O backup da `msdb` deve ser feito regularmente  
Também é recomendado fazer backup após alterações operacionais importantes  

Exemplos de alterações que justificam novo backup da `msdb`:  
- Criação de job
- Alteração de job
- Criação de schedule
- Alteração de schedule
- Criação de operador
- Criação de alerta
- Alteração em Database Mail
- Criação ou alteração de plano de manutenção
- Configuração de log shipping
- Alterações relevantes no SQL Server Agent

A frequência depende da criticidade das informações armazenadas na `msdb`  

---

## 7 - Crescimento da msdb

A `msdb` pode crescer bastante dependendo dos recursos utilizados  

Exemplo de cenário:  
```text
Cliente envia muitos e-mails com PDF em anexo
Esses e-mails ficam armazenados em tabelas de sistema relacionadas ao Database Mail
Por questões de auditoria, essas informações precisam ser mantidas por vários meses
O histórico de envio e os anexos fazem a msdb crescer de forma significativa
```

Outros fatores que podem causar crescimento da `msdb`:  
- Histórico de backups muito grande
- Histórico de restores muito grande
- Histórico de jobs acumulado por muito tempo
- Database Mail com muitos anexos
- Planos de manutenção com histórico extenso
- Log shipping
- Políticas e auditorias operacionais

Em ambientes onde a `msdb` cresce bastante, pode fazer sentido usar backup full periódico e backup diferencial nos demais dias  

---

## 8 - Exemplo de estratégia de backup

Exemplo conceitual para uma `msdb` com crescimento relevante:  
```text
Domingo     = backup full da msdb
Segunda     = backup diferencial da msdb
Terça       = backup diferencial da msdb
Quarta      = backup diferencial da msdb
Quinta      = backup diferencial da msdb
Sexta       = backup diferencial da msdb
Sábado      = backup diferencial da msdb
```

Essa estratégia reduz o volume necessário em cada backup diário, mantendo uma cadeia simples de recuperação baseada em full + diferencial  

---

## 9 - O que pode causar necessidade de restore da msdb

A restauração da `msdb` pode ser necessária em cenários como:  
- Corrupção da `msdb`
- Perda dos arquivos físicos da `msdb`
- Falha em atualização ou manutenção
- Exclusão acidental de jobs
- Exclusão acidental de operadores
- Exclusão acidental de planos de manutenção
- Problemas em tabelas internas da `msdb`
- Recuperação após rebuild dos bancos de sistema
- Migração ou reconstrução de instância

Antes de restaurar a `msdb`, é importante entender se o problema é estrutural ou se pode ser corrigido pontualmente  

---

## 10 - Cuidados antes de restaurar a msdb

Antes de restaurar a `msdb`, é importante validar:  
- Existe backup full válido da `msdb`
- Existe backup diferencial válido, quando aplicável
- O backup pertence à instância correta
- A versão é compatível
- O caminho do backup está acessível
- A conta de serviço do SQL Server tem permissão no arquivo
- O SQL Server Agent está parado
- Não existem conexões usando a `msdb`
- Existe plano de validação após o restore

A `msdb` é usada por vários componentes internos  
Por isso, o restore deve ser feito com cuidado e preferencialmente em janela controlada  

---

## 11 - Por que parar o SQL Server Agent

Para restaurar a `msdb`, ninguém deve estar usando o banco  
O SQL Server Agent normalmente mantém conexões com a `msdb`  
Por isso, antes do restore, o SQL Server Agent deve ser parado  

O SQL Server Agent utiliza a `msdb` para:
- Ler jobs
- Executar schedules
- Registrar histórico de jobs
- Ler operadores
- Ler alertas
- Controlar planos de manutenção
- Registrar informações operacionais

Se o SQL Server Agent estiver ativo, ele pode bloquear o restore da `msdb`  

---

## 12 - Identificando conexões na msdb

Antes de restaurar a `msdb`, é possível verificar conexões ativas  

Exemplo:
```sql
SELECT
    s.session_id,
    s.login_name,
    s.host_name,
    s.program_name,
    s.status,
    DB_NAME(r.database_id) AS database_name,
    r.command
FROM sys.dm_exec_sessions AS s
LEFT JOIN sys.dm_exec_requests AS r
    ON s.session_id = r.session_id
WHERE DB_NAME(r.database_id) = 'msdb'
    OR s.database_id = DB_ID('msdb')
ORDER BY s.session_id;
GO
```

Se existirem conexões ativas, será necessário encerrá-las ou colocar o banco em modo apropriado para restore  

---

## 13 - Colocando a msdb em SINGLE_USER

Para facilitar o restore, a `msdb` pode ser colocada em modo `SINGLE_USER` com rollback imediato  

Exemplo:
```sql
ALTER DATABASE msdb SET SINGLE_USER WITH ROLLBACK IMMEDIATE;
GO
```

Esse comando encerra conexões existentes e desfaz transações abertas  
Deve ser usado com cuidado em ambiente de produção  
Após o restore, a `msdb` deve retornar para `MULTI_USER`  

---

## 14 - Restore full da msdb

Exemplo conceitual de restore full da `msdb`:  
```sql
RESTORE DATABASE msdb
FROM DISK = 'C:\Backups\msdb_FULL.bak'
WITH REPLACE, RECOVERY;
GO
```

A opção `WITH REPLACE` permite substituir a `msdb` atual pelo conteúdo do backup  
A opção `RECOVERY` finaliza o restore e deixa o banco disponível  
Esse exemplo considera um restore somente do backup full  

---

## 15 - Restore full + diferencial da msdb

Quando existe backup diferencial, a sequência deve ser:  
```text
FULL WITH NORECOVERY
DIFF WITH RECOVERY
```

Exemplo conceitual:  
```sql
RESTORE DATABASE msdb
FROM DISK = 'C:\Backups\msdb_FULL.bak'
WITH REPLACE, NORECOVERY;
GO

RESTORE DATABASE msdb
FROM DISK = 'C:\Backups\msdb_DIFF.bak'
WITH RECOVERY;
GO
```

O backup full é restaurado com `NORECOVERY` para permitir a aplicação do diferencial  
O backup diferencial é restaurado com `RECOVERY` para finalizar o processo  

---

## 16 - Retornando a msdb para MULTI_USER

Após o restore, se necessário, a `msdb` deve retornar para modo `MULTI_USER`  

Exemplo:  
```sql
ALTER DATABASE msdb SET MULTI_USER;
GO
```

Depois disso, o SQL Server Agent pode ser iniciado novamente  

---

## 17 - Sequência geral de restore da msdb

Sequência conceitual:
1. Confirmar necessidade de restore
2. Validar backup full da `msdb`
3. Validar backup diferencial, quando existir
4. Parar o SQL Server Agent
5. Encerrar conexões que usam a `msdb`
6. Colocar a `msdb` em `SINGLE_USER`, se necessário
7. Restaurar backup full com `NORECOVERY` ou `RECOVERY`
8. Restaurar backup diferencial com `RECOVERY`, quando aplicável
9. Retornar `msdb` para `MULTI_USER`, se necessário
10. Iniciar o SQL Server Agent
11. Validar jobs, históricos, alertas e Database Mail

---

## 18 - Exemplo completo de restore full

Exemplo conceitual:  
```sql
ALTER DATABASE msdb SET SINGLE_USER WITH ROLLBACK IMMEDIATE;
GO

RESTORE DATABASE msdb
FROM DISK = 'C:\Backups\msdb_FULL.bak'
WITH REPLACE, RECOVERY;
GO

ALTER DATABASE msdb SET MULTI_USER;
GO
```

Após esse processo, iniciar novamente o SQL Server Agent  

---

## 19 - Exemplo completo de restore full + diferencial

Exemplo conceitual: 
```sql
ALTER DATABASE msdb SET SINGLE_USER WITH ROLLBACK IMMEDIATE;
GO

RESTORE DATABASE msdb
FROM DISK = 'C:\Backups\msdb_FULL.bak'
WITH REPLACE, NORECOVERY;
GO

RESTORE DATABASE msdb
FROM DISK = 'C:\Backups\msdb_DIFF.bak'
WITH RECOVERY;
GO

ALTER DATABASE msdb SET MULTI_USER;
GO
```

Após esse processo, iniciar novamente o SQL Server Agent  

---

## 20 - Recuperando msdb após rebuild dos bancos de sistema

Após rebuild dos bancos de sistema, a `msdb` é recriada em estado inicial  
Nesse cenário, os jobs, históricos, operadores, alertas e configurações anteriores não estarão presentes  
Se existir backup válido da `msdb`, ele deve ser restaurado para recuperar essas informações  

Fluxo resumido:  
```text
1 - Rebuild dos bancos de sistema
2 - Inicialização da instância
3 - Restore da master, se aplicável
4 - Restore da msdb
5 - Iniciar SQL Server Agent
6 - Validar jobs e configurações operacionais
```

Em muitos cenários, restaurar a `msdb` é essencial para recuperar a operação automatizada da instância  

---

## 21 - Quando não existe backup da msdb

Se não existir backup válido da `msdb`, será necessário recriar manualmente as configurações operacionais  

Exemplos:  
- Jobs
- Schedules
- Operadores
- Alertas
- Planos de manutenção
- Configurações de Database Mail
- Histórico operacional perdido
- Configurações de log shipping
- Políticas
- Rotinas automatizadas

Esse cenário pode ser trabalhoso  
Por isso, a `msdb` também deve fazer parte da estratégia de backup dos bancos de sistema  

---

## 22 - Validações após restaurar a msdb

Após restaurar a `msdb`, é importante validar os principais recursos operacionais  

Exemplo para verificar jobs:  
```sql
SELECT
    name,
    enabled,
    date_created,
    date_modified
FROM msdb.dbo.sysjobs
ORDER BY name;
GO
```

Exemplo para verificar histórico de backup:  
```sql
SELECT TOP (50)
    database_name,
    backup_start_date,
    backup_finish_date,
    type,
    backup_size
FROM msdb.dbo.backupset
ORDER BY backup_start_date DESC;
GO
```

Exemplo para verificar operadores:  
```sql
SELECT
    name,
    enabled,
    email_address
FROM msdb.dbo.sysoperators
ORDER BY name;
GO
```

Exemplo para verificar Database Mail:  
```sql
SELECT
    name,
    description,
    last_mod_datetime
FROM msdb.dbo.sysmail_profile
ORDER BY name;
GO
```

---

## 23 - Cuidados com histórico de backup e restore

A `msdb` armazena o histórico de backups e restores da instância  
Após restaurar a `msdb`, esse histórico volta ao estado do backup restaurado  
Isso significa que backups feitos após aquele ponto podem não aparecer mais no histórico da `msdb`  
Os arquivos físicos de backup podem continuar existindo, mas o histórico interno pode estar desatualizado  

Por isso, após restaurar a `msdb`, é importante validar:  
- Arquivos físicos de backup existentes
- Histórico em `msdb.dbo.backupset`
- Histórico em `msdb.dbo.backupmediafamily`
- Jobs de backup
- Rotinas externas de backup
- Documentação operacional

---

## 24 - Cuidados com SQL Server Agent

O SQL Server Agent depende fortemente da `msdb`  

Após restaurar a `msdb`, é importante validar:  
- Se o serviço inicia corretamente
- Se os jobs existem
- Se os schedules estão corretos
- Se os jobs estão habilitados ou desabilitados conforme esperado
- Se os owners dos jobs existem na instância
- Se os operadores estão configurados
- Se os alertas continuam válidos
- Se proxies e credenciais relacionadas continuam funcionando

Em alguns casos, jobs podem existir, mas falhar por causa de logins, owners, proxies ou credenciais ausentes  

---

## 25 - Cuidados com owners dos jobs

Após restore da `msdb`, os jobs podem apontar para owners que não existem mais na instância  
Isso pode ocorrer principalmente após rebuild ou recuperação em outra instância  

Exemplo de consulta:  
```sql
SELECT
    j.name AS job_name,
    SUSER_SNAME(j.owner_sid) AS owner_name,
    j.enabled
FROM msdb.dbo.sysjobs AS j
ORDER BY j.name;
GO
```

Se algum owner estiver inválido, pode ser necessário alterar o owner do job  

Exemplo conceitual:

```sql
EXEC msdb.dbo.sp_update_job
    @job_name = N'JobName',
    @owner_login_name = N'sa';
GO
```

---

## 26 - Cuidados com Database Mail

Se a instância utiliza Database Mail, a `msdb` armazena perfis, contas, mensagens e anexos relacionados  

Após restore da `msdb`, é importante validar:  
- Perfis
- Contas
- Permissões de uso dos perfis
- Histórico de mensagens
- Anexos armazenados
- Jobs ou alertas que enviam e-mail
- Configuração externa de SMTP

Exemplo para consultar mensagens recentes:  
```sql
SELECT TOP (50)
    mailitem_id,
    sent_status,
    send_request_date,
    sent_date,
    recipients,
    subject
FROM msdb.dbo.sysmail_allitems
ORDER BY send_request_date DESC;
GO
```

---

## 27 - Cuidados com log shipping

Log shipping é uma estratégia de alta disponibilidade e recuperação de desastre em que backups de log de transações são gerados periodicamente em um banco primário, copiados para um servidor secundário e restaurados nesse banco secundário em modo `NORECOVERY` ou `STANDBY`  
Esse processo mantém uma cópia do banco relativamente atualizada em outro servidor, dependendo da frequência dos backups, cópias e restores dos logs  
A `msdb` participa desse processo porque armazena metadados, configurações, histórico e jobs relacionados ao log shipping  
Por isso, após restaurar a `msdb`, ambientes que usam log shipping precisam ser validados com cuidado  
Após restore da `msdb`, ambientes com log shipping precisam de validação especial  

Pontos de atenção:  
- Jobs de backup do log
- Jobs de copy
- Jobs de restore
- Configurações primárias
- Configurações secundárias
- Monitoramento do log shipping
- Histórico de execução

Se a `msdb` restaurada estiver desatualizada, o estado operacional do log shipping pode não refletir a realidade atual  

---

## 28 - Diferença entre recuperar msdb e recuperar dados de usuário

A recuperação da `msdb` não recupera dados dos bancos de usuário  
Ela recupera metadados operacionais da instância  

Exemplo:  
```text
Restore da msdb recupera jobs, histórico, operadores e Database Mail
Restore de banco de usuário recupera dados da aplicação
```

Por isso, a recuperação da `msdb` deve ser vista como parte da recuperação da instância, não como recuperação dos dados de negócio  

---

## 29 - Fluxo resumido com backup da msdb

Fluxo resumido quando existe backup válido da `msdb`:  
```text
1 - Parar SQL Server Agent
2 - Encerrar conexões usando msdb
3 - Colocar msdb em SINGLE_USER, se necessário
4 - Restaurar backup full
5 - Restaurar backup diferencial, se aplicável
6 - Retornar msdb para MULTI_USER
7 - Iniciar SQL Server Agent
8 - Validar jobs
9 - Validar operadores e alertas
10 - Validar histórico de backup
11 - Validar Database Mail
```

---

## 30 - Fluxo resumido sem backup da msdb

Fluxo resumido quando não existe backup válido da `msdb`:  
```text
1 - Recriar jobs
2 - Recriar schedules
3 - Recriar operadores
4 - Recriar alertas
5 - Recriar planos de manutenção
6 - Reconfigurar Database Mail
7 - Reconfigurar log shipping, se aplicável
8 - Recriar rotinas administrativas
9 - Validar SQL Server Agent
10 - Criar backup atualizado da msdb
```

Esse cenário pode ser trabalhoso e gerar perda de histórico operacional  

---

## 31 - Resumo

O banco `msdb` armazena metadados operacionais importantes da instância SQL Server  
Ele é usado por recursos como SQL Server Agent, jobs, operadores, alertas, histórico de backup, histórico de restore, Database Mail e log shipping  
Por padrão, a `msdb` utiliza recovery model `SIMPLE`  
A estratégia de recuperação normalmente envolve backup full e, quando necessário, backup diferencial  
Antes de restaurar a `msdb`, é importante parar o SQL Server Agent e garantir que nenhuma conexão esteja usando o banco  
Após o restore, devem ser validados jobs, schedules, operadores, alertas, Database Mail, histórico de backup e recursos dependentes da `msdb`  
Sem backup da `msdb`, será necessário recriar manualmente configurações operacionais importantes da instância

---

## Referências

- [Banco de dados msdb](https://learn.microsoft.com/pt-br/sql/relational-databases/databases/msdb-database?view=sql-server-ver16)
- [sp_update_job (Transact-SQL)](https://learn.microsoft.com/pt-br/sql/relational-databases/system-stored-procedures/sp-update-job-transact-sql?view=sql-server-ver16)
- [Backup e restore: bancos de dados de sistema (SQL Server)](https://learn.microsoft.com/pt-br/sql/relational-databases/backup-restore/back-up-and-restore-of-system-databases-sql-server?view=sql-server-ver16)
- [Histórico de backup e informações de cabeçalho (SQL Server)](https://learn.microsoft.com/pt-br/sql/relational-databases/backup-restore/backup-history-and-header-information-sql-server?view=sql-server-ver16)