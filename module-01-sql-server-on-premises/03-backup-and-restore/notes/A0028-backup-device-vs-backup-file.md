# A0028 – Backup Device vs Backup File

> **Author:** Rafael Binda  
> **Created:** 2026-04-14  
> **Version:** 1.0  

---

## Descrição

Este conteúdo apresenta a diferença entre **Backup Device** e **Backup File** no SQL Server, mostrando como cada abordagem funciona, onde a informação fica registrada, como o SQL Server trata a mídia de backup e quais opções podem impactar o comportamento do arquivo gerado  
O objetivo é entender a diferença entre utilizar um destino lógico previamente registrado no `master` e informar diretamente um arquivo físico no comando `BACKUP DATABASE`

---

## Hands-on

[Q0025 - Backup Device vs Backup File](../scripts/Q0025-sql-backup-device-vs-backup-file.sql)  
[INST-Q0017 - Backup Device Inspection](../../../dba-scripts/SQL-instance-information/INST-Q0017-backup-device-inspection.sql)

---

## Backup Device

### Conceito

Backup Device é um objeto lógico registrado previamente no banco `master` apontando para um destino físico de backup, como um arquivo em disco ou dispositivo de fita  
Na prática, ele funciona como um alias para um caminho físico já cadastrado no SQL Server  
Essa configuração pode ser visualizada em:

```
-> Server Objects -> Backup Devices
```

### Criação de Backup Device

Pode ser criado pela interface gráfica ou via comando

Exemplo:

```sql
USE master
GO

EXEC dbo.sp_addumpdevice
    @devtype = N'disk',
    @logicalname = N'BackupMaster',
    @physicalname = N'C:\Backups\BackupMaster.bak'
GO
```

### Utilização do Backup Device

Após criar o device, o backup pode ser executado referenciando apenas o nome lógico

```sql
BACKUP DATABASE master
TO BackupMaster
GO
```

Nesse exemplo:
- `BackupMaster` é o nome lógico do device
- O caminho físico já foi previamente registrado no `master`

### Observações sobre Backup Device

- O SQL Server armazena a referência do device no banco `master`
- O device não é o backup em si, mas sim o registro lógico de onde o backup será gravado
- O uso de Backup Device era mais comum em ambientes legados
- Em ambientes modernos, normalmente é mais comum utilizar diretamente `TO DISK = 'caminho'`
- Se o arquivo físico for movido, removido ou o path deixar de existir, o device continuará registrado, mas o backup falhará ao ser executado

### Visualização do conteúdo da mídia

Pela interface é possível visualizar os backups contidos na mídia associada ao device

Caminho:
```
- Server Objects -> Backup Devices -> BackupMaster -> Botão direito -> Properties -> Media Contents
```

Isso permite listar os backup sets contidos naquele arquivo de backup

---

## Backup File

### Conceito

Backup File é o backup realizado diretamente para um arquivo físico informado no comando, sem necessidade de cadastro prévio de um device no banco `master`  
Essa é a forma mais comum de executar backup em ambientes atuais

### Exemplo

```sql
BACKUP DATABASE master
TO DISK = N'C:\Backups\BackupMaster.bak'
GO
```

### Comportamento do arquivo

Ao executar um backup para um arquivo `.bak` já existente, o comportamento dependerá das opções utilizadas no comando

Por padrão, o SQL Server utiliza `NOINIT`, ou seja:
- Acrescenta um novo backup set ao arquivo existente
- Mantém os backups anteriores dentro da mesma mídia
- Aumenta o tamanho do arquivo físico

Exemplo conceitual:
```
Execução 1 → o arquivo contém 1 backup set
Execução 2 → o arquivo passa a conter 2 backup sets
Execução 3 → o arquivo passa a conter 3 backup sets
```

Isso explica por que, ao executar novamente o backup para o mesmo arquivo, o `.bak` aumenta de tamanho

---

## Diferença entre Backup Device e Backup File

### Backup Device

- É um objeto lógico cadastrado no banco `master`
- Aponta para um destino físico previamente registrado
- Pode ser reutilizado em comandos de backup sem repetir o path
- Tem uso mais associado a ambientes legados ou configurações antigas

### Backup File

- É o arquivo físico informado diretamente no comando
- Não depende de cadastro prévio no `master`
- É mais flexível e mais comum em ambientes atuais
- Normalmente é a abordagem mais utilizada em rotinas modernas e scripts automatizados

---

## Via Interface

No SQL Server Management Studio, ao acessar as propriedades do banco e as opções de backup, algumas opções mudam conforme o contexto do banco e do tipo de backup escolhido

Exemplos de navegação:

```
-> AdventureWorks -> Properties -> Options -> Recovery Model    = Simple
                                                                  Bulk Logged
                                                                  Full 

-> AdventureWorks -> Tasks -> Back Up -> General -> Backup Type = Full 
                                                                  Differential
```

### Comportamento observado no Recovery Model SIMPLE

Quando o banco está em `RECOVERY MODEL SIMPLE`:
- Não aparece opção de backup de LOG
- A interface limita algumas possibilidades conforme o tipo de backup selecionado

Motivo:
- No modelo SIMPLE não existe backup de LOG
- Por isso não há como realizar recuperação ponto a ponto utilizando transaction log

Importante:
- A disponibilidade das opções de backup de **File** e **Filegroup** não depende apenas do Recovery Model
- Essas opções estão ligadas também ao tipo de banco, à estrutura física do banco e ao tipo de backup selecionado na interface
- O Recovery Model influencia diretamente a existência de backup de LOG, mas não é o único fator para exibição de opções de File e Filegroup

---

## Script gerado pela interface

Ao gerar o script pela interface, o SQL Server pode produzir algo semelhante a isto:

```sql
BACKUP DATABASE [ExamplesDB]
TO DISK = N'C:\Backups\BackupMaster.bak'
WITH 
    NOFORMAT,
    NOINIT,
    NAME = N'ExamplesDB_FULL Database Backup',
    SKIP,
    NOREWIND,
    NOUNLOAD,
    STATS = 10
GO
```

---

## Explicação dos parâmetros

### NOFORMAT

- Não recria a estrutura da mídia
- Mantém o media set existente
- Não apaga a definição da mídia

### NOINIT

- Acrescenta o backup ao arquivo existente
- Mantém os backup sets anteriores
- É o comportamento padrão do SQL Server

### INIT

- Sobrescreve os backup sets existentes na mídia
- Mantém a estrutura da mídia sem recriá-la
- Pode respeitar validações de nome de mídia e expiração quando configuradas

### FORMAT

- Recria a estrutura da mídia de backup
- Remove os backup sets anteriores
- Equivale a iniciar uma nova mídia de backup

### NAME

- Define o nome lógico do backup set
- Ajuda na identificação do backup dentro do arquivo

Importante:
- `NAME` define o nome do **backup set**
- Não define o nome do arquivo físico
- Não deve ser confundido com `MEDIANAME`, que identifica o media set

### STATS = 10

- Exibe feedback do progresso a cada 10% executado
- Ajuda no acompanhamento da execução do backup

### SKIP

- Ignora algumas verificações relacionadas à mídia
- É mais associado a cenários específicos e mídias removíveis

### NOREWIND

- Utilizado principalmente em operações com fita
- Indica que a mídia não deve ser rebobinada ao final

### NOUNLOAD

- Utilizado principalmente em operações com fita
- Indica que a mídia não deve ser descarregada ao final

---

## Media Options na interface

Ao acessar as opções de mídia no assistente de backup, alguns comportamentos são representados por configurações equivalentes às cláusulas T-SQL

### Append to the existing backup set

- Equivale a `NOINIT`
- Acrescenta um novo backup set ao arquivo ou mídia existente

### Overwrite all existing backup sets

- Normalmente está relacionado ao uso de `INIT`
- Sobrescreve os backup sets existentes na mídia

Observação:
- `INIT` sobrescreve os backup sets existentes
- `FORMAT` recria completamente a mídia
- embora muitas vezes as duas opções sejam associadas à sobrescrita, tecnicamente elas não são idênticas

### Check media set name and backup set expiration

- Faz validação do nome da mídia e da expiração antes de sobrescrever
- Ajuda a evitar sobrescrita indevida quando ainda existe validade ativa configurada

---

## Expiração de backup

O SQL Server permite definir expiração para um backup set utilizando opções como `EXPIREDATE` ou `RETAINDAYS`  
Quando a expiração está em uso, a sobrescrita pode ser bloqueada se o backup ainda estiver dentro do prazo de retenção  
Para esse comportamento funcionar corretamente, normalmente é necessário observar:

- Uso de `INIT` quando a intenção é sobrescrever backup sets existentes
- Existência de backup set anterior com expiração ativa
- Correspondência do `MEDIANAME`, quando esse controle estiver sendo utilizado
- Validações habilitadas na operação de backup

---

## Media Set e Backup Set

### Media Set

É o conjunto da mídia de backup utilizado pelo SQL Server

Exemplo:
- Um arquivo `.bak`
- Múltiplos arquivos de um backup dividido
- Uma fita
- Conjunto espelhado de mídias

### Backup Set

É cada backup individual gravado dentro da mídia

Exemplo:
- Um FULL gravado hoje
- Um DIFFERENTIAL gravado amanhã
- Outro FULL gravado depois

Todos esses podem estar armazenados dentro do mesmo arquivo físico quando se utiliza `NOINIT`

---

## Consultas úteis para inspeção da mídia

### Verificar os backup sets contidos em um arquivo

```sql
RESTORE HEADERONLY
FROM DISK = N'C:\Backups\BackupMaster.bak'
GO
```

### Verificar os arquivos contidos em um backup

```sql
RESTORE FILELISTONLY
FROM DISK = N'C:\Backups\BackupMaster.bak'
GO
```

### Verificar o conteúdo lógico da mídia

```sql
RESTORE LABELONLY
FROM DISK = N'C:\Backups\BackupMaster.bak'
GO
```

---

## Observações finais

- Backup Device é um conceito válido no SQL Server, mas pouco utilizado em ambientes modernos
- Backup File tende a ser mais prático, flexível e comum em rotinas atuais
- `NOINIT` pode fazer um mesmo arquivo conter vários backup sets, exigindo atenção no momento do restore
- `INIT` e `FORMAT` não são equivalentes, embora ambos possam sobrescrever conteúdos anteriores
- `NAME` identifica o backup set e não o arquivo físico
- Opções como `SKIP`, `NOREWIND` e `NOUNLOAD` estão mais associadas a cenários com fita do que a backup em disco
- Sempre que possível, validar o conteúdo da mídia com comandos como `RESTORE HEADERONLY`, `RESTORE FILELISTONLY` e `RESTORE LABELONLY`
- Em scripts modernos, normalmente é preferível utilizar diretamente `TO DISK = 'caminho'` em vez de depender de Backup Devices

---

## Referências

- [Backup Devices (SQL Server)](https://learn.microsoft.com/pt-br/sql/relational-databases/backup-restore/backup-devices-sql-server?view=sql-server-ver16)
- [sp_addumpdevice (Transact-SQL)](https://learn.microsoft.com/pt-br/sql/relational-databases/system-stored-procedures/sp-addumpdevice-transact-sql?view=sql-server-ver16)
- [RESTORE Statements - HEADERONLY (Transact-SQL)](https://learn.microsoft.com/pt-br/sql/t-sql/statements/restore-statements-headeronly-transact-sql?view=sql-server-ver16)
- [Media sets, media families e backup sets (SQL Server)](https://learn.microsoft.com/pt-br/sql/relational-databases/backup-restore/media-sets-media-families-and-backup-sets-sql-server?view=sql-server-ver16)
- [BACKUP (Transact-SQL)](https://learn.microsoft.com/pt-br/sql/t-sql/statements/backup-transact-sql?view=sql-server-ver16)
