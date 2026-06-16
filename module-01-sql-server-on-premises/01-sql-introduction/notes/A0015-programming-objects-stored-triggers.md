# A0015 – SQL Server Programming Objects

> **Author:** Rafael Binda  
> **Created:** 2026-03-17  
> **Version:** 1.0 

---

## Descrição  
 
Este documento apresenta uma visão geral sobre Triggers no SQL Server, incluindo conceitos, funcionamento, uso das tabelas virtuais INSERTED e DELETED e os principais tipos de triggers
 
---

## Hands-on  

[Q0013 - Triggers](../scripts/Q0013-sql-triggers.sql)  
[TRIG-Q0001 - Trigger Metadata Objects](../../../dba-scripts/SQL-programming-objects/TRIG-Q0001-trigger-metadata.sql) 

---

## SQL Server Triggers 

### O que é uma Trigger?

Uma **Trigger** é um objeto de banco de dados que executa automaticamente um conjunto de comandos SQL quando ocorre um determinado evento  
Triggers **não podem ser executadas diretamente pelo usuário**, pois são acionadas automaticamente quando o evento associado ocorre  
Esse evento pode ser:
- Uma operação de **INSERT**, **UPDATE** ou **DELETE**
- Eventos administrativos no banco ou instância

### 1 - Características das Triggers

- Executam automaticamente após um evento
- Não podem ser chamadas diretamente pelo usuário
- São utilizadas principalmente para **auditoria, validação ou automação de processos**
- Podem gerar **impacto de desempenho** se utilizadas de forma excessiva
- Podem aumentar a carga de trabalho na **tempdb**, dependendo da operação executada  

### 2 - Tabelas Virtuais INSERTED e DELETED

Durante a execução de uma Trigger, o SQL Server disponibiliza duas tabelas especiais que existem **somente durante a execução da trigger**:  
- `INSERTED`
- `DELETED`

Fora do contexto da Trigger essas tabelas **não podem ser acessadas**  

#### 2.1 - Operação INSERT 

Quando ocorre um **INSERT**, o SQL Server cria uma tabela chamada `INSERTED`  
Essa tabela possui:  
- A mesma estrutura da tabela original
- Os registros recém inseridos

Exemplo conceitual:
```sql
SELECT *
FROM INSERTED;
```

→ Isso permite identificar **quais dados o usuário inseriu no banco de dados**

#### 2.2 - Operação DELETE 

Quando ocorre um **DELETE**, o SQL Server cria uma tabela chamada `DELETED`  
Essa tabela contém:  
- A mesma estrutura da tabela original
- Os registros que foram removidos

Exemplo conceitual:
```sql
SELECT *
FROM DELETED;
```

Assim é possível verificar **quais dados foram excluídos**  

### 3 - Operação UPDATE 

Em operações de **UPDATE**, ambas as tabelas ficam disponíveis  
Isso permite comparar os valores antes e depois da modificação  

| Tabela | Conteúdo |
|------|------|
| `DELETED` | valores **antes da alteração** |
| `INSERTED` | valores **após a alteração** |

**Exemplo conceitual das operações**

```sql
IF EXISTS (SELECT 1 FROM INSERTED) 
AND NOT EXISTS (SELECT 1 FROM DELETED)
BEGIN
    -- operação de INSERT
END

IF EXISTS (SELECT 1 FROM INSERTED) 
AND EXISTS (SELECT 1 FROM DELETED)
BEGIN
    -- operação de UPDATE
END

IF EXISTS (SELECT 1 FROM DELETED) 
AND NOT EXISTS (SELECT 1 FROM INSERTED)
BEGIN
    -- operação de DELETE
END
```

### 4 - Tipos de Trigger

#### 4.1 - Trigger DML 

Esse é o tipo **mais utilizado no dia a dia**  
Triggers **DML (Data Manipulation Language)** são vinculadas a uma **tabela ou view**  
Elas são executadas quando ocorre:  
- `INSERT`
- `UPDATE`
- `DELETE`

**Exemplo de uso:** Registrar alterações em uma tabela de auditoria  

**Exemplo de cenário:**    

→ Tabela principal: `Customers`  
→ Tabela de auditoria: `Audit_Customers`  

**Resultado esperado:**  
Quando ocorrer algum `INSERT` ou `UPDATE`, as informações a seguir serão armazenadas na tabela de auditoria:  
→ ID  
→ Operation Type  
→ User  
→ Host  
→ CustomerID  
→ Name  
→ FirstName  
→ MiddleName  
→ LastName  
→ Deleted (flag lógica 0 ou 1)  

#### 4.2 - Trigger DDL 

Triggers **DDL (Data Definition Language)** são disparadas quando ocorrem eventos estruturais no banco de dados  
Exemplos:  
- `CREATE TABLE`
- `ALTER TABLE`
- `DROP TABLE`
- `CREATE PROCEDURE`

→ São utilizadas principalmente para **auditoria administrativa**  
→ O exemplo a seguir apresenta um cenário em que quando ocorrer qualquer CREATE TABLE será disparada a trigger e ela vai gravar informações para **auditoria** em uma tabela denominada dbo.SchemaChangesLog:  
```sql
CREATE OR ALTER TRIGGER trg_LogCreateTable
ON DATABASE
FOR CREATE_TABLE 
AS
BEGIN

    INSERT INTO dbo.SchemaChangesLog
    --Prossegue com a regra 

END 
```

#### 4.3 - Trigger de Login 

Triggers de login são executadas **no momento em que um usuário tenta se conectar ao SQL Server**  
Usos comuns:  
- Auditoria de acesso  
- Bloqueio de conexões indevidas  
- Restrição de acesso a determinadas ferramentas  

Exemplo conceitual:  
→ Se uma conexão for detectada a partir de uma ferramenta não autorizada (como Excel), a trigger pode executar um **ROLLBACK**, impedindo o acesso  

```sql
CREATE OR ALTER TRIGGER trg_BlockExcelLogin
ON ALL SERVER --Perceba que avalia em todo o servidor independente de banco de dados
FOR LOGON
AS
BEGIN

    DECLARE @ProgramName NVARCHAR(128);

    SELECT @ProgramName = program_name
    FROM sys.dm_exec_sessions
    WHERE session_id = @@SPID;

    -- Checa se a conexão está vindo do Excel
    IF @ProgramName LIKE '%Excel%'
    BEGIN

        ROLLBACK;

    END

END;
```

**Boas Práticas**

Triggers devem ser utilizadas com cuidado  
Boas práticas incluem:  
- Evitar lógica complexa dentro de triggers
- Evitar operações demoradas
- Utilizar triggers principalmente para **auditoria ou validação**
- Sempre considerar o impacto no desempenho

**Resumo**

| Tipo de Trigger | Scopo | Evento |
|---|---|---|
| DML Trigger | Tabela / View | INSERT, UPDATE, DELETE |
| DDL Trigger (Database) | Database | CREATE TABLE, ALTER TABLE, DROP PROCEDURE |
| DDL Trigger (Server) | Instância | CREATE LOGIN, ALTER LOGIN, etc |
| Login Trigger | Instância | LOGIN |

---

## Referências

- [Triggers DML](https://learn.microsoft.com/pt-br/sql/relational-databases/triggers/dml-triggers?view=sql-server-ver16)
- [Triggers DDL](https://learn.microsoft.com/pt-br/sql/relational-databases/triggers/ddl-triggers?view=sql-server-ver16)
- [Triggers de logon](https://learn.microsoft.com/pt-br/sql/relational-databases/triggers/logon-triggers?view=sql-server-ver16)
- [CREATE TRIGGER (Transact-SQL)](https://learn.microsoft.com/pt-br/sql/t-sql/statements/create-trigger-transact-sql?view=sql-server-ver16)
