# A0035 – Recovering the model Database

> **Author:** Rafael Binda  
> **Created:** 2026-04-28  
> **Version:** 1.0  

---

## Descrição

Este documento apresenta os principais conceitos relacionados à recuperação do banco de sistema `model` no SQL Server

O banco `model` é utilizado como modelo para criação de novos bancos de dados e também participa indiretamente do processo de inicialização da instância, pois a `tempdb` é recriada com base na estrutura da `model`

Quando a `model` está corrompida ou indisponível, a instância SQL Server pode não inicializar corretamente, tornando necessário restaurar um backup válido, copiar arquivos de uma instância compatível ou executar rebuild dos bancos de sistema

---

## 1 - O que é o banco model

O banco `model` é um banco de sistema usado como modelo para criação de novos bancos de dados  
Quando um novo banco é criado, o SQL Server usa a estrutura da `model` como base  
Configurações e objetos existentes na `model` podem ser herdados por novos bancos  

Exemplos de itens que podem existir na `model`:  
- Tabelas
- Views
- Procedures
- Users
- Schemas
- Configurações de banco
- Opções de banco
- Tamanho inicial dos arquivos
- Configuração de crescimento dos arquivos

Em ambientes comuns, a `model` costuma ser simples e pouco alterada  
Mesmo assim, ela é importante para o funcionamento da instância  

---

## 2 - Por que a model é importante

A `model` é importante porque serve como base para novos bancos de dados  
Além disso, ela participa do processo de criação da `tempdb` durante a inicialização da instância  
Como a `tempdb` é recriada toda vez que o SQL Server inicia, a `model` precisa estar operacional para que esse processo ocorra corretamente  

Relação conceitual:  
```text
model  = modelo usado para criação de bancos
tempdb = recriada no startup com base na model
```

Se a `model` estiver corrompida ou inacessível, a instância pode falhar durante o startup  

---

## 3 - Relação entre model e tempdb

A `tempdb` é recriada a cada inicialização do SQL Server  
Para essa recriação, o SQL Server utiliza a `model` como referência  
Por isso, falhas na `model` podem impedir a criação da `tempdb`  

Exemplo conceitual:  
```text
SQL Server inicia
SQL Server carrega a master
SQL Server precisa recuperar a model
SQL Server usa a model para recriar a tempdb
Se a model falha, a tempdb pode não ser criada
Se a tempdb não é criada, a instância não inicializa corretamente
```

Por isso, embora a falha possa aparecer como problema de inicialização da instância, a causa pode estar na `model`  

---

## 4 - Diferença entre master, model, msdb e tempdb

Cada banco de sistema possui um papel diferente na instância  

```text
master = catálogo principal da instância
model  = modelo para criação de bancos e base para recriação da tempdb
msdb   = metadados operacionais, jobs, histórico, alertas e Database Mail
tempdb = banco temporário recriado a cada inicialização
```

A `model` não armazena os metadados principais da instância como a `master`  
Também não armazena histórico operacional como a `msdb`  
Mas sua indisponibilidade pode impedir a inicialização normal da instância  

---

## 5 - O que pode causar falha na model

A `model` pode apresentar problemas por motivos semelhantes a outros bancos de dados  

Exemplos:  
- Corrupção no arquivo de dados da `model`
- Corrupção no arquivo de log da `model`
- Arquivos físicos ausentes
- Caminho incorreto dos arquivos
- Volume indisponível
- Permissão insuficiente para a conta de serviço
- Falha no storage
- Disco sem espaço
- Alteração incorreta em arquivos do sistema
- Problema após tentativa de mover bancos de sistema
- Restauração incorreta
- Intervenção manual indevida

Quando a `model` falha, a instância pode não conseguir concluir o startup  

---

## 6 - Como identificar problema na model

Quando a instância não inicializa, a investigação deve começar pelos logs  

Locais importantes:  
```text
SQL Server Error Log
Windows Event Viewer
Windows Logs
Application
```

O SQL Server Error Log pode indicar falha ao abrir, recuperar ou acessar a `model`  
O Windows Event Viewer pode registrar falhas do serviço SQL Server durante a inicialização  

Pontos para verificar:  
- Mensagens envolvendo `model`
- Mensagens envolvendo `tempdb`
- Mensagens de erro de I/O
- Caminho dos arquivos da `model`
- Existência dos arquivos físicos
- Permissões da conta de serviço
- Espaço disponível no volume
- Estado do storage

É importante avaliar se o problema é realmente na `model` ou se é um reflexo de outro problema, como falha em volume ou permissão  

---

## 7 - Verificando os arquivos da model

Quando a instância ainda inicializa, é possível verificar os arquivos da `model` via T-SQL  

Exemplo:  
```sql
SELECT
    DB_NAME(database_id) AS database_name,
    file_id,
    type_desc,
    name AS logical_name,
    physical_name,
    state_desc
FROM sys.master_files
WHERE database_id = DB_ID('model')
ORDER BY file_id;
GO
```

Também é possível usar:  
```sql
EXEC sp_helpdb 'model';
GO
```

Essas consultas ajudam a identificar:  
- Nome lógico dos arquivos
- Caminho físico
- Arquivo de dados
- Arquivo de log
- Estado dos arquivos

---

## 8 - Estratégias de recuperação da model

As principais estratégias de recuperação da `model` são:  
```text
1 - Restaurar backup válido da model
2 - Copiar arquivos da model de outra instância compatível
3 - Executar rebuild dos bancos de sistema
```

A melhor alternativa depende do estado da instância, da existência de backup e da compatibilidade com outra instância disponível  

---

## 9 - Quando existe backup da model

Se existir backup válido da `model`, a recuperação pode ser feita por restore  
Esse é o cenário mais organizado e seguro

Exemplo conceitual:  
```sql
RESTORE DATABASE model
FROM DISK = 'C:\Backups\model_FULL.bak'
WITH REPLACE, RECOVERY;
GO
```

Antes do restore, é importante garantir que não existam conexões usando a `model`  
Em cenário normal, a `model` não costuma ter conexões de usuário, mas ainda assim a validação é importante  

---

## 10 - Backup da model

O backup da `model` deve fazer parte da estratégia de backup dos bancos de sistema  

Exemplo:  
```sql
BACKUP DATABASE model
TO DISK = 'C:\Backups\model_FULL.bak'
WITH CHECKSUM, COMPRESSION, STATS = 10;
GO
```

O backup da `model` deve ser feito após alterações relevantes nesse banco  

Exemplos:  
- Alteração de tamanho inicial dos arquivos
- Alteração de crescimento dos arquivos
- Criação de objetos padrão
- Alteração de opções do banco
- Alteração de usuários ou permissões
- Customizações feitas para novos bancos

Em muitos ambientes, a `model` muda pouco  
Mesmo assim, é importante manter backup válido  

---

## 11 - Quando copiar arquivos de outra instância compatível

Se a instância não inicializa por corrupção da `model` e existe outra instância com o mesmo build, pode ser possível copiar os arquivos da `model` dessa outra instância  
Essa alternativa deve ser avaliada com cuidado  

Condições importantes:  
- Mesma versão do SQL Server
- Mesmo build ou build compatível
- Mesma edição, quando possível
- Mesma collation, quando possível
- Mesma arquitetura
- Instância de origem saudável
- Serviços parados no momento da cópia
- Caminhos ajustados corretamente

Exemplo conceitual:  
```text
1 - Parar a instância de origem, se necessário
2 - Copiar model.mdf e modellog.ldf
3 - Parar a instância com problema
4 - Substituir os arquivos corrompidos da model
5 - Ajustar permissões dos arquivos
6 - Iniciar a instância com problema
7 - Validar SQL Server Error Log
```

Essa abordagem pode resolver cenários simples de perda ou corrupção da `model`, mas não substitui uma estratégia adequada de backup  

---

## 12 - Quando executar rebuild dos bancos de sistema

Se não existir backup válido da `model` e não houver outra instância compatível, pode ser necessário executar rebuild dos bancos de sistema  

O rebuild recria os bancos:  
```text
master
model
msdb
tempdb
```

Esse procedimento é mais invasivo porque não recria apenas a `model`  
Ele recria o conjunto dos bancos de sistema  
Após o rebuild, pode ser necessário restaurar backups da `master` e da `msdb` para recuperar configurações e metadados operacionais  

---

## 13 - Preservar master e msdb antes do rebuild

Se a falha estiver concentrada na `model`, e a `master` e a `msdb` ainda estiverem preservadas nos arquivos físicos, pode ser importante tentar protegê-las antes de executar procedimentos destrutivos  

Exemplo conceitual:  
```text
Copiar master.mdf e mastlog.ldf para local seguro
Copiar msdbdata.mdf e msdblog.ldf para local seguro
Copiar arquivos de bancos de usuário para local seguro, se necessário
```

Essa ação deve ser feita com os serviços parados  
O objetivo é preservar evidências e aumentar as opções de recuperação  

Mesmo assim, a estratégia mais segura continua sendo restaurar backups válidos dos bancos de sistema  

---

## 14 - Uso da trace flag -T3608

A trace flag `-T3608` faz o SQL Server recuperar apenas a `master` durante a inicialização  
Isso pode permitir acesso emergencial à instância quando outros bancos de sistema impedem o startup normal  

Exemplo conceitual:  
```text
-T3608 = recupera somente a master no startup
```

Esse recurso pode ser útil quando a `model` está corrompida e a instância não inicializa normalmente  
A ideia é iniciar a instância de forma limitada para conseguir acessar a `master` e executar ações emergenciais, como backup da `master` e da `msdb`, antes de realizar rebuild dos bancos de sistema  

---

## 15 - Quando usar -T3608

O uso de `-T3608` deve ser tratado como recurso emergencial  

Ele pode ser útil em cenários como:  
- `model` corrompida impedindo startup
- Necessidade de acessar a `master` antes do rebuild
- Necessidade de fazer backup da `master`
- Necessidade de fazer backup da `msdb`, quando possível
- Necessidade de preservar metadados antes de reconstruir bancos de sistema

Esse parâmetro não deve ser usado como configuração permanente  
Após o procedimento emergencial, ele deve ser removido  

---

## 16 - Configurando -T3608 no SQL Server Configuration Manager

Caminho conceitual:  
```text
SQL Server Configuration Manager -> SQL Server Services -> SQL Server (MSSQLSERVER) -> Properties -> Startup Parameters
```

Parâmetro a ser adicionado temporariamente:  
```text
-T3608
```

Em alguns cenários, pode ser combinado com modo single-user:  
```text
-mSQLCMD
-T3608
```

Também pode ser necessário usar configuração mínima:  
```text
-f
-mSQLCMD
-T3608
```

Após o procedimento, os parâmetros devem ser removidos e a instância deve ser reiniciada normalmente  

---

## 17 - Conectando via sqlcmd com -T3608

Com a instância iniciada em modo emergencial, a conexão deve ser feita via `sqlcmd`  

Exemplo para instância default:  
```cmd
sqlcmd -S SRVSQLSERVER -E
```

Exemplo usando login SQL Server:  
```cmd
sqlcmd -S SRVSQLSERVER -U sa -P senha
```

Exemplo para instância nomeada:  
```cmd
sqlcmd -S SRVSQLSERVER\NOMEINSTANCIA -E
```

Se a instância estiver em modo single-user, é importante garantir que SQL Server Agent, aplicações e ferramentas de monitoramento estejam paradas para não consumir a única conexão disponível  

---

## 18 - Backup emergencial da master

Se a instância conseguiu subir com `-T3608`, pode ser possível fazer backup emergencial da `master` antes de qualquer rebuild  

Exemplo:  
```sql
BACKUP DATABASE master
TO DISK = 'C:\Backups\master_emergency.bak'
WITH CHECKSUM, COMPRESSION, STATS = 10;
GO
```

Esse backup pode ser útil para preservar metadados da instância antes de uma intervenção mais destrutiva  

---

## 19 - Backup emergencial da msdb

Dependendo do estado da instância e da possibilidade de acessar a `msdb`, também pode ser interessante tentar backup emergencial da `msdb`  

Exemplo:  
```sql
BACKUP DATABASE msdb
TO DISK = 'C:\Backups\msdb_emergency.bak'
WITH CHECKSUM, COMPRESSION, STATS = 10;
GO
```

Se a `msdb` não estiver recuperada ou acessível nesse modo, essa etapa pode não ser possível  
Nesse caso, será necessário depender de backups anteriores ou reconstrução manual das configurações operacionais  

---

## 20 - Rebuild dos bancos de sistema

Quando não há backup válido da `model` e não existe uma instância compatível para copiar os arquivos, o rebuild pode ser necessário  
O rebuild é feito pelo setup do SQL Server  

Caminho conceitual:  
```text
C:\Program Files\Microsoft SQL Server\160\Setup Bootstrap\SQL2022
```

Exemplo conceitual:  
```cmd
setup /QUIET /ACTION=REBUILDDATABASE /INSTANCENAME=MSSQLSERVER /SQLSYSADMINACCOUNTS="SRVSQLSERVER\SQLSERVICE" /SAPWD="SenhaForteDoSA" /SQLCOLLATION=Latin1_General_100_CI_AI_SC_UTF8
```

Parâmetros principais:  
```text
/QUIET                  = executa sem interface gráfica
/ACTION=REBUILDDATABASE = indica rebuild dos bancos de sistema
/INSTANCENAME           = nome da instância
/SQLSYSADMINACCOUNTS    = conta que será sysadmin após o rebuild
/SAPWD                  = senha do login sa
/SQLCOLLATION           = collation da instância
```

É importante informar a collation correta da instância original  

---

## 21 - Após o rebuild

Após o rebuild, os bancos de sistema estarão em estado inicial  
Isso significa que a instância perdeu configurações que estavam na `master` e na `msdb`, caso backups não sejam restaurados  

Fluxo resumido:  
```text
1 - Executar rebuild dos bancos de sistema
2 - Iniciar a instância
3 - Restaurar master, se existir backup válido
4 - Restaurar msdb, se existir backup válido
5 - Validar model
6 - Validar tempdb
7 - Validar bancos de usuário
8 - Validar logins, jobs, certificados e configurações
```

Se não houver backup da `master` e da `msdb`, será necessário reconstruir configurações manualmente  

---

## 22 - Attach dos bancos de usuário após rebuild

Após rebuild sem restore da `master`, os bancos de usuário podem não aparecer na instância  
Isso ocorre porque a nova `master` não possui os metadados desses bancos  
Se os arquivos dos bancos de usuário estiverem preservados, pode ser necessário fazer attach  

Exemplo conceitual:  
```sql
CREATE DATABASE ExampleDB
ON
(
    FILENAME = 'C:\MSSQLSERVER\DATA\ExampleDB.mdf'
),
(
    FILENAME = 'C:\MSSQLSERVER\LOG\ExampleDB_log.ldf'
)
FOR ATTACH;
GO
```

Após o attach, validar:  
- Estado do banco
- Usuários órfãos
- Permissões
- Owner do banco
- Jobs relacionados
- Certificados e chaves
- Aplicações dependentes

---

## 23 - Restaurando backup da model

Quando a instância consegue inicializar e existe backup válido da `model`, o restore pode ser feito diretamente  

Exemplo conceitual:  
```sql
RESTORE DATABASE model
FROM DISK = 'C:\Backups\model_FULL.bak'
WITH REPLACE, RECOVERY;
GO
```

Após restaurar a `model`, é importante reiniciar a instância para validar se a `tempdb` será recriada corretamente no próximo startup  

---

## 24 - Validando a model após recuperação

Após recuperar a `model`, valide o estado do banco  

Exemplo:  
```sql
SELECT
    name,
    state_desc,
    recovery_model_desc,
    user_access_desc
FROM sys.databases
WHERE name = 'model';
GO
```

Valide também os arquivos físicos:

```sql
SELECT
    DB_NAME(database_id) AS database_name,
    file_id,
    type_desc,
    name AS logical_name,
    physical_name,
    state_desc
FROM sys.master_files
WHERE database_id = DB_ID('model')
ORDER BY file_id;
GO
```

Se a `model` estiver íntegra, a instância deve conseguir criar novos bancos e recriar a `tempdb` durante o startup  

---

## 25 - Validando criação de novo banco

Como a `model` é usada como base para novos bancos, uma forma simples de validação é criar um banco de teste  

Exemplo:  
```sql
CREATE DATABASE ModelValidationDB;
GO

DROP DATABASE ModelValidationDB;
GO
```

Se o banco for criado e removido corretamente, isso indica que a `model` está funcional para criação de novos bancos  

---

## 26 - Validando a tempdb após recuperar a model

Como a `tempdb` depende da `model` durante a inicialização, é importante validar se a `tempdb` foi recriada corretamente após o restart  

Exemplo:  
```sql
USE tempdb;
GO

SELECT
file_id,
type_desc,
name AS logical_name,
physical_name,
state_desc
FROM sys.database_files
ORDER BY file_id;
GO
```

Também é importante verificar:  
- SQL Server Error Log
- Windows Event Viewer
- Estado da instância
- Estado da `tempdb`
- Mensagens relacionadas à `model`
- Mensagens relacionadas à criação da `tempdb`

---

## 27 - Cuidados com customizações na model

Alguns ambientes customizam a `model` para que novos bancos sejam criados com objetos ou configurações específicas  

Exemplos:  
- Tamanho inicial dos arquivos
- Crescimento dos arquivos
- Recovery model
- Collation
- Schemas
- Users
- Tabelas padrão
- Procedures administrativas
- Configurações de banco

Se a `model` for restaurada a partir de backup antigo, copiada de outra instância ou recriada via rebuild, essas customizações podem ser perdidas ou alteradas  

---

## 28 - Cuidados com collation

A collation da instância e dos bancos de sistema deve ser tratada com cuidado  
Ao executar rebuild dos bancos de sistema, a collation precisa ser informada corretamente  
Se a collation for diferente da original, podem ocorrer diferenças importantes no comportamento da instância  

Pontos impactados:  
- Comparação de strings
- Ordenação
- Sensibilidade a acentos
- Sensibilidade a maiúsculas e minúsculas
- Comportamento de bancos criados depois do rebuild
- Compatibilidade com aplicações

Por isso, antes de rebuild, é importante identificar e registrar a collation original  

---

## 29 - Cuidados com versão e build

Ao copiar arquivos da `model` de outra instância, é importante considerar versão e build  
A instância de origem deve ser compatível com a instância de destino  

Pontos de atenção:  
- Versão do SQL Server
- Build
- Cumulative Update
- Edição
- Collation
- Caminhos dos arquivos
- Contas de serviço
- Recursos instalados

Copiar arquivos de uma instância incompatível pode impedir a inicialização ou gerar comportamento inesperado  

---

## 30 - O que não fazer

Em uma falha relacionada à `model`, evite ações precipitadas  

Pontos de atenção:  
- Não apagar arquivos sem preservar cópia
- Não executar rebuild sem entender o impacto
- Não copiar arquivos de instância incompatível
- Não ignorar collation original
- Não esquecer que o rebuild recria também `master`, `msdb` e `tempdb`
- Não deixar `-T3608`, `-f` ou `-mSQLCMD` configurados após a recuperação
- Não ignorar SQL Server Error Log e Windows Event Viewer
- Não assumir que o problema é da `tempdb` sem investigar a `model`

---

## 31 - Fluxo resumido com backup da model

Fluxo resumido quando existe backup válido da `model`:  
```text
1 - Confirmar erro relacionado à model
2 - Verificar SQL Server Error Log e Event Viewer
3 - Validar backup da model
4 - Restaurar backup da model, se a instância permitir
5 - Reiniciar a instância
6 - Validar model
7 - Validar tempdb
8 - Validar criação de novo banco
9 - Criar backup atualizado da model
```

---

## 32 - Fluxo resumido copiando model de outra instância

Fluxo resumido quando existe outra instância compatível:  
```text
1 - Confirmar versão e build da instância de origem
2 - Confirmar collation e compatibilidade
3 - Parar serviços envolvidos
4 - Preservar arquivos atuais da model com problema
5 - Copiar model.mdf e modellog.ldf da instância compatível
6 - Ajustar permissões dos arquivos
7 - Iniciar a instância com problema
8 - Validar SQL Server Error Log
9 - Validar model
10 - Validar tempdb
11 - Criar backup atualizado da model
```

---

## 33 - Fluxo resumido com -T3608 e rebuild

Fluxo resumido em cenário grave sem backup válido da `model`:  
```text
1 - Confirmar que a model impede o startup
2 - Parar SQL Server Agent e serviços dependentes
3 - Adicionar -T3608, e se necessário -mSQLCMD ou -f
4 - Iniciar a instância em modo emergencial
5 - Conectar via sqlcmd
6 - Tentar backup emergencial da master
7 - Tentar backup emergencial da msdb
8 - Parar a instância
9 - Remover parâmetros emergenciais
10 - Executar rebuild dos bancos de sistema
11 - Restaurar master, se houver backup
12 - Restaurar msdb, se houver backup
13 - Validar model e tempdb
14 - Recriar configurações faltantes
15 - Criar backups atualizados dos bancos de sistema
```

---

## 34 - Resumo

O banco `model` é usado como modelo para criação de novos bancos de dados  
Ele também é importante para a recriação da `tempdb` durante a inicialização da instância  
Se a `model` estiver corrompida ou inacessível, a instância SQL Server pode não inicializar corretamente  
A melhor estratégia de recuperação é restaurar um backup válido da `model`  
Quando não existe backup, pode ser possível copiar arquivos de uma instância compatível  
Em cenários mais graves, pode ser necessário usar `-T3608` para inicialização emergencial e depois executar rebuild dos bancos de sistema  
O rebuild recria `master`, `model`, `msdb` e `tempdb`, por isso deve ser tratado como procedimento invasivo  
Após recuperar a `model`, é importante validar a criação de novos bancos e a recriação da `tempdb`

---

## Referências

- [Banco de dados model](https://learn.microsoft.com/pt-br/sql/relational-databases/databases/model-database?view=sql-server-ver16)
- [Reconstruir bancos de dados do sistema](https://learn.microsoft.com/pt-br/sql/relational-databases/databases/rebuild-system-databases?view=sql-server-ver16)
- [DBCC TRACEON – Sinalizadores de rastreamento (Transact-SQL)](https://learn.microsoft.com/pt-br/sql/t-sql/database-console-commands/dbcc-traceon-trace-flags-transact-sql?view=sql-server-ver16)
- [Opções de inicialização do serviço do Database Engine](https://learn.microsoft.com/pt-br/sql/database-engine/configure-windows/database-engine-service-startup-options?view=sql-server-ver16)