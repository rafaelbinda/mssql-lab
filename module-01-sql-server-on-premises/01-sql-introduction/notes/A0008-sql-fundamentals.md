# A0008 – SQL Server Fundamentals

> **Author:** Rafael Binda  
> **Created:** 2026-02-23  
> **Version:** 1.0  

---

## Descrição

Documento sobre os fundamentos do SQL Server, abrangendo tipos de instruções SQL (DDL, DML, DCL, DQL), ferramentas de execução, batches, comentários, regras para nomes de objetos, variáveis, operadores, instruções dinâmicas e controle de fluxo de execução.

---

## Hands-on

[Q0003 - SQL Server Fundamentals](../scripts/Q0003-sql-fundamentals.sql)

---

## 1 - Tipos de instruções SQL

### DDL — Data Definition Language
- `CREATE`
- `ALTER`
- `DROP`

→ Usado para **definir / alterar estruturas** (tabelas, índices, schemas, views, procedures etc.)

### DCL — Data Control Language
- `GRANT`
- `DENY`
- `REVOKE`

→ Usado para **permissões** (controle de acesso)

### DML — Data Manipulation Language
- `INSERT`
- `UPDATE`
- `DELETE`

→ Usado para **manipular dados** (linhas)

### DQL — Data Query Language
- `SELECT`

→ Usado para **consultar dados**

**Em algumas literaturas, DQL é tratado como parte de DML**

---

## 2 - Ferramentas para executar SQL

→ Para enviar comandos para o servidor, é preciso de um **cliente** que se conecte no SQL Server e execute as operações

Exemplos:
- SQL Server Management Studio (SSMS)
- Azure Data Studio
- SQLCMD.EXE

---

## 3 - Batch e o `GO`

### O que é um batch?
→ Um **batch** é um conjunto de comandos SQL que o servidor executa como uma unidade

### O que é `GO`?
- `GO` **não é comando T-SQL**
- É um **separador de batch** entendido somente pelo **SSMS** e pelo **SQLCMD**
- Ao usar `GO`, eu "quebro" o script em batches independentes

### Tipos de erro dentro de um batch

- Erro de sintaxe
→ O servidor detecta antes de executar  

- Erro de execução
→ A sintaxe passa, mas falha ao executar  

---

## 4 - Erro de sintaxe X Erro de execução (comportamento)

### Erro de sintaxe
- Quando eu executo um batch, o SQL Server **primeiro valida a sintaxe do batch inteiro**
- Se existir erro de sintaxe no meio, ele **não executa** o batch

### Erro de execução
- A sintaxe é válida.
- O SQL Server começa a executar e pode falhar no meio:
  - comandos anteriores podem ter sido aplicados
  - comandos posteriores podem não executar

→ Em transações (BEGIN TRAN / COMMIT / ROLLBACK) é possível controlar atomicidade

---

## 5 - Comentários

- Comentário de linha: `--`
- Comentário de bloco: `/* ... */`

---

## 6 - Regras para nomes de objetos

- Máximo de 128 caracteres
- 1º caractere: usar letra
→ Boa prática: na prática SQL Server permite mais coisas, mas é melhor evitar dor de cabeça  
- Demais: letras, números e alguns símbolos como por exemplo:  
→ `@`   variável  
→ `#`   tabela temporária  
→ `##`  tabela temporária global  
→ `_`   apenas organização visual  
→ `$`   permitido mas sem função especial

### Delimitadores
- `[ ... ]` ou `" ... "` podem delimitar nome de objeto (ex.: `[first name]`)
- Boa prática: usar `[ ... ]`

Sugestão:  
→ Evitar aspas duplas `"..."` porque o `QUOTED_IDENTIFIER` pode mudar o comportamento e gerar confusão.

- `' ... '` delimita **string**

### Nome completo (4-part name)
`instância.banco.schema.objeto`

- **instância:** nome do servidor/instância
- **banco:** database
- **schema:** "pasta lógica" de objetos (muitos bancos usam só `dbo`)
- **objeto:** nome real (tabela/view/proc)

**Por que qualificar `schema.objeto`?**
- Boa prática: sempre qualificar
- Evita ambiguidades principalmente se existirem objetos com mesmo nome em schemas diferentes
- Melhora previsibilidade e manutenção
- Reduz chance de resolver objeto errado por default schema do usuário

---

## 7 - Variáveis

### Variáveis locais

- Definidas pelo usuário com `DECLARE`
- Sempre começam com `@`
- Armazenam valores temporários durante a execução do script
- Recebem valores com `SET` ou `SELECT`

**Sintaxe:** `DECLARE @Nome Tipo(Tamanho)`

Exemplo:
```sql
DECLARE @X INT;
SET @X = 10;

SELECT @X AS Valor;
```

→ A variável existe apenas dentro do **batch** onde foi declarada  
→ Quando o SQL Server encontra um `GO`, o batch é encerrado  
→ Após o `GO`, a variável deixa de existir

Exemplo:
```sql
DECLARE @X INT = 10;
GO
SELECT @X;  -- Erro: variável não existe mais
```

> **Importante:** variáveis não são globais na sessão inteira. Elas são válidas apenas no batch atual

### Variáveis de sistema (globais)

- Começam com `@@`
- São fornecidas pelo SQL Server
- Retornam informações do ambiente ou da execução

Exemplos:
- `@@ERROR` → código de erro da última instrução executada
- `@@VERSION` → versão do SQL Server
- `@@ROWCOUNT` → quantidade de linhas afetadas pela última instrução

---

## 8 - Operadores

### Tipos
- Aritméticos: `+ - * /`
- Comparação: `= <> > < >= <=`
- Concatenação: `+` (string)
- Lógicos: `AND OR NOT`

Precedência: Usar `(...)` para deixar explícito

---

## 9 - Instruções dinâmicas (Dynamic SQL)

- `EXEC()` executa uma string como comando
- Se concatenar valores não-string é preciso converter com `CAST`, `CONVERT`, `STR`
- Boa prática: preferir `sp_executesql` (permite parâmetros, fica mais seguro e reaproveita plano)
- Atenção com SQL Injection quando houver entrada externa

---

## 10 - Controlando fluxo de execução

- Bloco: `BEGIN ... END`
- Condicional: `IF ... ELSE`
- Loop: `WHILE`
- Lista/decisão: `CASE`

---

## Referências

- [Instruções Transact-SQL (DDL, DML, DCL)](https://learn.microsoft.com/pt-br/sql/t-sql/statements/statements?view=sql-server-ver16)
- [sp_executesql (Transact-SQL)](https://learn.microsoft.com/pt-br/sql/relational-databases/system-stored-procedures/sp-executesql-transact-sql?view=sql-server-ver16)
- [Controle de fluxo (Transact-SQL)](https://learn.microsoft.com/pt-br/sql/t-sql/language-elements/control-of-flow?view=sql-server-ver16)
