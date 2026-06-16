# A0012 – SQL Server Programming Objects

> **Author:** Rafael Binda  
> **Created:** 2026-03-11  
> **Version:** 4.0 

---

## Descrição

Este documento apresenta uma visão geral sobre Views no SQL Server, incluindo conceitos, uso prático, funcionamento interno e considerações de desempenho
 
---

## Hands-on  

[Q0010 - Views](../scripts/Q0010-sql-views.sql)   
[VIEW-Q0001 - View Metadata Objects](../../../dba-scripts/SQL-programming-objects/VIEW-Q0001-view-metadata.sql)   

---

## SQL Server Views

**O que é uma View?**

- Uma **View** é uma **consulta armazenada no SQL Server**  
- Para o usuário, a view **aparece como se fosse uma tabela**, porém na realidade ela é apenas um **SELECT armazenado**  
- Quando uma consulta é executada sobre uma view, o SQL Server executa a consulta definida na view  

---

### 1 - Benefícios de usar Views

Simplificação da administração de permissões  
Views ajudam a simplificar a segurança do banco de dados 
Facilita desenvolvimento de relatórios e exportações de dados e integrações porque a lógica de consulta pode ficar **centralizada no banco**    

Exemplo:
1. Um relatório precisa exibir informações que exigem **5 JOINs entre tabelas**  
2. Se o acesso for concedido diretamente às tabelas, será necessário liberar permissões em todas elas  
3. Se for criada uma **view com essa consulta**, basta conceder acesso **apenas à view**  

→ Dessa forma, a administração de permissões fica **mais simples e mais segura**  

### 2 - Camada de abstração

Views criam uma **camada de abstração entre a aplicação e a estrutura das tabelas**, isso permite:  
- Alterar tabelas internas sem impactar diretamente a aplicação
- Centralizar regras de consulta
- Padronizar acesso aos dados

### 3 - Características importantes das Views

- Uma view retorna **apenas um conjunto de resultados**
- Pode conter:
  - `JOIN`
  - `LEFT JOIN`
  - `RIGHT JOIN`
  - `UNION`
  - `GROUP BY`
  - `WHERE`
- Apesar disso, o resultado final é sempre **um único SELECT**

### 4 - Como o SQL Server executa uma View

**4.1 - Criação da view**

A view armazena **apenas a definição da consulta**

```sql
CREATE VIEW Sales.vw_CustomersOrders
AS
SELECT ...
FROM ...
JOIN ...
LEFT JOIN ...
```
---

**4.2 - Criação da view utilizando SCHEMABINDING**  
→ WITH SCHEMABINDING é uma opção usada quando você cria uma view para vincular (bind) a view à estrutura das tabelas que ela utiliza  
→ Impede que a estrutura das tabelas usadas pela view seja alterada de uma forma que quebraria a view  

**4.2.1 - Sem SCHEMABINDING (comportamento padrão)**

Imagine a view:  

```sql
CREATE VIEW Sales.vw_Test
AS
SELECT CustomerID, AccountNumber
FROM Sales.Customer;
```

Agora alguém executa:
```sql
ALTER TABLE Sales.Customer
DROP COLUMN AccountNumber;
```

→ O SQL Server permite isso  
→ Resultado: **A view continua existindo, mas fica quebrada**  
→ Quando alguém tentar usar a view:

```sql
SELECT * FROM Sales.vw_Test;
```
Erro:
```sql
Invalid column name 'AccountNumber'
```

**4.2.2 - Com SCHEMABINDING**

Agora criando a view assim:  
```sql
CREATE VIEW Sales.vw_Test
WITH SCHEMABINDING
AS
SELECT CustomerID, AccountNumber
FROM Sales.Customer;
```

Se alguém tentar:  
```sql
ALTER TABLE Sales.Customer
DROP COLUMN AccountNumber;
```

→ Resultado: o SQL Server bloqueia a operação:  
```sql
Cannot ALTER TABLE because the object 'vw_Test' is schema bound
--Ou seja:
--A estrutura da tabela não pode ser alterada enquanto a view depender dela
```

Quando é usado WITH SCHEMABINDING, algumas regras passam a valer:

- Deve usar schema nos objetos
  
→ Errado:  

```sql
FROM Customer
```

→ Correto:  
```sql
FROM Sales.Customer
```

- Não pode usar `SELECT *`
  
→ Errado: 
```sql
SELECT *
FROM Sales.Customer
```

→ Correto:  
```sql
SELECT CustomerID, AccountNumber
FROM Sales.Customer
```

- Objetos devem estar no mesmo banco
  
Não é possível referenciar outro banco de dados:
```sql
OtherDatabase.dbo.Table
```

- Quando DBAs usam SCHEMABINDING
  
Principalmente em Indexed Views
```sql
CREATE UNIQUE CLUSTERED INDEX IX_vw_SalesSummary_CustomerID
ON Sales.vw_SalesSummary (CustomerID);
GO
```

- Sem SCHEMABINDING não é possível criar índice na view

**Resumo simples**  
| Característica	| Sem SCHEMABINDING |	Com SCHEMABINDING |  
| ---------------| ------------------| ------------------|
| Alterar tabela	| permitido	| bloqueado |
| View pode quebrar	|sim |	não |
| Pode usar SELECT *	|sim	| não |
| Indexed view	| não	| sim |

### 5 - Consulta na view 

```sql
SELECT *
FROM Sales.vw_CustomersOrders
```

### 6 - Resolução dinâmica da view 

Quando executamos uma consulta na view, o SQL Server **não executa a view separadamente**, ele faz o seguinte:  
1. Pega o `SELECT` que foi enviado pelo usuário  
2. Junta com a definição da view  
3. Gera **uma única consulta final**, ou seja, o SQL Server executa algo equivalente a:  

```sql
SELECT *
FROM (
    SELECT ...
    FROM ...
    JOIN ...
) AS vw
```

→ Esse processo é chamado de **resolução dinâmica da view**  

### 7 - Desempenho

Uma view **não melhora nem piora o desempenho por si só**  
O desempenho dependerá de:  
- Da consulta
- Dos índices
- Das tabelas envolvidas
- Do plano de execução

→ Views são principalmente uma **ferramenta de organização e abstração**  

---

**7.1 - Estudo usando plano de execução**

Habilite o plano de execução no SSMS conforme informado a seguir:
```
Query → Include Actual Execution Plan
ou
CTRL + M
```

→ Esse processo representa o fluxo de execução que o SQL Server utiliza para retornar os dados  

O plano de execução deve ser interpretado:
- **Da direita para a esquerda**
- **De cima para baixo**

**Para efeito de comparação**  
**Faça a leitura do plano de execução ao executar uma consulta COM view**  
**Faça a leitura do plano de execução ao executar a mesma consulta SEM view**  

---

## Referências

- [Views (SQL Server)](https://learn.microsoft.com/pt-br/sql/relational-databases/views/views?view=sql-server-ver16)
- [CREATE VIEW (Transact-SQL)](https://learn.microsoft.com/pt-br/sql/t-sql/statements/create-view-transact-sql?view=sql-server-ver16)
