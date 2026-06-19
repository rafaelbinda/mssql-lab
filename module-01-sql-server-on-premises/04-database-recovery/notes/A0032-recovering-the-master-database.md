# A0032 – Recovering the master Database

> **Author:** Rafael Binda  
> **Created:** 2026-04-26  
> **Version:** 1.0  

---

## Descrição

Este documento apresenta os principais conceitos relacionados à recuperação do banco de sistema `master` no SQL Server

O banco `master` é essencial para a inicialização da instância e armazena metadados fundamentais do SQL Server

Quando a `master` está corrompida ou indisponível, a instância pode não inicializar, tornando necessário restaurar um backup válido ou reconstruir os bancos de sistema

---

## 1 - O que é o banco master

O banco `master` é o catálogo principal da instância SQL Server  
Ele armazena metadados essenciais para o funcionamento da instância  
Sem a `master`, a instância não consegue operar normalmente  

Exemplos de informações armazenadas ou controladas pela `master`:  
- Logins da instância
- Configurações de servidor
- Linked servers
- Endpoints
- Informações sobre bancos de dados existentes
- Localização dos arquivos dos bancos de dados
- Configurações de segurança em nível de instância
- Certificados e chaves criadas na `master`
- Configurações necessárias para inicialização da instância

---

## 2 - Por que a master é crítica

A `master` é carregada durante o processo de inicialização do SQL Server  
Se o SQL Server não conseguir localizar ou carregar os arquivos da `master`, o serviço pode não inicializar  

Isso pode ocorrer por problemas como:  
- Corrupção no arquivo de dados da `master`
- Corrupção no arquivo de log da `master`
- Arquivos físicos ausentes
- Caminho de arquivo incorreto
- Falha no volume onde a `master` está armazenada
- Problema de permissão nos arquivos
- Problema no storage
- Falha durante alteração de configuração da instância

Quando a instância não inicializa, não é possível simplesmente abrir o SSMS e executar um restore comum  
Nesses casos, o procedimento precisa ser feito em modo especial de inicialização ou, em situações mais graves, após rebuild dos bancos de sistema  

---

## 3 - Bancos de sistema no SQL Server

Os principais bancos de sistema são:
```text
master = catálogo principal da instância
msdb   = histórico e metadados operacionais
tempdb = área temporária recriada a cada inicialização
model  = modelo usado para criação de novos bancos
```

A `master` é o banco mais crítico para a inicialização da instância  
A `msdb`, `tempdb` e `model` também são importantes, mas possuem comportamentos e estratégias de recuperação diferentes  
Este documento foca apenas na recuperação da `master`  

---

## 4 - Situações possíveis

Em uma falha envolvendo a `master`, normalmente existem dois cenários principais:

```text
1 - O SQL Server consegue inicializar
2 - O SQL Server não consegue inicializar
```

Dentro desses cenários, existe outra divisão importante:  
```text
1 - Existe backup válido da master
2 - Não existe backup válido da master
```

A existência de backup da `master` muda completamente o cenário de recuperação  

---

## 5 - Quando o SQL Server consegue inicializar

Se o SQL Server ainda consegue inicializar, mesmo com algum problema na `master`, o caminho mais seguro é restaurar a `master` a partir de um backup válido  

Nesse caso, o procedimento geral é:  
1. Parar o SQL Server Agent
2. Colocar a instância em modo single-user
3. Conectar via `sqlcmd`
4. Executar o restore da `master` com `WITH REPLACE`
5. Aguardar o SQL Server encerrar automaticamente após o restore
6. Remover os parâmetros especiais de inicialização
7. Reiniciar o SQL Server normalmente
8. Validar a instância

A restauração da `master` exige cuidado porque afeta a instância inteira  

---

## 6 - Quando o SQL Server não consegue inicializar

Se o SQL Server não consegue inicializar porque a `master` está corrompida, ausente ou inacessível, o restore direto pode não ser possível  
Nesse cenário, pode ser necessário fazer primeiro o rebuild dos bancos de sistema

O rebuild recria os bancos:  
```text
master
model
msdb
tempdb
```

Depois do rebuild, se existir backup válido da `master`, o próximo passo é restaurá-lo  
Se não existir backup da `master`, será necessário reconstruir manualmente as configurações da instância  

Isso pode incluir:  
- Recriação de logins
- Recriação de linked servers
- Recriação de endpoints
- Reconfiguração de permissões
- Reconfiguração de certificados
- Reconfiguração de jobs relacionados à instância
- Attach dos bancos de usuário
- Correção de usuários órfãos
- Reaplicação de configurações em nível de servidor

---

## 7 - Nunca esquecer o backup da master

O backup da `master` é essencial para recuperação da instância  
Sem backup da `master`, a recuperação pode se tornar muito mais trabalhosa  
Em caso de perda total da `master` sem backup, pode ser necessário recriar manualmente várias configurações da instância  

Exemplos de itens que podem ser perdidos ou precisar de reconstrução manual:
- Logins SQL Server
- Permissões em nível de servidor
- Linked servers
- Certificados
- Chaves
- Endpoints
- Credenciais
- Configurações específicas da instância
- Metadados de bancos de dados
- Configurações de segurança

Por isso, sempre que houver alteração importante em nível de instância, é recomendado garantir que exista backup atualizado da `master`  

Exemplos de alterações que justificam novo backup da `master`:  
- Criação de login
- Alteração de login
- Criação de linked server
- Alteração de configuração da instância
- Criação de endpoint
- Criação de certificado na `master`
- Alteração relevante de segurança

---

## 8 - Antes de restaurar a master

Antes de restaurar a `master`, é importante confirmar:  
- Existe backup válido da `master`
- O backup pertence à mesma instância ou a uma instância compatível
- A versão, edição e patch level são compatíveis
- O caminho do arquivo de backup está acessível
- A conta de serviço do SQL Server tem permissão no arquivo
- O SQL Server Agent está parado
- Não existem conexões concorrentes consumindo o modo single-user
- Existe plano de validação após o restore

Em recuperação de desastre, a instância usada para restaurar a `master` deve ser o mais próxima possível da instância original  
Isso inclui versão, edição, patch level, recursos instalados e configuração externa  

---

## 9 - Parâmetros de inicialização

Para restaurar a `master`, normalmente é necessário iniciar a instância em modo single-user  
Isso pode ser feito usando parâmetros de inicialização  
Principais parâmetros envolvidos:

```text
-m = inicia a instância em modo single-user
-f = inicia a instância em configuração mínima
```

Também é possível restringir o modo single-user ao `sqlcmd` usando:

```text
-mSQLCMD
```

Esse formato ajuda a evitar que outra aplicação, como SSMS, SQL Server Agent ou monitoramento, consuma a única conexão disponível  

---

## 10 - Configurando pelo SQL Server Configuration Manager

Caminho conceitual:

```text
SQL Server Configuration Manager -> SQL Server Services -> SQL Server (MSSQLSERVER) -> Properties -> Startup Parameters
```

Parâmetros que podem ser adicionados temporariamente:  
```text
-mSQLCMD
```

ou, em cenários específicos:  
```text
-f
-mSQLCMD
```

Após o procedimento, esses parâmetros devem ser removidos  
Se eles permanecerem configurados, a instância continuará inicializando em modo especial  

---

## 11 - Por que parar o SQL Server Agent

Antes de iniciar a instância em modo single-user, é importante parar o SQL Server Agent  
O motivo é simples:  
```text
Modo single-user = apenas uma conexão permitida
```

Se o SQL Server Agent iniciar e conectar primeiro, ele pode consumir a única conexão disponível  
Isso impede a conexão via `sqlcmd` para executar o restore da `master`  
Também é importante evitar conexões automáticas de:  
- SSMS Object Explorer
- Aplicações
- Serviços de monitoramento
- Jobs
- Ferramentas de backup
- Serviços de integração

---

## 12 - Conexão via sqlcmd

Com a instância em modo single-user, a conexão recomendada é via `sqlcmd`  
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

Para instância default, normalmente não é necessário informar `\MSSQLSERVER` no nome da conexão  
`MSSQLSERVER` é o nome interno da instância default, mas a conexão normalmente usa apenas o nome do servidor  

---

## 13 - Restore da master

Após conectar via `sqlcmd`, o restore da `master` pode ser executado  

Exemplo conceitual:
```sql
RESTORE DATABASE master
FROM DISK = 'C:\Backups\master.bak'
WITH REPLACE;
GO
```

A opção `WITH REPLACE` permite substituir a `master` existente pelo conteúdo do backup  
Após restaurar a `master`, o SQL Server encerra automaticamente a instância  
Esse comportamento é esperado  
Depois disso, é necessário remover os parâmetros especiais de inicialização e iniciar o serviço normalmente  

---

## 14 - Sequência pelo SQL Server Configuration Manager

Sequência conceitual para restore da `master` quando a instância consegue inicializar em modo especial:  
1. Abrir o SQL Server Configuration Manager
2. Acessar SQL Server Services
3. Abrir as propriedades do serviço SQL Server
4. Adicionar o parâmetro `-mSQLCMD`
5. Parar o SQL Server Agent
6. Parar o serviço SQL Server
7. Iniciar o serviço SQL Server
8. Abrir o Prompt de Comando como administrador
9. Conectar via `sqlcmd`
10. Executar `RESTORE DATABASE master FROM DISK = ... WITH REPLACE`
11. Aguardar a instância encerrar automaticamente
12. Remover o parâmetro `-mSQLCMD`
13. Iniciar o serviço SQL Server normalmente
14. Iniciar o SQL Server Agent
15. Validar a instância

---

## 15 - Exemplo com sqlcmd

Exemplo usando autenticação Windows:  
```cmd
sqlcmd -S SRVSQLSERVER -E
```

Dentro do `sqlcmd`:  
```sql
RESTORE DATABASE master
FROM DISK = 'C:\Backups\master.bak'
WITH REPLACE;
GO
```

Exemplo usando login `sa`:  
```cmd
sqlcmd -S SRVSQLSERVER -U sa -P senha
```

Dentro do `sqlcmd`:  
```sql
RESTORE DATABASE master
FROM DISK = 'C:\Backups\master.bak'
WITH REPLACE;
GO
```

Após o `GO`, o restore é executado  
Ao concluir, o SQL Server encerra a instância automaticamente  

---

## 16 - E se o SQL Server não inicializa

Se o SQL Server não inicializa porque a `master` está corrompida ou inacessível, pode ser necessário rebuild dos bancos de sistema  
O rebuild recria os bancos de sistema em seu estado inicial  

Bancos recriados no rebuild:  
```text
master
model
msdb
tempdb
```

Esse procedimento não reconstrói apenas a `master`  
Ele recria os bancos de sistema  
Por isso, configurações e objetos existentes nesses bancos podem ser perdidos se não houver backup válido  

---

## 17 - Rebuild dos bancos de sistema

O rebuild dos bancos de sistema é feito usando o setup do SQL Server  

Normalmente, o setup fica em um caminho semelhante a:  
```text
C:\Program Files\Microsoft SQL Server\160\Setup Bootstrap\SQL2022
```

O número `160` representa a versão interna do SQL Server 2022  
O nome da pasta pode variar conforme a versão instalada  
Antes de executar o rebuild, é importante estar no diretório correto do setup  

---

## 18 - Exemplo de comando para rebuild

Exemplo conceitual de rebuild dos bancos de sistema:

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

É importante usar a collation correta da instância original  
Se a collation for informada incorretamente, a instância reconstruída pode ficar diferente da original  

---

## 19 - Após o rebuild

Após finalizar o rebuild, os serviços devem ser iniciados novamente  
A instância estará com os bancos de sistema recriados em estado inicial  
Nesse momento, se existir backup da `master`, o próximo passo é restaurá-lo  

Fluxo resumido:
1. Executar rebuild dos bancos de sistema
2. Iniciar o serviço SQL Server
3. Acessar a instância usando a conta configurada como sysadmin ou o login `sa`
4. Configurar modo single-user para restore da `master`
5. Restaurar o backup da `master`
6. Remover parâmetros especiais de inicialização
7. Reiniciar a instância normalmente
8. Validar os bancos e configurações

Se não existir backup da `master`, a instância precisará ser reconstruída manualmente  

---

## 20 - Quando não existe backup da master

Quando não existe backup válido da `master`, a recuperação se torna muito mais complexa  
Após o rebuild, a `master` estará limpa, como em uma instalação nova  
Nesse cenário, será necessário recriar manualmente os objetos e configurações em nível de instância  

Exemplos:
- Logins
- Linked servers
- Credenciais
- Endpoints
- Permissões
- Certificados
- Chaves
- Configurações de servidor
- Jobs dependentes de contexto da instância
- Operadores
- Alertas
- Configurações de Database Mail, quando aplicável

Também será necessário tornar os bancos de usuário visíveis novamente, normalmente por meio de attach ou restore  

---

## 21 - Attach dos bancos de usuário

Após rebuild sem restore da `master`, os bancos de usuário que existiam antes podem não aparecer na instância  
Isso acontece porque a nova `master` não possui os metadados desses bancos  
Nesse caso, se os arquivos `.mdf`, `.ndf` e `.ldf` ainda existirem e estiverem íntegros, pode ser necessário fazer attach dos bancos  

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

Após o attach, é necessário validar:  
- Estado do banco
- Usuários órfãos
- Permissões
- Owner do banco
- Jobs relacionados
- Linked servers usados pela aplicação
- Certificados e chaves, quando aplicável

---

## 22 - Preservando msdb e model quando possível

Se apenas a `master` foi afetada, e os arquivos da `msdb` e `model` ainda estão íntegros, pode ser útil preservar esses arquivos antes de qualquer procedimento destrutivo  

Exemplo conceitual:  
```text
Copiar msdb.mdf e msdb.ldf para local seguro
Copiar model.mdf e model.ldf para local seguro
Executar rebuild dos bancos de sistema
Avaliar restauração ou reaproveitamento conforme cenário
```

Essa ação deve ser feita com cuidado e com os serviços parados  
O objetivo é evitar perda desnecessária de metadados operacionais, como jobs, histórico e configurações armazenadas na `msdb`  
Mesmo assim, em recuperação real, a estratégia mais segura continua sendo restaurar backups válidos dos bancos de sistema  

---

## 23 - Diferença entre restore da master e rebuild

Restore da `master` e rebuild dos bancos de sistema não são a mesma coisa  

```text
Restore da master = recupera a master a partir de um backup
Rebuild           = recria master, model, msdb e tempdb em estado inicial
```

O restore preserva o estado da `master` conforme o backup utilizado  
O rebuild cria uma nova estrutura de bancos de sistema  
Depois do rebuild, se houver backup, ainda pode ser necessário restaurar a `master` e a `msdb`  

---

## 24 - Validações após recuperar a master

Após recuperar a `master`, é importante validar a instância  

Pontos de validação:  
- Serviço SQL Server inicializa normalmente
- SQL Server Agent inicializa normalmente
- Bancos de usuário aparecem corretamente
- Bancos de usuário estão online
- Logins existem
- Permissões estão corretas
- Linked servers existem
- Jobs estão disponíveis
- Certificados e chaves estão presentes
- Aplicações conseguem conectar
- SQL Server Error Log não apresenta erros críticos

Também é importante executar consultas de validação em nível de instância  

Exemplo:

```sql
SELECT
    name,
    state_desc,
    recovery_model_desc
FROM sys.databases
ORDER BY name;
GO
```

Exemplo para verificar logins:

```sql
SELECT
    name,
    type_desc,
    is_disabled
FROM sys.server_principals
ORDER BY name;
GO
```

---

## 25 - Cuidados com certificados e chaves

Se a instância utiliza recursos como TDE ou backup encryption, a recuperação da `master` precisa ser tratada com ainda mais cuidado  
Certificados e chaves podem estar armazenados ou protegidos a partir da `master`  
Sem esses objetos, alguns bancos ou backups podem não ser acessíveis  

Por isso, além do backup da `master`, também é importante manter backup de:  
- Service Master Key
- Database Master Key
- Certificados
- Private keys
- Senhas usadas na proteção das chaves

Em ambientes com TDE, perder certificados pode impedir a abertura ou restauração de bancos criptografados  

---

## 26 - Cuidados com versão e patch level

O restore da `master` deve respeitar compatibilidade de versão  
Como boa prática, a instância de destino deve estar no mesmo nível ou em nível compatível com a instância de origem  

Pontos de atenção:  
- Versão do SQL Server
- Edição do SQL Server
- Build
- Cumulative Update
- Collation
- Recursos instalados
- Caminhos dos arquivos
- Contas de serviço
- Configurações externas

Diferenças relevantes podem dificultar ou impedir a recuperação correta da instância  

---

## 27 - Fluxo resumido com backup da master

Fluxo resumido quando existe backup válido da `master` e a instância consegue subir em modo especial:  
```text
1 - Parar SQL Server Agent
2 - Adicionar -mSQLCMD nos Startup Parameters
3 - Reiniciar SQL Server
4 - Conectar via sqlcmd
5 - Executar RESTORE DATABASE master WITH REPLACE
6 - Aguardar encerramento automático da instância
7 - Remover -mSQLCMD
8 - Iniciar SQL Server normalmente
9 - Iniciar SQL Server Agent
10 - Validar a instância
```

---

## 28 - Fluxo resumido sem backup da master

Fluxo resumido quando não existe backup válido da `master`:  
```text
1 - Preservar arquivos existentes, quando possível
2 - Executar rebuild dos bancos de sistema
3 - Acessar a instância com conta sysadmin definida no rebuild
4 - Recriar configurações da instância
5 - Recriar logins
6 - Recriar linked servers
7 - Recriar certificados, chaves e credenciais
8 - Fazer attach ou restore dos bancos de usuário
9 - Corrigir usuários órfãos
10 - Recriar jobs, operadores e alertas, quando necessário
11 - Validar aplicações
12 - Criar backup atualizado da master
```

Esse cenário pode ser trabalhoso e demorado  
Por isso, o backup da `master` é indispensável  

---

## 29 - Resumo

O banco `master` é essencial para o funcionamento da instância SQL Server  
Ele armazena metadados e configurações fundamentais em nível de servidor  
Se a `master` estiver corrompida ou inacessível, a instância pode não inicializar  
A melhor estratégia de recuperação é restaurar um backup válido da `master`  
Para isso, normalmente é necessário iniciar a instância em modo single-user e conectar via `sqlcmd`  
Se a instância não inicializar, pode ser necessário executar rebuild dos bancos de sistema  
Sem backup da `master`, será necessário reconstruir manualmente logins, permissões, linked servers, certificados e demais configurações da instância  
Nunca se deve negligenciar o backup da `master`

---

## Referências

- [Restaurar o banco de dados master (Transact-SQL)](https://learn.microsoft.com/pt-br/sql/relational-databases/backup-restore/restore-the-master-database-transact-sql?view=sql-server-ver16)
- [Reconstruir bancos de dados do sistema](https://learn.microsoft.com/pt-br/sql/relational-databases/databases/rebuild-system-databases?view=sql-server-ver16)
- [Banco de dados master](https://learn.microsoft.com/pt-br/sql/relational-databases/databases/master-database?view=sql-server-ver16)
- [Opções de inicialização do serviço do Database Engine](https://learn.microsoft.com/pt-br/sql/database-engine/configure-windows/database-engine-service-startup-options?view=sql-server-ver16)