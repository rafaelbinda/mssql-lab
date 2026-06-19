# A0030 – Database Corruption and DBCC CHECKDB

> **Author:** Rafael Binda  
> **Created:** 2026-04-26  
> **Version:** 1.0  

---

## Descrição

Este documento apresenta os principais conceitos relacionados à corrupção de bancos de dados no SQL Server, incluindo causas comuns, detecção de páginas corrompidas, uso da propriedade `PAGE_VERIFY`, consulta à tabela `msdb.dbo.suspect_pages` e execução do comando `DBCC CHECKDB`

Também são abordadas as opções de reparo disponíveis, seus riscos, a importância de manter uma estratégia consistente de backup, retenção e verificação de integridade, além de uma introdução aos comandos `DBCC IND`, `DBCC PAGE` e aos parâmetros de inicialização `-f` e `-m`

---

## Hands-on

[Q0027 - Data Page Corruption and CHECKDB Repair](../scripts/Q0027-sql-data-page-corruption-and-checkdb-repair.sql)  
[Q0030 - Index Corruption and CHECKDB](../scripts/Q0030-sql-index-corruption-and-checkdb.sql)  
[INST-Q0019 - Active Suspect Pages Overview](../../../dba-scripts/SQL-instance-information/INST-Q0019-suspect-pages-active-overview.sql)  
[INST-Q0020 - Suspect Pages History Overview](../../../dba-scripts/SQL-instance-information/INST-Q0020-suspect-pages-history-overview.sql)

---

## 1 - O que é corrupção de banco de dados

Um banco de dados pode ser considerado corrompido quando o conteúdo de uma ou mais páginas apresenta problemas de integridade  
No SQL Server, os dados são armazenados em páginas de **8 KB**  
Essas páginas podem conter dados, índices ou informações internas de alocação  
Quando uma página apresenta inconsistência física ou lógica, o SQL Server pode não conseguir ler corretamente seu conteúdo  

Exemplos de corrupção incluem:
- Página com conteúdo inconsistente
- Página com checksum inválido
- Página com estrutura interna inválida
- Página de dados danificada
- Página de índice danificada
- Página de alocação danificada

---

## 2 - Por que um banco de dados corrompe

A corrupção normalmente não acontece por causa de uma instrução `SELECT`, `INSERT`, `UPDATE` ou `DELETE` comum  
Na maioria dos casos, ela está relacionada a problemas fora da lógica normal do banco de dados  

Exemplos de causas possíveis:  
- Falha em memória RAM
- Falha em cache de disco
- Problema físico em disco ou storage
- Falhas de firmware
- Falhas de driver
- Problemas no subsistema de I/O
- Desligamentos inesperados
- Falhas de software
- Escritas incompletas em disco
- Problemas em controladoras de armazenamento

Por isso, **quando existe corrupção, não basta apenas tentar reparar o banco, também é necessário investigar a causa raiz do problema**

---

## 3 - Páginas de 8 KB

O SQL Server organiza os dados em páginas de 8 KB  
Cada página possui um cabeçalho com metadados internos, como:  
- Identificação da página
- Tipo da página
- Banco de dados
- Arquivo
- Número da página
- Informações de alocação
- Informações de validação

A corrupção pode ocorrer em diferentes tipos de página  
Exemplos:  
- Página de dados
- Página de índice
- Página de alocação
- Página de controle interno

---

## 4 - Páginas de dados, índice e alocação

Nem toda página armazena linha de tabela diretamente  
Algumas páginas armazenam dados, outras armazenam índices e outras armazenam estruturas internas de controle  
Principais tipos de página para estudo:  
```text
PageType 1  = Data Page
PageType 2  = Index Page
PageType 10 = IAM Page
```

### Data Page

A página de dados armazena as linhas da tabela  
Em uma tabela heap, as linhas ficam em páginas de dados sem uma estrutura clustered organizada  
Em uma tabela com clustered index, as páginas de dados representam o nível folha do índice clustered  

### Index Page

A página de índice armazena estruturas de navegação do índice  
Ela pode fazer parte de um índice clustered ou nonclustered  
Em um índice nonclustered, as páginas de índice armazenam chaves e ponteiros para localização dos dados  
Se a corrupção estiver somente em página de índice nonclustered, pode existir cenário onde um rebuild do índice resolva o problema sem perda dos dados base, mesmo assim, a avaliação deve ser feita com cuidado

### Allocation Page

As páginas de alocação controlam quais páginas e extents pertencem aos objetos do banco  

Exemplos importantes:
```text
IAM  = Index Allocation Map
PFS  = Page Free Space
GAM  = Global Allocation Map
SGAM = Shared Global Allocation Map
DCM  = Differential Changed Map
BCM  = Bulk Changed Map
```

Corrupção em página de alocação pode ser mais sensível, porque essas páginas participam do controle interno de espaço e organização dos objetos

---

## 5 - PAGE_VERIFY

A propriedade `PAGE_VERIFY` é fundamental para ajudar o SQL Server a detectar corrupção de página  
Ela define como o SQL Server valida a integridade das páginas gravadas em disco  

As principais opções são:
- `CHECKSUM`
- `TORN_PAGE_DETECTION`
- `NONE`

---

## 6 - PAGE_VERIFY CHECKSUM

A opção `CHECKSUM` é a opção recomendada e mais utilizada atualmente  
Quando `CHECKSUM` está habilitado, o SQL Server calcula um valor de verificação para a página no momento em que ela é gravada em disco  
Esse valor fica armazenado no cabeçalho da página  
Quando a página é lida novamente, o SQL Server recalcula o checksum e compara com o valor armazenado  
Se os valores forem diferentes, isso indica que a página pode ter sido alterada ou danificada fora do controle esperado do SQL Server  

Exemplo de consulta para verificar a configuração:
```sql
SELECT
    name AS database_name,
    page_verify_option_desc
FROM sys.databases
ORDER BY name;
GO
```

Exemplo para configurar `CHECKSUM`:
```sql
ALTER DATABASE AdventureWorks
SET PAGE_VERIFY CHECKSUM;
GO
```

---

## 7 - PAGE_VERIFY, TORN_PAGE_DETECTION e NONE

A opção `TORN_PAGE_DETECTION` é mais antiga e foi mantida por compatibilidade  
Ela era usada principalmente em versões antigas do SQL Server  
Essa opção ajuda a identificar páginas parcialmente gravadas, conhecidas como torn pages  
Atualmente, `CHECKSUM` é a opção mais indicada  
A opção `NONE` desabilita a verificação de página  
Essa configuração não é recomendada para bancos de dados de produção  
Quando `PAGE_VERIFY` está como `NONE`, o SQL Server perde uma camada importante de detecção de corrupção  

Exemplo de consulta para identificar bancos sem `CHECKSUM`:
```sql
SELECT
    name AS database_name,
    page_verify_option_desc
FROM sys.databases
WHERE page_verify_option_desc <> 'CHECKSUM'
ORDER BY name;
GO
```

---

## 8 - Corrupção pode demorar para ser identificada

Um ponto importante sobre corrupção é que ela pode existir por muito tempo sem ser percebida  
Podem se passar dias, meses ou até anos sem que a página corrompida seja acessada, portanto, a corrupção pode ser percebida somente quando:  
- Alguém executa um `SELECT` que acessa aquela página
- Um processo tenta atualizar dados naquela página
- Um índice que utiliza aquela página é acessado
- Um backup com verificação detecta inconsistência
- O `DBCC CHECKDB` é executado
- Uma rotina de manutenção acessa a estrutura danificada

Por isso, não é seguro assumir que o banco está íntegro apenas porque a aplicação continua funcionando  
A aplicação pode simplesmente não ter acessado ainda a área corrompida

### Erros 823, 824 e 825

Alguns erros importantes relacionados a I/O e integridade são:  
```text
823 = erro físico ou erro reportado pelo sistema operacional durante operação de I/O
824 = erro lógico de consistência detectado pelo SQL Server após leitura ou escrita de página
825 = leitura que falhou inicialmente, mas teve sucesso após novas tentativas
```

O erro `824` normalmente indica que o Windows informou que a página foi lida com sucesso, mas o SQL Server encontrou algo inconsistente no conteúdo da página  

Exemplos de inconsistência:
- Checksum inválido
- Torn page
- Page ID incorreto
- Short transfer
- Stale read
- Page audit failure

O erro `823` normalmente aponta para uma falha mais direta no caminho de I/O, como erro retornado pelo sistema operacional, disco, storage, driver ou controladora

O erro `825` é um sinal de alerta importante, porque indica que uma leitura precisou ser repetida até funcionar

Mesmo que a query não falhe, esse erro pode indicar problema intermitente no subsistema de I/O

É recomendado configurar alertas para os erros:  
- `823`
- `824`
- `825`

Quando esses erros aparecem, também é importante verificar:  
- SQL Server Error Log
- Windows Event Viewer
- Saúde do storage
- Drivers
- Firmware
- Controladoras
- Tabela `msdb.dbo.suspect_pages`

---

## 9 - msdb.dbo.suspect_pages

Quando o SQL Server identifica uma página suspeita, ele pode registrar essa informação na tabela `msdb.dbo.suspect_pages`  
Essa tabela fica no banco de sistema `msdb`  
Ela armazena informações sobre páginas que apresentaram falhas de leitura, checksum, torn page ou outros problemas relacionados  

Exemplo de consulta:
```sql
SELECT
    DB_NAME(database_id) AS database_name,
    file_id,
    page_id,
    event_type,
    error_count,
    last_update_date
FROM msdb.dbo.suspect_pages
ORDER BY last_update_date DESC;
GO
```

A coluna `event_type` ajuda a identificar o tipo de evento registrado  

Exemplos comuns:
- Erro 823
- Erro 824
- Bad checksum
- Torn page
- Página restaurada
- Página reparada

Essa tabela é útil para investigação, mas não substitui o `DBCC CHECKDB`

---

## 10 - DBCC CHECKDB

O comando `DBCC CHECKDB` verifica a integridade lógica e física de todos os objetos de um banco de dados  
Ele é uma das principais ferramentas para identificar corrupção no SQL Server  

Exemplo básico:
```sql
DBCC CHECKDB (AdventureWorks);
GO
```

Exemplo com menos mensagens informativas:  
```sql
DBCC CHECKDB (AdventureWorks) WITH NO_INFOMSGS;
GO
```

Exemplo retornando resultado em formato tabular:  
```sql
DBCC CHECKDB (AdventureWorks) WITH NO_INFOMSGS, TABLERESULTS;
GO
```

A opção `NO_INFOMSGS` inibe mensagens informativas que normalmente não são relevantes para a investigação  
A opção `TABLERESULTS` retorna o resultado em formato de tabela, o que pode ajudar na análise ou registro do resultado

---

## 11 - Frequência do DBCC CHECKDB

O `DBCC CHECKDB` deve ser executado periodicamente

A frequência ideal depende de fatores como:  
- Tamanho do banco
- Janela de manutenção disponível
- Criticidade do ambiente
- Volume de alterações
- Política de backup
- Tempo de retenção dos backups

Um ponto fundamental é que **a execução do `DBCC CHECKDB` deve ocorrer dentro do período de retenção dos backups**

Exemplo:  
Se a empresa mantém backups por 14 dias, não faz sentido executar `DBCC CHECKDB` somente a cada 30 dias  
Nesse cenário, uma corrupção poderia existir há mais tempo do que o período de retenção disponível  
Quando isso acontece, pode não existir mais um backup íntegro anterior à corrupção

---

## 12 - Retenção de backup x DBCC CHECKDB

A estratégia de integridade precisa estar alinhada com a retenção dos backups

Exemplo de cenário inadequado:  
- Backup retido por 7 dias
- `DBCC CHECKDB` executado uma vez por mês

Nesse caso, se o `DBCC CHECKDB` identificar hoje uma corrupção que já existia há 20 dias, os backups íntegros anteriores à corrupção podem não existir mais

Exemplo de cenário mais adequado:  
- Backup retido por 30 dias
- `DBCC CHECKDB` executado semanalmente

Nesse cenário, a chance de identificar corrupção enquanto ainda existe backup íntegro disponível é maior  
O ponto principal é:  
O `DBCC CHECKDB` precisa ser executado com frequência suficiente para permitir recuperação usando backups ainda disponíveis

---

## 13 - Como resolver corrupção

A melhor forma de resolver corrupção é restaurar o banco a partir de um backup íntegro  
Quando o banco usa recovery model `FULL` ou `BULK_LOGGED`, a primeira ação antes do restore normalmente deve ser tentar executar um **tail-log backup**  
O tail-log backup tenta capturar os registros do transaction log que ainda não foram copiados em backup, isso ajuda a reduzir perda de dados e manter a sequência da cadeia de logs  

A ordem preferencial de recuperação normalmente deve ser:
1. Executar tail-log backup, quando aplicável
2. Restaurar backup full íntegro
3. Aplicar backup diferencial, se existir
4. Aplicar backups de log na sequência correta, se o banco estiver em recovery model `FULL` ou `BULK_LOGGED`
5. Avaliar restore de página, quando aplicável
6. Usar opções de reparo do `DBCC CHECKDB` apenas como último recurso

Exemplo conceitual de tail-log backup para banco online antes do restore:  
```sql
BACKUP LOG AdventureWorks
TO DISK = 'C:\Backups\AdventureWorks_TailLog.trn'
WITH NORECOVERY;
GO
```

Exemplo conceitual de tail-log backup para banco danificado:  
```sql
BACKUP LOG AdventureWorks
TO DISK = 'C:\Backups\AdventureWorks_TailLog.trn'
WITH CONTINUE_AFTER_ERROR;
GO
```

O reparo via `DBCC CHECKDB` não deve ser a primeira opção  
Ele deve ser considerado apenas quando não existe alternativa melhor de recuperação  

---

## 14 - O risco de não detectar corrupção rapidamente

Se uma página corrompida não for detectada rapidamente, pode ocorrer perda da janela de recuperação

Exemplo:  
- A página corrompe hoje
- Ninguém acessa essa página por 60 dias
- A retenção de backup é de 30 dias
- O problema só é identificado depois de 60 dias

Nesse cenário, provavelmente não existe mais backup íntegro anterior à corrupção  
Se não houver backup válido, a alternativa pode ser tentar reparo com `DBCC CHECKDB`  
Nesse caso, o resultado é incerto  
Dependendo do tipo de corrupção, pode haver perda de dados

---

## 15 - Opções de reparo do DBCC CHECKDB

O `DBCC CHECKDB` possui opções de reparo  

As principais são:
- `REPAIR_REBUILD`
- `REPAIR_ALLOW_DATA_LOSS`
- `REPAIR_FAST`

Essas opções devem ser usadas com muito cuidado  
Para executar reparo, o banco precisa estar em modo `SINGLE_USER`

---

## 16 - REPAIR_REBUILD

A opção `REPAIR_REBUILD` executa reparos que não envolvem perda de dados  
Ela pode corrigir alguns tipos de problemas, principalmente relacionados a estruturas que podem ser reconstruídas  

Exemplo conceitual:
```sql
ALTER DATABASE AdventureWorks SET SINGLE_USER WITH ROLLBACK IMMEDIATE;
GO

DBCC CHECKDB (AdventureWorks, REPAIR_REBUILD) WITH NO_INFOMSGS;
GO

ALTER DATABASE AdventureWorks SET MULTI_USER;
GO
```

Mesmo sendo uma opção mais segura que `REPAIR_ALLOW_DATA_LOSS`, ela ainda deve ser usada com cautela  
O ideal continua sendo recuperar a partir de backup quando possível

---

## 17 - REPAIR_ALLOW_DATA_LOSS

A opção `REPAIR_ALLOW_DATA_LOSS` tenta reparar todos os erros encontrados, mas pode causar perda de dados  
Essa opção deve ser tratada como último recurso  

Exemplo conceitual:
```sql
ALTER DATABASE AdventureWorks SET SINGLE_USER WITH ROLLBACK IMMEDIATE;
GO

DBCC CHECKDB (AdventureWorks, REPAIR_ALLOW_DATA_LOSS) WITH NO_INFOMSGS;
GO

ALTER DATABASE AdventureWorks SET MULTI_USER;
GO
```

O próprio nome da opção já deixa claro o risco `ALLOW_DATA_LOSS` significa permitir perda de dados  
Isso não quer dizer que sempre haverá perda de dados, mas significa que o SQL Server está autorizado a tomar ações que podem remover dados ou estruturas danificadas para devolver consistência ao banco  

---

## 18 - REPAIR_FAST

A opção `REPAIR_FAST` é mantida apenas por compatibilidade  
Ela não executa ações de reparo  
Não deve ser considerada como uma estratégia real de correção de corrupção

---

## 19 - Nível mínimo de reparo indicado pelo DBCC CHECKDB

Quando o `DBCC CHECKDB` encontra corrupção, ele pode indicar o nível mínimo de reparo necessário  

Exemplo:
Se o resultado indicar `REPAIR_REBUILD`, significa que o problema pode ser corrigido sem perda de dados  
Se o resultado indicar `REPAIR_ALLOW_DATA_LOSS`, significa que o reparo pode envolver perda de dados  
É importante avaliar com cuidado a mensagem retornada pelo `DBCC CHECKDB`  
O fato de o SQL Server indicar `REPAIR_ALLOW_DATA_LOSS` não significa necessariamente que haverá perda de linhas de negócio  
Em alguns casos, a corrupção pode estar em um índice nonclustered, e o reparo pode acabar reconstruindo ou removendo uma estrutura derivada dos dados base  
Mesmo assim, o risco existe  
Por isso, essa opção deve ser usada somente quando não houver alternativa melhor  

---

## 20 - Backup é a principal estratégia de recuperação

O ideal é ter backup  
O `DBCC CHECKDB` ajuda a identificar problemas de integridade  
Mas ele não substitui backup  

A estratégia correta envolve:  
- Backup full
- Backup diferencial, quando aplicável
- Backup de log, quando o recovery model permitir
- Retenção adequada
- Teste de restore
- Execução periódica de `DBCC CHECKDB`
- Monitoramento de erros críticos
- Investigação de causa raiz

Nunca se deve tratar o `DBCC CHECKDB` como única estratégia de recuperação de banco

---

## 21 - CHECKDB não corrige a causa raiz

O `DBCC CHECKDB` pode identificar corrupção  
Em alguns casos, pode reparar estruturas  
Mas ele não resolve a causa raiz do problema  
Se a corrupção foi causada por storage, memória, firmware, driver ou controladora, o problema pode voltar a ocorrer  

Após identificar corrupção, também é necessário avaliar:  
- Logs do SQL Server
- Event Viewer do Windows
- Alertas de storage
- Saúde dos discos
- Firmware
- Drivers
- Controladoras
- Memória
- Antivírus ou filtros de I/O
- Histórico de quedas ou desligamentos inesperados

Corrigir apenas o banco pode não ser suficiente

---

## 22 - Fluxo recomendado de investigação

Um fluxo básico de investigação pode seguir esta ordem:  
1. Confirmar o erro reportado
2. Verificar o SQL Server Error Log
3. Verificar o Windows Event Viewer
4. Consultar `msdb.dbo.suspect_pages`
5. Executar `DBCC CHECKDB`
6. Avaliar o tipo de corrupção
7. Verificar se existe backup íntegro
8. Avaliar tail-log backup, quando aplicável
9. Avaliar restore em ambiente separado
10. Definir estratégia de recuperação
11. Investigar causa raiz

Exemplo de consulta em `suspect_pages`:  
```sql
SELECT
    DB_NAME(database_id) AS database_name,
    file_id,
    page_id,
    event_type,
    error_count,
    last_update_date
FROM msdb.dbo.suspect_pages
ORDER BY last_update_date DESC;
GO
```

Exemplo de execução do `CHECKDB`:
```sql
DBCC CHECKDB (AdventureWorks) WITH NO_INFOMSGS, TABLERESULTS;
GO
```

---

## 23 - DBCC IND

O comando `DBCC IND` lista as páginas associadas a um objeto dentro do banco de dados

Exemplo:
```sql
DBCC IND ('AdventureWorks', 'Person.Person', -1);
GO
```

O terceiro parâmetro representa o índice que será analisado

Exemplos:
- `-1` lista todas as páginas da tabela e dos índices
- `0` lista páginas do heap, quando existir
- `1` lista páginas do índice clustered, quando existir
- Outros valores representam índices específicos

O `DBCC IND` é útil para estudos internos sobre páginas, índices e estruturas de armazenamento

---

## 24 - Principais colunas do DBCC IND

Algumas colunas importantes retornadas pelo `DBCC IND`:
- `PageFID` identifica o arquivo físico onde a página está armazenada
- `PagePID` identifica o número da página dentro do arquivo
- `IAMFID` identifica o arquivo onde está a página IAM relacionada
- `IAMPID` identifica o número da página IAM relacionada
- `ObjectID` identifica o objeto ao qual a página pertence
- `IndexID` identifica o índice relacionado à página
- `PartitionNumber` identifica a partição relacionada à página
- `PageType` identifica o tipo da página retornada

A combinação `PageFID` e `PagePID` representa o endereço físico da página

Exemplo:
```text
PageFID = 1
PagePID = 264
```

Esse endereço representa a página `1:264`

---

## 25 - Page types

Alguns tipos de página importantes:
```text
PageType 1  = Data Page
PageType 2  = Index Page
PageType 10 = IAM Page
```

A página de dados armazena linhas da tabela  
A página de índice armazena estruturas de índice  
A página IAM - *Index Allocation Map* controla informações de alocação

---

## 26 - Exemplo mental sobre IAM Page

Uma forma simples de pensar:
```text
IAM Page           = mapa que mostra onde estão as páginas/extents de um objeto
PageFID e PagePID  = endereço físico da página encontrada
IAMFID e IAMPID    = endereço físico da página IAM que mapeia aquela alocação
```

A página IAM não é a página de dados em si  
Ela funciona como um mapa de alocação  
Ela ajuda o SQL Server a saber quais extents pertencem a uma allocation unit de determinado objeto  

Exemplo mental:
```text
IAM Page          = mapa do bairro
PageFID e PagePID = endereço físico da casa
IAMFID e IAMPID   = endereço físico do mapa que mostra onde a casa está
```

Esse exemplo não é uma tradução literal da arquitetura interna, mas ajuda a memorizar a relação entre a página encontrada e a página IAM usada para mapear a alocação

---

## 27 - DBCC PAGE

O comando `DBCC PAGE` permite visualizar o conteúdo interno de uma página

Exemplo:  
```sql
DBCC TRACEON (3604);
GO

DBCC PAGE (AdventureWorks, 1, 264, 3);
GO
```

Parâmetros do exemplo:  
```text
AdventureWorks = banco de dados
1              = file_id
264            = page_id
3              = nível de detalhamento da saída
```

A trace flag `3604` direciona a saída do comando para a sessão atual  
O último parâmetro do `DBCC PAGE` representa o nível de detalhamento da saída  

Exemplos comuns:  
```text
1 = Exibe o cabeçalho da página e os slots com dump em hexadecimal
2 = Exibe o cabeçalho da página e o dump completo da página
3 = Exibe o cabeçalho da página e interpreta os registros de forma mais detalhada
```

Na prática, para estudo didático, o nível `3` costuma ser o mais útil, porque facilita a leitura do conteúdo interno da página

---

## 28 - DBCC WRITEPAGE

O comando `DBCC WRITEPAGE` pode ser usado em estudos controlados para simular corrupção de página  
Esse comando não deve ser usado em produção  
Ele é útil apenas em laboratório, com bancos descartáveis, para entender como o SQL Server reage a corrupção  

Exemplo de uso conceitual:
```text
DBCC WRITEPAGE
```

Importante:  
- Não usar em ambiente produtivo
- Não usar em bancos reais
- Não usar em bancos com dados importantes
- Usar apenas em laboratório
- Usar apenas com objetivo de estudo

---

## 29 - Banco em SINGLE_USER para reparo

Para executar opções de reparo do `DBCC CHECKDB`, o banco precisa estar em modo `SINGLE_USER`

Exemplo:  
```sql
ALTER DATABASE AdventureWorks SET SINGLE_USER WITH ROLLBACK IMMEDIATE;
GO
```

Após o reparo, o banco deve retornar para `MULTI_USER`

Exemplo:  
```sql
ALTER DATABASE AdventureWorks SET MULTI_USER;
GO
```

A opção `WITH ROLLBACK IMMEDIATE` encerra conexões existentes e desfaz transações abertas para permitir a alteração do modo de acesso  
**Essa ação deve ser usada com cuidado em produção**

---

## 30 - Parâmetros de inicialização -f e -m

Em alguns cenários de recuperação, pode ser necessário iniciar a instância SQL Server com parâmetros especiais  
Esses parâmetros são configurados no SQL Server Configuration Manager  

Caminho conceitual:
```text
SQL Server Configuration Manager -> SQL Server Services -> SQL Server (MSSQLSERVER) -> Properties -> Startup Parameters
```

Principais parâmetros:
```text
-m = inicia a instância em modo single-user
-f = inicia a instância em configuração mínima
```

### Parâmetro -m

O parâmetro `-m` inicia a instância em modo single-user  
Ele pode ser usado quando é necessário obter acesso exclusivo à instância para alguma ação administrativa ou de recuperação  

Exemplo de uso comum:  
- Restaurar o banco `master`
- Executar manutenção emergencial
- Corrigir problema de acesso administrativo
- Evitar conexões concorrentes durante procedimento crítico

Ao usar `-m`, é importante parar o SQL Server Agent antes de subir a instância  
O SQL Server Agent pode consumir a única conexão disponível em modo single-user  

### Parâmetro -f

O parâmetro `-f` inicia a instância com configuração mínima  
Ele pode ser útil quando uma configuração impede a inicialização normal do SQL Server  

Exemplos de uso comum:
- Configuração incorreta de memória
- Problema em parâmetro de inicialização
- Necessidade de subir a instância de forma mínima para ajuste emergencial

### Atenção

Os parâmetros `-f` e `-m` **afetam a instância SQL Server**  
Eles não são a mesma coisa que colocar apenas um banco de dados em `SINGLE_USER`  
Para reparo com `DBCC CHECKDB`, normalmente basta colocar o banco afetado em `SINGLE_USER`  
Para cenários em que a instância não inicializa corretamente, os parâmetros `-f` e `-m` podem ser necessários  
Após o procedimento, os parâmetros devem ser removidos e o serviço SQL Server deve ser reiniciado normalmente  

---

## 31 - Sempre executar CHECKDB novamente após reparo

Depois de qualquer tentativa de reparo, é importante executar o `DBCC CHECKDB` novamente  

Exemplo:  
```sql
DBCC CHECKDB (AdventureWorks) WITH NO_INFOMSGS, TABLERESULTS;
GO
```

Isso confirma se ainda existem inconsistências pendentes

O fluxo deve ser:  
1. Executar `DBCC CHECKDB`
2. Avaliar o erro
3. Definir estratégia de recuperação
4. Executar restore ou reparo
5. Executar `DBCC CHECKDB` novamente
6. Confirmar que o banco está íntegro

---

## 32 - Estratégia ideal

A estratégia ideal para lidar com corrupção envolve prevenção, detecção e recuperação

### Prevenção
- Storage confiável
- Memória saudável
- Drivers e firmware atualizados
- Monitoramento do Windows Event Viewer
- Monitoramento do SQL Server Error Log
- Alertas para erros críticos

### Detecção
- `PAGE_VERIFY CHECKSUM`
- Alertas para erros 823, 824 e 825
- Consulta à `msdb.dbo.suspect_pages`
- Execução periódica de `DBCC CHECKDB`

### Recuperação
- Tail-log backup, quando aplicável
- Backups válidos
- Retenção adequada
- Teste de restore
- Restore em ambiente separado
- Restore de página, quando aplicável
- Reparo com `DBCC CHECKDB` apenas como último recurso

---

## 33 - Pontos de atenção para ambientes reais

Em ambiente real, ao identificar corrupção, **evite agir de forma precipitada**  

Antes de executar qualquer reparo destrutivo:  
- Avalie o erro com calma
- Preserve evidências
- Verifique backups disponíveis
- Teste restore em outro ambiente
- Consulte o SQL Server Error Log
- Consulte o Event Viewer
- Avalie o storage
- Verifique se há outras bases afetadas
- Evite executar `REPAIR_ALLOW_DATA_LOSS` diretamente em produção

O comando `REPAIR_ALLOW_DATA_LOSS` pode resolver a consistência estrutural do banco, mas pode remover dados  
A recuperação via backup normalmente é mais segura  

---

## 34 - Resumo

Corrupção de banco de dados é um problema crítico e deve ser tratado com prioridade  
O SQL Server possui recursos importantes para identificação e investigação, como:  
- `PAGE_VERIFY`
- `msdb.dbo.suspect_pages`
- `DBCC CHECKDB`
- `DBCC IND`
- `DBCC PAGE`

A melhor estratégia de recuperação continua sendo backup válido e testado  
O `DBCC CHECKDB` deve fazer parte da rotina de manutenção, mas não deve ser tratado como substituto de backup  

Opções como `REPAIR_REBUILD` e `REPAIR_ALLOW_DATA_LOSS` devem ser usadas com cuidado e, preferencialmente, apenas quando não existir alternativa melhor

---

## Referências

- [DBCC CHECKDB (Transact-SQL)](https://learn.microsoft.com/pt-br/sql/t-sql/database-console-commands/dbcc-checkdb-transact-sql?view=sql-server-ver16)
- [suspect_pages (Transact-SQL)](https://learn.microsoft.com/pt-br/sql/relational-databases/system-tables/suspect-pages-transact-sql?view=sql-server-ver16)
- [ALTER DATABASE SET options (Transact-SQL) – PAGE_VERIFY](https://learn.microsoft.com/pt-br/sql/t-sql/statements/alter-database-transact-sql-set-options?view=sql-server-ver16)
- [Erro do MSSQLSERVER 824](https://learn.microsoft.com/pt-br/sql/relational-databases/errors-events/mssqlserver-824-database-engine-error?view=sql-server-ver16)
- [Opções de inicialização do serviço do Database Engine](https://learn.microsoft.com/pt-br/sql/database-engine/configure-windows/database-engine-service-startup-options?view=sql-server-ver16)

---
