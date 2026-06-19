# A0027 – Backup Options and Media Handling

> **Author:** Rafael Binda  
> **Created:** 2026-04-13  
> **Version:** 3.0  

---

## Descrição

Este conteúdo aborda as principais opções de configuração de backup no SQL Server, com foco no comportamento da mídia de backup, manipulação de backup sets e impacto das opções utilizadas durante a execução  
São apresentados conceitos como INIT, FORMAT, NOINIT, compressão, mirror, checksum e copy_only, além de sua influência na estrutura e reutilização dos arquivos de backup  
O objetivo é compreender como o SQL Server gerencia a mídia de backup e como essas opções impactam diretamente a estratégia de backup e restore

---

## Hands-on

[Q0024 - Backup Options and Media Handling](../scripts/Q0024-sql-backup-options-and-media-handling.sql)

---

## Opções para backup

- Um arquivo de backup pode conter vários backups (backup set)
- Um backup pode ser dividido em múltiplos arquivos (split backup)
- É possível realizar backup diretamente para URL (Azure Blob Storage) a partir do SQL Server 2014
- O histórico de backup e restore é armazenado em tabelas do banco msdb

Principais tabelas:

- msdb.dbo.backupset
- msdb.dbo.backupmediafamily
- msdb.dbo.restorehistory

Observação:

- Essas tabelas são essenciais para auditoria, troubleshooting e validação da cadeia de backup

---

## Opções do backup para manipulação de mídia

### FORMAT

- Sobrescreve completamente o arquivo de backup
- Não verifica a validade dos backups existentes
- Remove todos os backup sets da mídia
- Reinicializa a estrutura da mídia de backup

Uso típico:

- Quando se deseja garantir que nenhum backup anterior será mantido

### INIT

- Sobrescreve o arquivo de backup
- Verifica a validade da mídia antes de sobrescrever
- Remove os backup sets existentes mantendo a estrutura da mídia

Diferença para FORMAT:

- INIT respeita a mídia existente
- FORMAT recria a mídia

### NOINIT (padrão)

- Acrescenta o backup ao arquivo existente
- Mantém todos os backups anteriores
- É o comportamento padrão do SQL Server

Impacto:

- Pode gerar arquivos grandes com múltiplos backup sets
- Exige controle na hora do restore (identificação correta do backup set)

Exemplo conceitual:

- Execução 1 → arquivo contém 1 backup
- Execução 2 → arquivo passa a conter 2 backups
- Execução 3 → arquivo passa a conter 3 backups

Cada novo backup é adicionado como um novo backup set dentro do mesmo arquivo

### COMPRESSION

- Compacta os dados durante o backup
- Reduz uso de disco e I/O
- Pode melhorar o tempo de execução

Impacto:

- Reduz tempo de escrita em disco
- Pode reduzir tempo de restore

#### Configuração de compressão padrão

O SQL Server pode ser configurado para utilizar compressão de backup por padrão em nível de instância

```sql
EXEC sp_configure 'show advanced options', 1;
RECONFIGURE;

EXEC sp_configure 'backup compression default', 1;
RECONFIGURE;
```

##### Comportamento

- Quando habilitado, todos os backups passam a utilizar compressão por padrão
- Reduz o consumo de espaço em disco
- Diminui a quantidade de I/O durante o processo de backup
- Em muitos cenários, pode melhorar o tempo de execução

#### Considerações

- A compressão aumenta o consumo de CPU
- O impacto depende do workload e da capacidade do servidor
- Em ambientes com gargalo de I/O, costuma trazer ganho significativo
- Pode ser sobrescrita no comando de backup:

```sql
BACKUP DATABASE ExamplesDB
TO DISK = 'C:\Backup.bak'
WITH NO_COMPRESSION;
```

#### Observação

- A configuração é aplicada em nível de instância
- Afeta todos os backups realizados após a alteração
- Recomenda-se validar impacto em ambientes de produção antes de habilitar

### MIRROR TO

- Permite criar até 4 cópias idênticas do backup
- As mídias devem ser do mesmo tipo e com desempenho similar
- Todas as cópias são geradas simultaneamente

Objetivo:

- Redundância imediata do backup

Diferença:

- Mirror: cópias completas idênticas
- Split: divide o backup em múltiplos arquivos

Exemplo inválido:

- Utilizar mídias de tipos diferentes (ex: DISK e TAPE)
- Utilizar dispositivos com desempenho muito diferente, pode causar falha na operação

### CHECKSUM

- Adiciona validação de integridade ao backup
- Permite detectar corrupção durante backup ou restore
- Valida páginas durante o processo

Importante:

- Não corrige erro, apenas detecta
- Ajuda a evitar restore de backup corrompido

Impacto:

- Pequeno aumento no uso de CPU durante o backup
- Pode gerar leve aumento no tempo de execução
- Impacto geralmente menor que o uso de COMPRESSION

Boa prática:

- Utilizar sempre que possível em ambiente de produção

### COPY_ONLY

- Executa backup sem interferir na cadeia de backups
- Não altera a base dos backups diferenciais
- Não interfere na sequência de backups de log

Uso típico:

- Backups ad-hoc (fora da rotina padrão definida)
- Cópias para testes
- Entrega de backup para terceiros

[Mais informações sobre COPY_ONLY - ver item 7](A0023-backup-fundamentals.md)

---

## Permissões para backup em diretório de rede

Quando o backup é gravado em um diretório de rede ou em uma pasta compartilhada, o SQL Server precisa ter permissão de escrita no destino  
É importante lembrar que quem grava o arquivo de backup não é o usuário conectado no SSMS, mas sim a conta de serviço utilizada pela instância do SQL Server  
Em cenários com múltiplas instâncias ou múltiplos servidores, pode ser necessário criar contas específicas para separar permissões por instância

Exemplo conceitual:
- Instância 1 grava backups em `S:\DATABASES\backup\instancia1`
- Instância 2 grava backups em `S:\DATABASES\backup\instancia2`
- Cada instância utiliza uma conta diferente com permissão apenas no seu diretório

### Exemplo com PowerShell

O exemplo abaixo cria usuários locais e concede permissão de modificação em pastas específicas

```powershell
$pass1 = ConvertTo-SecureString "StrongPassword_1" -AsPlainText -Force
$pass2 = ConvertTo-SecureString "StrongPassword_2" -AsPlainText -Force

# Create local users
New-LocalUser -Name "USRSQL1" -Password $pass1 -FullName "USRSQL1" -Description "Conta para backup SQL instância 1"
New-LocalUser -Name "USRSQL2" -Password $pass2 -FullName "USRSQL2" -Description "Conta para backup SQL instância 2"

# Grant Modify permission to each user in the specific backup folder
icacls "S:\DATABASES\backup\instancia1" /grant "USRSQL1:(OI)(CI)M" /T
icacls "S:\DATABASES\backup\instancia2" /grant "USRSQL2:(OI)(CI)M" /T
```

### Observações importantes

- A senha usada no exemplo deve ser substituída por uma senha segura
- Evite deixar senhas reais documentadas em arquivos do repositório
- A conta configurada no serviço do SQL Server deve ter permissão no destino do backup
- Permissão `Modify` permite criar, alterar e excluir arquivos dentro da pasta
- Em ambiente de domínio, prefira contas de domínio ou contas de serviço gerenciadas quando aplicável
- Para caminhos UNC, utilize o formato `\\servidor\compartilhamento\pasta`
- Sempre teste a escrita do backup no destino antes de depender dele em produção

---

## Referências

- [BACKUP (Transact-SQL)](https://learn.microsoft.com/pt-br/sql/t-sql/statements/backup-transact-sql?view=sql-server-ver16)
- [Compressão de backup (SQL Server)](https://learn.microsoft.com/pt-br/sql/relational-databases/backup-restore/backup-compression-sql-server?view=sql-server-ver16)
- [Conjuntos de mídia de backup espelhados (SQL Server)](https://learn.microsoft.com/pt-br/sql/relational-databases/backup-restore/mirrored-backup-media-sets-sql-server?view=sql-server-ver16)
- [Histórico de backup e informações de cabeçalho (SQL Server)](https://learn.microsoft.com/pt-br/sql/relational-databases/backup-restore/backup-history-and-header-information-sql-server?view=sql-server-ver16)
