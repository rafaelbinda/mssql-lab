# A0007 – SQL Server Architecture

> **Author:** Rafael Binda  
> **Created:** 2026-02-18  
> **Version:** 3.0  

---

## Descrição

Informações a respeito da arquitetura do SQL Server

---

## Hands-on

[Q0002 - SQL Server Create Database](../scripts/Q0002-create-database.sql)  
[INST-Q0004 - SQL Physical Storage Layout](../../../dba-scripts/SQL-instance-information/INST-Q0004-physical-storage-layout.sql)

---

## 1 - Arquitetura do SQL Server

→ Recomendado pela Microsoft manter as seguintes extensões:  
- `.mdf` → Arquivo de dados  
- `.ldf` → Arquivo de logs
- `.ndf` → Arquivos secundários

### Arquivo .mdf

→ O arquivo com extensão `.mdf` é sempre o arquivo primário  
→ Só existe 1

### Arquivo .ndf

→ Os arquivos com extensão `.ndf` são arquivos secundários  
→ Pode ter "n" secundários

### Porque criar mais de um arquivo de dados?

- 1º motivo - Aumentar a capacidade de armazenamento

**Exemplo simplificado:**  
→ O espaço de armazenamento do arquivo primário está se esgotando, é possível criar um arquivo secundário em outro diretório que funcionará como uma expansão de espaço.

- 2º motivo - Desempenho / Paralelismo de I/O

**Exemplo simplificado:**  
1 - Existem dois arquivos de dados cada um em um volume diferente no storage  
2 - É criada uma tabela em um arquivo de dados em um volume D:\ do storage  
3 - É criada uma tabela em outro arquivo de dados em um volume E:\ do storage  
4 - Quando executar uma operação de JOIN entre essas tabelas o SQL consegue ler em paralelo essas tabelas  
**Porque?**  
→ Ele cria uma thread por arquivo de dados.  
→ Estando esses arquivos de dados em discos diferentes ele tem como operar em paralelo ganhando desempenho.

### Arquivo .ldf

→ O arquivo `.ldf` é um registro sequencial das operações de atualização que ocorrem no banco de dados  
→ É obrigatório pelo menos 1 arquivo de log  
→ É um arquivo de escrita sequencial (LSN - Log Sequence Number)

**Existe algum motivo de criar mais de um arquivo de log?**  
→ Para desempenho não adianta nada devido a escrita sequencial  
→ Aumentar o armazenamento

**Exemplo simplificado:**  
1 - O arquivo de log está em um disco que está sem espaço  
2 - É criado um segundo arquivo de log em outro disco/volume  
3 - Quando encerra a gravação completa do log no primeiro arquivo e não tem mais espaço físico ele começa a gravar no segundo arquivo de log

---

## 2 - Arquitetura interna do arquivo de dados

### Extents

→ Uma extent possui 8 páginas de 8 KB totalizando 64 KB  
→ Os dados são gravados em páginas de tamanho fixo  
→ Essa é a unidade de crescimento ou redução dos arquivos  
→ Sempre cresce múltiplo de 64 KB

---

## 3 - Bancos de Dados de Sistema

### MASTER

→ Mais importante de todos  
→ Catálogo da instância, com informações de Metadata dos objetos de instância  
→ Tem o nome dos bancos de dados que existem no banco  
→ É nele que está a localização dos arquivos de dados e logs de todos os bancos de dados existentes  
→ No processo de inicialização é o primeiro banco a ser inicializado  
→ Sem ele inicializado o SQL não abre

### MSDB

→ Armazena Metadata de diversas funcionalidades como SQL Agent  
→ Armazena Metadata de histórico Backup e Restore

### TEMPDB

→ Rascunho do SQL Server  
→ Banco de dados de operações temporárias

### MODEL

→ Modelo de criação para novos bancos de dados  
→ É um template

### DISTRIBUTION

→ Metadata da Replicação  
→ Nem sempre está presente no SQL Server  
→ Só vai aparecer se for configurado banco de dados distribuído  
→ Mantém 3 papéis:  
- Publishier → origem dos dados  
- Distributor → quem gerencia  
- Subscriber → destino dos dados

---

## 4 - Checkpoint

### O que acontece quando executamos um UPDATE no SQL Server?

### Cenário

Imagine uma aplicação conectada ao SQL Server executando um comando `UPDATE` para alterar registros em uma tabela.

### 4.1 - Execução inicial em memória (Buffer Cache)

O SQL Server **não altera os dados diretamente no disco**.

Primeiro ele trabalha em memória:

→ O comando `UPDATE` é executado no **Buffer Cache** (também chamado de **Data Cache**)  
→ O SQL Server mantém em memória apenas as páginas de dados mais utilizadas, não todo o banco  
→ Isso existe para garantir **alto desempenho**

### 4.2 - Verificação se a página já está em memória

O SQL Server verifica:  
→ A página de dados que será alterada já está no Buffer Cache?

- **Se SIM:**  
  → O UPDATE ocorre imediatamente em memória

- **Se NÃO:**  
  → O SQL Server lê a página do disco (.mdf/.ndf)  
  → Carrega a página para o Buffer Cache  
  → Executa o UPDATE em memória

### 4.3 - Proteção contra falhas — Transaction Log (.ldf)

→ Alterar somente em memória seria perigoso  
→ Se o servidor desligasse nesse momento, todas as alterações seriam perdidas  
Por isso:

→ Antes de confirmar a transação, o SQL Server grava as mudanças no **Transaction Log (.ldf)**

Esse processo registra:
- Valores **antes** da alteração
- Valores **depois** da alteração
- Utiliza uma sequência lógica através do **LSN (Log Sequence Number)**

Fluxo interno:
→ Localiza o último LSN  
→ Gera novos registros de log  
→ Grava a operação no log  
→ Libera o usuário (COMMIT)

Esse princípio é conhecido como:
→ **Write-Ahead Logging (WAL)**  
→ *(primeiro grava no LOG, depois no Data File)*

### 4.4 - Dados modificados ficam apenas em memória

Após o COMMIT:
→ As páginas alteradas ficam marcadas como **Dirty Pages**  
→ O dado atualizado ainda está somente no Buffer Cache  
→ O processo continua:
- Altera em memória  
- Registra no log  
- Altera em memória  
- Registra no log

### 4.5 - Execução do Processo de Checkpoint

→ Periodicamente o SQL Server executa o processo chamado **Checkpoint**

### O Checkpoint

→ Escreve as transações já finalizadas do Buffer Cache para os arquivos de dados (.mdf/.ndf).

### Objetivos

- Garantir recuperação rápida em caso de falha  
- Reduzir necessidade de REDO durante recovery  
- Evitar escrita constante direta no disco

### Por que não escrever direto no disco?

Se cada UPDATE fosse gravado imediatamente no disco:
→ O custo de I/O seria muito alto  
→ O desempenho cairia drasticamente  
→ Ocorreriam Page Splits constantemente  
→ Haveria expansão contínua das **Extents (64 KB)**

### Por isso o SQL Server utiliza

- Cache em memória
- Transaction Log
- Checkpoints periódicos

### Resumo do fluxo cronológico

1. UPDATE chega ao SQL Server  
2. Página é localizada ou carregada para o Buffer Cache  
3. Alteração ocorre em memória  
4. Mudança é registrada no Transaction Log (.ldf)  
5. Transação é confirmada (COMMIT)  
6. Página fica como Dirty Page  
7. Checkpoint grava definitivamente nos Data Files

---

## Referências

- [Arquivos e grupos de arquivos de banco de dados](https://learn.microsoft.com/pt-br/sql/relational-databases/databases/database-files-and-filegroups?view=sql-server-ver16)
- [Guia de arquitetura de páginas e extensões](https://learn.microsoft.com/pt-br/sql/relational-databases/pages-and-extents-architecture-guide?view=sql-server-ver16)
- [Pontos de verificação de banco de dados (Checkpoint)](https://learn.microsoft.com/pt-br/sql/relational-databases/logs/database-checkpoints-sql-server?view=sql-server-ver16)
- [Guia de arquitetura e gerenciamento do log de transações](https://learn.microsoft.com/pt-br/sql/relational-databases/sql-server-transaction-log-architecture-and-management-guide?view=sql-server-ver16)
