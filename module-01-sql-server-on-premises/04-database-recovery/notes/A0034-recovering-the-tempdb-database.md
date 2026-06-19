# A0034 – Recovering the tempdb Database

> **Author:** Rafael Binda  
> **Created:** 2026-04-28  
> **Version:** 1.0  

---

## Descrição

Este documento apresenta os principais conceitos relacionados à recuperação do banco de sistema `tempdb` no SQL Server

A `tempdb` é diferente dos demais bancos de sistema porque é recriada toda vez que a instância SQL Server é inicializada

Por esse motivo, a recuperação da `tempdb` normalmente não envolve restore de backup, mas sim correção de problemas que impedem sua recriação, como caminhos inválidos, volumes indisponíveis, arquivos inacessíveis ou configurações incorretas

---

## 1 - O que é o banco tempdb

O banco `tempdb` é um banco de sistema utilizado como área temporária pela instância SQL Server  
Ele é compartilhado por todos os usuários e processos conectados à instância  
A `tempdb` é usada para armazenar objetos temporários, estruturas internas e operações intermediárias executadas pelo SQL Server  

Exemplos de uso da `tempdb`:  
- Tabelas temporárias locais
- Tabelas temporárias globais
- Variáveis de tabela
- Worktables
- Workfiles
- Operações de sort
- Operações de hash
- Cursors
- Version store
- Operações de índice com `SORT_IN_TEMPDB`
- Objetos internos usados pelo otimizador e executor de queries

---

## 2 - Por que a tempdb é diferente

A `tempdb` é recriada toda vez que o SQL Server Database Engine é iniciado  
Isso significa que os objetos existentes na `tempdb` não são preservados entre reinicializações  
Diferente de bancos de usuário, a `tempdb` não possui uma estratégia tradicional de backup e restore  

Pontos importantes:  
- A `tempdb` é recriada no startup da instância
- O conteúdo anterior da `tempdb` é descartado
- Objetos temporários não são persistidos após restart
- Backup da `tempdb` não é permitido
- Restore da `tempdb` não é permitido
- O recovery model da `tempdb` é `SIMPLE`
- A `tempdb` não deve ser tratada como banco persistente

Por isso, quando existe problema com a `tempdb`, o foco geralmente é permitir que a instância consiga recriá-la corretamente durante a inicialização  

---

## 3 - Relação entre model e tempdb

Durante a inicialização da instância, o SQL Server utiliza a `model` como base para recriar a `tempdb`  
Por esse motivo, a `model` precisa estar operacional para que a `tempdb` seja criada corretamente  
Se a `model` estiver corrompida ou indisponível, a instância pode falhar antes de conseguir recriar a `tempdb`  

Relação conceitual:  
```text
model  = modelo usado para criação de novos bancos
tempdb = banco temporário recriado a cada inicialização
```

Mesmo que a `tempdb` seja o banco que aparece na mensagem de erro, a causa pode estar relacionada à `model`, ao caminho configurado, ao volume ou às permissões  

---

## 4 - Quando a tempdb pode impedir a inicialização

A `tempdb` pode impedir a inicialização do SQL Server quando seus arquivos não conseguem ser criados ou acessados  

Isso pode ocorrer em cenários como:  
- Volume da `tempdb` indisponível
- Diretório da `tempdb` removido
- Caminho da `tempdb` incorreto
- Permissão insuficiente para a conta de serviço
- Falha no storage
- Partição corrompida
- Disco sem espaço
- Arquivo travado por antivírus ou outro processo
- Configuração incorreta de arquivo
- Problema após tentativa de mover a `tempdb`

Exemplo conceitual:  
```text
A tempdb estava configurada no volume E:
O volume E: deixou de existir
Durante o startup, o SQL Server tenta criar a tempdb em E:
Como o caminho não existe, o serviço não inicializa
```

---

## 5 - Como identificar o problema

Quando o SQL Server não inicializa, é necessário verificar os logs do sistema operacional e do SQL Server  
No Windows, um dos principais locais para iniciar a investigação é o Event Viewer  

Caminho conceitual:  
```text
Windows -> Administrative Tools -> Event Viewer -> Windows Logs -> Application
```

É comum encontrar eventos de erro indicando falha ao abrir, criar ou acessar arquivos da `tempdb`  

Também é importante verificar:
- SQL Server Error Log
- Windows Event Viewer
- Mensagens do serviço SQL Server
- Caminho dos arquivos configurados
- Existência do diretório
- Permissões da conta de serviço
- Espaço disponível em disco
- Estado do volume ou storage

Mensagens relacionadas a falhas na abertura de arquivos durante o startup podem indicar problema de caminho, arquivo, permissão ou storage  

---

## 6 - Verificando a configuração atual da tempdb

Quando a instância ainda consegue inicializar, é possível verificar os arquivos da `tempdb` via T-SQL  

Exemplo:  
```sql
USE tempdb;
GO

SELECT
    file_id,
    type_desc,
    name AS logical_name,
    physical_name,
    size,
    growth,
    is_percent_growth
FROM sys.database_files
ORDER BY file_id;
GO
```

Também é possível usar:  
```sql
EXEC sp_helpdb 'tempdb';
GO
```

Essas informações ajudam a identificar:  
- Nome lógico do arquivo
- Caminho físico atual
- Nome físico do arquivo
- Tamanho configurado
- Crescimento configurado

---

## 7 - Logical name, path e filename

Para mover ou corrigir os arquivos da `tempdb`, é importante entender a diferença entre nome lógico e caminho físico  

Exemplo:  
```text
logical name = tempdev
path         = C:\MSSQL_TEMPDB\
filename     = tempdb.mdf
```

No comando `ALTER DATABASE ... MODIFY FILE`, o parâmetro `NAME` recebe o nome lógico do arquivo  
O parâmetro `FILENAME` recebe o novo caminho físico completo do arquivo  

Exemplo conceitual:  
```sql
ALTER DATABASE tempdb MODIFY FILE
(
    NAME = tempdev,
    FILENAME = 'C:\MSSQL_TEMPDB\tempdb.mdf'
);
GO
```

Nesse exemplo:  
```text
tempdev                         = nome lógico do arquivo
C:\MSSQL_TEMPDB\tempdb.mdf      = novo caminho físico completo
```

---

## 8 - Movendo arquivos da tempdb com a instância online

Quando a instância ainda inicializa, a correção do caminho da `tempdb` pode ser feita com `ALTER DATABASE ... MODIFY FILE`  

Exemplo para o arquivo de dados:  
```sql
ALTER DATABASE tempdb MODIFY FILE
(
    NAME = tempdev,
    FILENAME = 'C:\MSSQL_TEMPDB\tempdb.mdf'
);
GO
```

Exemplo para o arquivo de log:  
```sql
ALTER DATABASE tempdb MODIFY FILE
(
    NAME = templog,
    FILENAME = 'C:\MSSQL_TEMPDB\templog.ldf'
);
GO
```

Após alterar o caminho, é necessário reiniciar o serviço SQL Server  
Na próxima inicialização, o SQL Server tentará criar os arquivos da `tempdb` no novo local configurado  

---

## 9 - Criando o novo diretório

Antes de reiniciar a instância, o novo diretório precisa existir  
A conta de serviço do SQL Server precisa ter permissão para criar e acessar os arquivos no novo local  

Exemplo:  
```text
C:\MSSQL_TEMPDB\
```

Pontos de validação:  
- Diretório existe
- Conta de serviço tem permissão
- Volume tem espaço disponível
- Caminho está correto
- Antivírus não está bloqueando o diretório
- Storage está saudável

Se o diretório não existir ou a conta de serviço não tiver permissão, a instância pode falhar novamente na inicialização  

---

## 10 - Simulando perda de volume da tempdb

Em laboratório, uma simulação comum é alterar ou remover o caminho onde a `tempdb` estava configurada  

Exemplo conceitual:  
```text
1 - Parar o serviço SQL Server
2 - Renomear o diretório atual da tempdb
3 - Tentar iniciar o serviço SQL Server
4 - Validar que a instância não inicializa
5 - Verificar o erro no Event Viewer
6 - Subir a instância com parâmetros especiais
7 - Corrigir o caminho da tempdb
8 - Reiniciar a instância normalmente
```

Esse tipo de teste deve ser feito apenas em laboratório  
Não deve ser executado em ambiente de produção

---

## 11 - Quando a instância não inicializa

Se a instância não inicializa porque o caminho da `tempdb` está inválido, não será possível abrir o SSMS normalmente para corrigir o problema  
Nesse cenário, é necessário iniciar a instância com parâmetros especiais para permitir conexão mínima e alteração da configuração  

Parâmetros normalmente usados:  
```text
-f = configuração mínima
-m = modo single-user
```

O objetivo é subir a instância com o mínimo necessário para conseguir conectar via `sqlcmd` e alterar o caminho da `tempdb`  

---

## 12 - Parâmetro -f

O parâmetro `-f` inicia a instância em configuração mínima  
Ele pode ser usado quando alguma configuração impede a inicialização normal do SQL Server  
Em cenários de problema com `tempdb`, ele pode permitir que a instância suba o suficiente para corrigir a configuração dos arquivos  

Exemplos de uso do `-f`:  

- Corrigir caminho inválido da `tempdb`
- Corrigir configuração incorreta de memória
- Subir a instância com configuração mínima para ajuste emergencial
- Bypass temporário de algumas configurações que impedem o startup

Após corrigir o problema, o parâmetro `-f` deve ser removido  

---

## 13 - Parâmetro -m

O parâmetro `-m` inicia a instância em modo single-user  
Isso permite apenas uma conexão com a instância  
Em procedimentos emergenciais, ele ajuda a garantir uma conexão administrativa exclusiva  

Exemplo:  
```text
-mSQLCMD
```

O uso de `-mSQLCMD` restringe a conexão single-user ao `sqlcmd`, reduzindo o risco de outra aplicação consumir a única conexão disponível  

Antes de iniciar a instância com `-m`, é importante parar o SQL Server Agent  
O Agent pode consumir a única conexão disponível  

---

## 14 - Configurando -f e -m no SQL Server Configuration Manager

Caminho conceitual:  
```text
SQL Server Configuration Manager -> SQL Server Services -> SQL Server (MSSQLSERVER) -> Properties -> Startup Parameters
```

Parâmetros que podem ser adicionados temporariamente:  
```text
-f
-mSQLCMD
```

Depois de adicionar os parâmetros:  
```text
1 - Parar SQL Server Agent
2 - Parar SQL Server
3 - Iniciar SQL Server
4 - Conectar via sqlcmd
5 - Corrigir a configuração da tempdb
6 - Parar SQL Server
7 - Remover -f e -mSQLCMD
8 - Iniciar SQL Server normalmente
```

---

## 15 - Conectando via sqlcmd

Com a instância iniciada em modo especial, a conexão deve ser feita via `sqlcmd`  

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

---

## 16 - Corrigindo o caminho da tempdb

Após conectar via `sqlcmd`, execute o `ALTER DATABASE ... MODIFY FILE` para apontar a `tempdb` para um caminho válido  

Exemplo para arquivo de dados:  
```sql
ALTER DATABASE tempdb MODIFY FILE
(
    NAME = tempdev,
    FILENAME = 'C:\MSSQL_TEMPDB\tempdb.mdf'
);
GO
```

Exemplo para arquivo de log:  
```sql
ALTER DATABASE tempdb MODIFY FILE
(
    NAME = templog,
    FILENAME = 'C:\MSSQL_TEMPDB\templog.ldf'
);
GO
```

Se existirem múltiplos arquivos de dados, todos os arquivos com caminho inválido devem ser corrigidos  

---

## 17 - Exemplo com múltiplos arquivos de dados

Exemplo conceitual:  
```sql
ALTER DATABASE tempdb MODIFY FILE
(
    NAME = tempdev,
    FILENAME = 'C:\MSSQL_TEMPDB\tempdb.mdf'
);
GO

ALTER DATABASE tempdb MODIFY FILE
(
    NAME = temp2,
    FILENAME = 'C:\MSSQL_TEMPDB\tempdb_mssql_2.ndf'
);
GO

ALTER DATABASE tempdb MODIFY FILE
(
    NAME = temp3,
    FILENAME = 'C:\MSSQL_TEMPDB\tempdb_mssql_3.ndf'
);
GO

ALTER DATABASE tempdb MODIFY FILE
(
    NAME = templog,
    FILENAME = 'C:\MSSQL_TEMPDB\templog.ldf'
);
GO
```

Os nomes lógicos precisam refletir os nomes existentes na instância  
Por isso, sempre que possível, consulte a configuração atual antes de alterar  

---

## 18 - Reiniciando após a correção

Depois de alterar os caminhos da `tempdb`, é necessário reiniciar o SQL Server  

Antes de iniciar normalmente:  
```text
1 - Parar o serviço SQL Server
2 - Remover os parâmetros -f e -mSQLCMD
3 - Confirmar que o diretório novo existe
4 - Confirmar permissões da conta de serviço
5 - Iniciar o serviço SQL Server normalmente
```

Na próxima inicialização, o SQL Server deve criar os arquivos da `tempdb` no novo local  

---

## 19 - Validação após a recuperação

Após a instância inicializar normalmente, valide a configuração da `tempdb`  

Exemplo:  
```sql
USE tempdb;
GO

SELECT
    file_id,
    type_desc,
    name AS logical_name,
    physical_name,
    size,
    growth,
    is_percent_growth
FROM sys.database_files
ORDER BY file_id;
GO
```

Também é importante verificar:

- SQL Server Error Log
- Windows Event Viewer
- Estado da instância
- Espaço livre no volume
- Permissões no diretório
- Se os arquivos foram criados no novo caminho
- Se aplicações conseguem conectar normalmente

---

## 20 - tempdb não tem restore tradicional

A `tempdb` não é restaurada a partir de backup  
Ela é recriada automaticamente durante o startup do SQL Server  
Por isso, comandos como backup e restore da `tempdb` não fazem parte da estratégia de recuperação  

Conceito principal:  
```text
Problema na tempdb = corrigir recriação, caminho, permissão, espaço ou configuração
Não é restore tradicional de backup
```

Se a instância não sobe por causa da `tempdb`, o objetivo é permitir que ela consiga recriar os arquivos em um local válido  

---

## 21 - Diferença entre recuperar tempdb e mover tempdb

Recuperar a `tempdb` normalmente significa corrigir um problema que impede a instância de recriá-la  
Mover a `tempdb` significa alterar intencionalmente seus arquivos para outro volume ou diretório  

Exemplo:  
```text
Mover tempdb      = ação planejada
Recuperar tempdb  = ação emergencial após falha
```

Na prática, os dois procedimentos podem usar o mesmo comando `ALTER DATABASE ... MODIFY FILE`  
A diferença está no contexto e na urgência da ação  

---

## 22 - Cuidados com espaço em disco

A falta de espaço no volume da `tempdb` pode causar falhas graves na instância ou nas queries  
A `tempdb` pode crescer rapidamente dependendo da carga de trabalho  

Exemplos de operações que podem consumir muito espaço:  
- Sorts grandes
- Hash joins
- Hash aggregates
- Version store
- Índices com `SORT_IN_TEMPDB`
- Tabelas temporárias grandes
- Cursors
- Consultas com spills
- Operações online
- Transações longas com row versioning

Por isso, o volume da `tempdb` precisa ser monitorado continuamente  

---

## 23 - Cuidados com múltiplos arquivos

Ambientes SQL Server frequentemente utilizam múltiplos arquivos de dados para `tempdb`  
Isso pode melhorar a distribuição de alocação e reduzir contenção, dependendo da carga  
Ao recuperar ou mover a `tempdb`, todos os arquivos configurados precisam ser avaliados  

Pontos importantes:  
- Todos os arquivos precisam apontar para caminhos válidos
- Os arquivos devem ter tamanhos coerentes
- O crescimento deve ser configurado de forma adequada
- A conta de serviço precisa ter permissão em todos os caminhos
- O volume precisa ter espaço suficiente
- Após restart, todos os arquivos devem ser recriados corretamente

---

## 24 - Cuidados com permissões

A conta de serviço do SQL Server precisa ter permissão no diretório da `tempdb`  
Sem permissão adequada, a instância pode falhar ao criar os arquivos  

Permissões necessárias normalmente envolvem:  
- Ler o diretório
- Criar arquivos
- Gravar arquivos
- Modificar arquivos
- Excluir arquivos

Em recuperação emergencial, permissões incorretas podem fazer parecer que o caminho está errado, quando o problema real é acesso negado  

---

## 25 - Cuidados com antivírus e filtros de I/O

Antivírus, ferramentas de backup, agentes de segurança ou filtros de I/O podem interferir nos arquivos do SQL Server  
Em ambientes reais, é importante validar se o diretório da `tempdb` está seguindo as recomendações de exclusão da solução de segurança usada pela empresa  

Pontos de atenção:  
- Bloqueio de criação de arquivo
- Delay na criação dos arquivos
- Quarentena indevida
- Varredura intensa em arquivos `.mdf`, `.ndf` e `.ldf`
- Impacto de performance
- Erros intermitentes de I/O

---

## 26 - Fluxo resumido quando a instância ainda inicializa

Fluxo para corrigir ou mover a `tempdb` quando a instância ainda sobe:  
```text
1 - Verificar arquivos atuais da tempdb
2 - Criar novo diretório
3 - Garantir permissão para a conta de serviço
4 - Executar ALTER DATABASE tempdb MODIFY FILE
5 - Reiniciar o SQL Server
6 - Validar criação dos arquivos no novo caminho
7 - Validar SQL Server Error Log
8 - Validar aplicações
```

---

## 27 - Fluxo resumido quando a instância não inicializa

Fluxo para recuperar a instância quando a `tempdb` impede o startup:  
```text
1 - Verificar Windows Event Viewer
2 - Verificar SQL Server Error Log
3 - Identificar erro de caminho, permissão, espaço ou volume
4 - Criar novo diretório válido
5 - Garantir permissão para a conta de serviço
6 - Adicionar -f e -mSQLCMD nos Startup Parameters
7 - Parar SQL Server Agent
8 - Iniciar SQL Server
9 - Conectar via sqlcmd
10 - Executar ALTER DATABASE tempdb MODIFY FILE
11 - Parar SQL Server
12 - Remover -f e -mSQLCMD
13 - Iniciar SQL Server normalmente
14 - Validar arquivos da tempdb
15 - Validar SQL Server Error Log
```

---

## 28 - Exemplo completo de recuperação

Exemplo conceitual:  
```text
Cenário:
A tempdb estava configurada em E:\MSSQL_TEMPDB\
O volume E: deixou de existir
A instância SQL Server não inicializa
```

Ação:
```text
1 - Criar novo diretório em C:\MSSQL_TEMPDB\
2 - Garantir permissão da conta de serviço
3 - Iniciar a instância com -f e -mSQLCMD
4 - Conectar via sqlcmd
5 - Alterar os arquivos da tempdb para C:\MSSQL_TEMPDB\
6 - Remover parâmetros especiais
7 - Reiniciar normalmente
```

Comandos conceituais:  
```sql
ALTER DATABASE tempdb MODIFY FILE
(
    NAME = tempdev,
    FILENAME = 'C:\MSSQL_TEMPDB\tempdb.mdf'
);
GO

ALTER DATABASE tempdb MODIFY FILE
(
    NAME = templog,
    FILENAME = 'C:\MSSQL_TEMPDB\templog.ldf'
);
GO
```

Após reiniciar, validar:  
```sql
USE tempdb;
GO

SELECT
    file_id,
    type_desc,
    name AS logical_name,
    physical_name
FROM sys.database_files
ORDER BY file_id;
GO
```

---

## 29 - O que não fazer

Em uma falha relacionada à `tempdb`, evite ações precipitadas  

Pontos de atenção:  
- Não tentar restaurar backup da `tempdb`
- Não apagar arquivos sem entender o cenário
- Não alterar caminhos sem validar permissões
- Não esquecer de remover `-f` e `-m`
- Não deixar SQL Server Agent ativo consumindo conexão single-user
- Não iniciar a instância normalmente antes de corrigir o caminho
- Não ignorar Event Viewer e SQL Server Error Log
- Não tratar falha de `tempdb` como se fosse falha comum de banco de usuário

---

## 30 - Resumo

A `tempdb` é um banco de sistema temporário recriado a cada inicialização do SQL Server  
Ela não permite backup e restore tradicional  
Quando há falha relacionada à `tempdb`, o objetivo normalmente é corrigir caminho, permissão, espaço, volume ou configuração para permitir sua recriação no startup  
Se a instância ainda inicializa, a correção pode ser feita com `ALTER DATABASE tempdb MODIFY FILE` e restart do serviço  
Se a instância não inicializa, pode ser necessário usar `-f` e `-mSQLCMD` para subir em modo especial, conectar via `sqlcmd` e corrigir o caminho da `tempdb`  
Após a recuperação, é essencial validar o SQL Server Error Log, o Event Viewer e a configuração física dos arquivos da `tempdb`

---

## Referências

- [Banco de dados tempdb](https://learn.microsoft.com/pt-br/sql/relational-databases/databases/tempdb-database?view=sql-server-ver16)
- [ALTER DATABASE (Transact-SQL) – Opções de arquivo e filegroup](https://learn.microsoft.com/pt-br/sql/t-sql/statements/alter-database-transact-sql-file-and-filegroup-options?view=sql-server-ver16)
- [Mover bancos de dados de sistema](https://learn.microsoft.com/pt-br/sql/relational-databases/databases/move-system-databases?view=sql-server-ver16)
- [Opções de inicialização do serviço do Database Engine](https://learn.microsoft.com/pt-br/sql/database-engine/configure-windows/database-engine-service-startup-options?view=sql-server-ver16)