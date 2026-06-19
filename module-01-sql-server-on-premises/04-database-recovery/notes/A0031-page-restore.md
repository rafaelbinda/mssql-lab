# A0031 – Page Restore

> **Author:** Rafael Binda  
> **Created:** 2026-04-26  
> **Version:** 1.0  

---

## Descrição

Este documento apresenta os principais conceitos relacionados ao processo de Page Restore no SQL Server

O Page Restore permite restaurar uma ou mais páginas específicas de um banco de dados, sem necessariamente restaurar o banco inteiro

Esse tipo de recuperação pode ser útil quando a corrupção está limitada a páginas específicas e existe uma cadeia de backups válida para trazer essas páginas novamente para um estado consistente

---

## Hands-on

[Q0028 - Successful Page Restore](../scripts/Q0028-sql-successful-page-restore.sql)  
[Q0029 - Successful Multiple Page Restore](../scripts/Q0029-sql-successful-multiple-page-restore.sql)  
[INST-Q0019 - Active Suspect Pages Overview](../../../dba-scripts/SQL-instance-information/INST-Q0019-suspect-pages-active-overview.sql)  
[INST-Q0020 - Suspect Pages History Overview](../../../dba-scripts/SQL-instance-information/INST-Q0020-suspect-pages-history-overview.sql)

---

## 1 - O que é Page Restore

Page Restore é o processo de restauração de uma ou mais páginas específicas de um banco de dados  
Em vez de restaurar o banco inteiro, o SQL Server permite restaurar somente as páginas danificadas  

Esse recurso é especialmente útil quando:  
- A corrupção está isolada em poucas páginas
- Existe backup válido contendo essas páginas
- Existe cadeia de log suficiente para atualizar as páginas restauradas
- O objetivo é reduzir o tempo de indisponibilidade
- O banco é grande e restaurar tudo seria muito custoso

Exemplo conceitual:
```text
Banco inteiro        = 1.7 TB
Página corrompida    = 8 KB
Estratégia possível  = restaurar somente a página afetada
```

---

## 2 - Por que Page Restore pode ser útil

Quando uma página corrompe, uma alternativa seria restaurar o banco inteiro  
Porém, em bancos grandes, isso pode ser inviável ou demorado  

Exemplo:
```text
Banco de dados      = 2 TB
Restore completo    = várias horas
Página afetada      = poucas páginas de 8 KB
```

Nesse cenário, se a corrupção estiver limitada a poucas páginas, o Page Restore pode ser uma alternativa mais eficiente  
O objetivo é recuperar as páginas danificadas e aplicar os backups necessários para deixá-las no mesmo ponto lógico do restante do banco

---

## 3 - Relação com páginas corrompidas

Uma página candidata a Page Restore normalmente é identificada a partir de:  
- Erro 823
- Erro 824
- Erro 825
- `msdb.dbo.suspect_pages`
- `DBCC CHECKDB`
- Mensagens no SQL Server Error Log
- Mensagens no Windows Event Viewer

A tabela `msdb.dbo.suspect_pages` pode ajudar a identificar o banco, o arquivo e o número da página afetada

Exemplo:  
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

O endereço da página é formado por:  
```text
file_id:page_id
```

Exemplo:  
```text
1:264
```

Nesse exemplo:  
```text
1   = file_id
264 = page_id
```

---

## 4 - Relação com DBCC CHECKDB

O `DBCC CHECKDB` pode identificar páginas corrompidas e indicar mensagens relacionadas ao problema

Exemplo:  
```sql
DBCC CHECKDB (PageRestoreDB) WITH NO_INFOMSGS, TABLERESULTS;
GO
```

O resultado pode indicar informações como:  
- Banco afetado
- Objeto afetado
- Índice afetado
- Tipo de corrupção
- Página afetada
- Nível mínimo de reparo sugerido

Quando a página afetada é identificada, é possível avaliar se Page Restore é uma alternativa viável

---

## 5 - Pré-requisitos conceituais

Para que Page Restore seja possível, normalmente é necessário ter:  
- Backup full válido
- Backup diferencial válido, quando existir
- Backups de log necessários para avançar a página restaurada
- Identificação correta das páginas danificadas
- Cadeia de log íntegra
- Banco em recovery model compatível com restauração por log
- Espaço e acesso aos arquivos de backup

Em bancos que usam recovery model `FULL` ou `BULK_LOGGED`, os backups de log são fundamentais para trazer a página restaurada até o ponto correto

---

## 6 - Por que aplicar os backups de log

Uma página restaurada a partir de um backup full volta ao estado em que estava no momento daquele backup, porém, o restante do banco pode estar em um LSN mais recente

Exemplo:  
```text
Domingo 00:00 = backup full
Segunda       = alterações no banco
Terça         = alterações no banco
Quarta        = página corrompida
```

Se a página for restaurada a partir do backup full de domingo, ela volta para o estado de domingo  
Mas o restante do banco está no estado de quarta-feira  
Por isso, é necessário aplicar os backups de log  
Os backups de log carregam as alterações ocorridas depois do backup full  
Assim, a página restaurada é atualizada até o mesmo ponto lógico do banco  

---

## 7 - Relação com LSN

O LSN - *Log Sequence Number* representa a sequência lógica das alterações registradas no transaction log  
Durante um Page Restore, uma página restaurada a partir de um backup antigo precisa ser atualizada com os registros de log posteriores  

Exemplo conceitual:  
```text
Backup full          = LSN 100
Backup log 1         = LSN 101 até 200
Backup log 2         = LSN 201 até 300
Backup log 3         = LSN 301 até 400
Banco atual          = LSN 400
Página restaurada    = LSN 100
```

Nesse caso, a página precisa receber os logs até chegar ao LSN 400  

---

## 8 - Sequência geral do Page Restore

A sequência conceitual de um Page Restore pode ser:  
1. Identificar a página corrompida
2. Confirmar a corrupção com `DBCC CHECKDB`
3. Verificar backups disponíveis
4. Verificar cadeia de log
5. Restaurar a página a partir do backup full com `NORECOVERY`
6. Restaurar backup diferencial com `NORECOVERY`, quando existir
7. Restaurar backups de log em sequência com `NORECOVERY`
8. Executar tail-log backup, quando aplicável
9. Restaurar o tail-log backup com `RECOVERY`
10. Executar `DBCC CHECKDB` novamente

A sequência exata depende do cenário, da edição do SQL Server, do estado do banco e dos backups disponíveis

---

## 9 - Sintaxe base de Page Restore

Exemplo conceitual:  
```sql
RESTORE DATABASE PageRestoreDB
PAGE = '1:264'
FROM DISK = 'C:\Backups\PageRestoreDB_FULL.bak'
WITH NORECOVERY;
GO
```

Nesse exemplo:  
```text
PageRestoreDB                         = banco de dados
PAGE = '1:264'                        = página que será restaurada
C:\Backups\PageRestoreDB_FULL.bak     = backup full
NORECOVERY                            = mantém a página em estado de restore para receber backups posteriores
```

---

## 10 - Restaurando mais de uma página

É possível informar mais de uma página no mesmo comando  

Exemplo conceitual:  
```sql
RESTORE DATABASE PageRestoreDB
PAGE = '1:264, 2:333, 4:14'
FROM DISK = 'C:\Backups\PageRestoreDB_FULL.bak'
WITH NORECOVERY;
GO
```

Nesse exemplo, serão restauradas as páginas:  
```text
1:264
2:333
4:14
```

Cada item representa:  
```text
file_id:page_id
```

---

## 11 - Restaurando backup diferencial

Se existir backup diferencial aplicável, ele deve ser restaurado após o backup full e antes dos backups de log  

Exemplo conceitual:  
```sql
RESTORE DATABASE PageRestoreDB
FROM DISK = 'C:\Backups\PageRestoreDB_DIFF.bak'
WITH NORECOVERY;
GO
```

O backup diferencial reduz a quantidade de backups de log que precisam ser aplicados  
Ele leva a página restaurada para um ponto mais recente do que o backup full  

---

## 12 - Restaurando backups de log

Após o backup full e o diferencial, os backups de log devem ser aplicados em sequência

Exemplo conceitual:  
```sql
RESTORE LOG PageRestoreDB
FROM DISK = 'C:\Backups\PageRestoreDB_LOG_001.trn'
WITH NORECOVERY;
GO

RESTORE LOG PageRestoreDB
FROM DISK = 'C:\Backups\PageRestoreDB_LOG_002.trn'
WITH NORECOVERY;
GO

RESTORE LOG PageRestoreDB
FROM DISK = 'C:\Backups\PageRestoreDB_LOG_003.trn'
WITH NORECOVERY;
GO
```

A ordem dos logs precisa respeitar a cadeia de LSN  
Se um backup de log intermediário estiver ausente ou inválido, a recuperação pode não ser possível até o ponto desejado  

---

## 13 - Tail-log backup no Page Restore

Depois de restaurar os backups de log disponíveis, pode ser necessário executar um tail-log backup  
O tail-log backup captura a parte final do transaction log que ainda não foi copiada em backup  
Isso ajuda a evitar perda de dados e manter a cadeia de log completa  

Exemplo conceitual:  
```sql
BACKUP LOG PageRestoreDB
TO DISK = 'C:\Backups\PageRestoreDB_TAIL.trn'
WITH NORECOVERY;
GO
```

Em cenários com corrupção ou erro durante o backup do log, pode ser necessário avaliar `CONTINUE_AFTER_ERROR`

Exemplo conceitual:  
```sql
BACKUP LOG PageRestoreDB
TO DISK = 'C:\Backups\PageRestoreDB_TAIL.trn'
WITH CONTINUE_AFTER_ERROR;
GO
```

A escolha da opção depende do estado do banco e do tipo de falha

---

## 14 - Restaurando o tail-log backup com RECOVERY

Após gerar o tail-log backup, ele deve ser restaurado para finalizar a sequência

Exemplo conceitual:  
```sql
RESTORE LOG PageRestoreDB
FROM DISK = 'C:\Backups\PageRestoreDB_TAIL.trn'
WITH RECOVERY;
GO
```

A opção `RECOVERY` finaliza o processo de restore  
Depois disso, a página deve estar consistente com o restante do banco

---

## 15 - NORECOVERY e RECOVERY

Durante o processo de Page Restore, os comandos intermediários usam `NORECOVERY`  
Isso permite que o SQL Server continue recebendo backups adicionais na sequência de restore

Exemplo:  
```text
FULL  WITH NORECOVERY
DIFF  WITH NORECOVERY
LOG   WITH NORECOVERY
LOG   WITH NORECOVERY
TAIL  WITH RECOVERY
```

A opção `RECOVERY` deve ser usada somente no último restore da sequência  
Quando `RECOVERY` é executado, o SQL Server finaliza a recuperação e não permite aplicar mais backups naquela sequência  

---

## 16 - Page Restore online

O SQL Server Enterprise Edition oferece suporte a Page Restore online  
Quando o Page Restore é online, o banco pode permanecer disponível durante o processo, dependendo do cenário  
Isso pode reduzir indisponibilidade em ambientes críticos  
Mesmo assim, as páginas que estão em restore não podem ser acessadas normalmente até o processo ser finalizado  
Em edições sem suporte a online restore, ou em cenários onde o banco está offline, o processo pode exigir indisponibilidade maior  

---

## 17 - Page Restore offline

No Page Restore offline, o banco ou parte relevante dele pode ficar indisponível durante o processo

Esse cenário pode ocorrer quando:  
- A edição não suporta online restore
- O banco já está offline
- O tipo de falha impede operação online
- A estratégia escolhida exige indisponibilidade

A escolha entre online e offline depende de:  
- Edição do SQL Server
- Estado do banco
- Tipo de corrupção
- Arquitetura de arquivos e filegroups
- Criticidade do ambiente
- Janela de manutenção disponível

---

## 18 - Quando Page Restore pode não ser suficiente

Page Restore pode não ser suficiente em alguns cenários  

Exemplos:
- Muitas páginas corrompidas
- Corrupção em metadados críticos
- Corrupção em páginas de alocação importantes
- Falta de backup full válido
- Falta de backups de log necessários
- Quebra na cadeia de log
- Corrupção espalhada pelo banco
- Problema físico ainda ativo no storage

Nesses casos, pode ser necessário avaliar:  
- Restore completo do banco
- Restore em outro servidor
- Restore parcial
- Restore de filegroup
- Uso de `DBCC CHECKDB` com opções de reparo
- Recuperação manual de dados
- Recriação de objetos afetados

---

## 19 - Page Restore não substitui backup

Page Restore depende diretamente de uma boa estratégia de backup  
Sem backups válidos, não existe página íntegra para restaurar  
Sem backups de log, pode não ser possível trazer a página restaurada até o LSN atual  

Por isso, Page Restore não substitui:  
- Backup full
- Backup diferencial
- Backup de log
- Retenção adequada
- Teste de restore
- Validação da cadeia de backup
- Execução periódica de `DBCC CHECKDB`

---

## 20 - Exemplo de linha do tempo

Exemplo conceitual:
```text
Domingo 00:00  FULL
Segunda 00:00  DIFF
Segunda 08:00  LOG 001
Segunda 12:00  LOG 002
Segunda 16:00  LOG 003
Segunda 17:00  Corrupção identificada
Segunda 17:01  Backup do tail-log
Segunda 17:05  Page Restore a partir do FULL
Segunda 17:10  Restore do DIFF
Segunda 17:15  Restore LOG 001
Segunda 17:20  Restore LOG 002
Segunda 17:25  Restore LOG 003
Segunda 17:35  Restore tail-log backup WITH RECOVERY
```

A página restaurada primeiro volta ao estado do backup full    
Depois recebe o diferencial  
Depois recebe os backups de log  
Por fim, recebe o tail-log backup e é recuperada

---

## 21 - Exemplo de sequência completa

Exemplo conceitual de sequência completa:

```sql
BACKUP LOG PageRestoreDB
TO DISK = 'C:\Backups\PageRestoreDB_TAIL.trn'
WITH NORECOVERY;
GO

RESTORE DATABASE PageRestoreDB
PAGE = '1:264'
FROM DISK = 'C:\Backups\PageRestoreDB_FULL.bak'
WITH NORECOVERY;
GO

RESTORE DATABASE PageRestoreDB
FROM DISK = 'C:\Backups\PageRestoreDB_DIFF.bak'
WITH NORECOVERY;
GO

RESTORE LOG PageRestoreDB
FROM DISK = 'C:\Backups\PageRestoreDB_LOG_001.trn'
WITH NORECOVERY;
GO

RESTORE LOG PageRestoreDB
FROM DISK = 'C:\Backups\PageRestoreDB_LOG_002.trn'
WITH NORECOVERY;
GO

RESTORE LOG PageRestoreDB
FROM DISK = 'C:\Backups\PageRestoreDB_TAIL.trn'
WITH RECOVERY;
GO
```

Esse exemplo é conceitual

Em ambiente real, é necessário validar:  
- Nome correto do banco
- Arquivos corretos de backup
- Ordem correta dos backups
- LSNs
- Estado do banco
- Páginas afetadas
- Edição do SQL Server
- Impacto na aplicação

---

## 22 - Validação após Page Restore

Após finalizar o Page Restore, é importante validar o banco

Exemplo:

```sql
DBCC CHECKDB (PageRestoreDB) WITH NO_INFOMSGS, TABLERESULTS;
GO
```

Também é importante verificar novamente a tabela `suspect_pages`

Exemplo:

```sql
SELECT
    DB_NAME(database_id) AS database_name,
    file_id,
    page_id,
    event_type,
    error_count,
    last_update_date
FROM msdb.dbo.suspect_pages
WHERE database_id = DB_ID('PageRestoreDB')
ORDER BY last_update_date DESC;
GO
```

A validação deve confirmar se ainda existem páginas suspeitas ou inconsistências pendentes

---

## 23 - Cuidados antes de executar Page Restore

Antes de executar Page Restore em ambiente real, é importante:  
- Confirmar a página afetada
- Confirmar se a corrupção ainda existe
- Verificar se há backup full válido
- Verificar se há backup diferencial aplicável
- Verificar se todos os backups de log estão disponíveis
- Conferir a cadeia de LSN
- Fazer cópia dos arquivos de backup
- Validar espaço em disco
- Avaliar impacto na aplicação
- Avaliar janela de manutenção
- Testar o restore em ambiente separado, quando possível
- Investigar a causa raiz da corrupção

---

## 24 - Pontos de atenção

Page Restore é uma técnica poderosa, mas exige cuidado

Pontos importantes:  
- A página precisa ser identificada corretamente
- Os backups precisam estar íntegros
- A cadeia de log precisa estar completa
- A sequência de restore precisa respeitar o LSN
- O uso de `RECOVERY` deve ocorrer somente no final
- O tail-log backup pode ser necessário para evitar perda de dados
- O banco deve ser validado novamente com `DBCC CHECKDB`
- A causa raiz da corrupção precisa ser investigada

---

## 25 - Resumo

Page Restore permite restaurar páginas específicas de um banco de dados SQL Server  
Ele pode ser útil quando a corrupção está limitada a poucas páginas  
A página restaurada a partir de um backup antigo precisa receber os backups diferenciais e de log necessários para chegar ao LSN correto  

O processo normalmente envolve:  
- Identificar a página corrompida
- Confirmar a corrupção
- Restaurar a página com `NORECOVERY`
- Aplicar diferencial, se existir
- Aplicar logs em sequência
- Executar tail-log backup, quando aplicável
- Restaurar o último log com `RECOVERY`
- Executar `DBCC CHECKDB` novamente

Page Restore não substitui uma estratégia de backup  
Sem backup válido e cadeia de log íntegra, a recuperação pode não ser possível

---

## Referências

- [Restaurar páginas (SQL Server)](https://learn.microsoft.com/pt-br/sql/relational-databases/backup-restore/restore-pages-sql-server?view=sql-server-ver16)
- [RESTORE Statements - Arguments (Transact-SQL)](https://learn.microsoft.com/pt-br/sql/t-sql/statements/restore-statements-arguments-transact-sql?view=sql-server-ver16)
- [suspect_pages (Transact-SQL)](https://learn.microsoft.com/pt-br/sql/relational-databases/system-tables/suspect-pages-transact-sql?view=sql-server-ver16)
- [Gerenciar a tabela suspect_pages (SQL Server)](https://learn.microsoft.com/pt-br/sql/relational-databases/backup-restore/manage-the-suspect-pages-table-sql-server?view=sql-server-ver16)