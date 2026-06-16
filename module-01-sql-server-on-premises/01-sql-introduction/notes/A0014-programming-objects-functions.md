# A0014 – SQL Server Programming Objects

> **Author:** Rafael Binda  
> **Created:** 2026-03-17  
> **Version:** 1.0 

---

## Descrição  

Este documento apresenta uma visão geral sobre Functions no SQL Server, incluindo conceitos, tipos de funções e exemplos de uso
 
---

## Hands-on  

[Q0012 - Functions](../scripts/Q0012-sql-functions.sql)  
[FUNC-Q0001 - Function Metadata Objects](../../../dba-scripts/SQL-programming-objects/FUNC-Q0001-function-metadata.sql) 

---

## SQL Server Functions 
 
### O que é uma Função?

Uma **Função** no SQL Server é um objeto de banco de dados que recebe parâmetros, executa uma lógica e retorna um resultado  
Elas são similares às funções encontradas em linguagens de programação 

Características principais:
- Podem receber **parâmetros**
- Executam **lógica em T-SQL**
- **Sempre retornam um valor ou uma tabela**
- Seu objetivo principal é **processar dados e retornar resultados**
- **Não podem alterar dados**, ou seja, funções não podem executar comandos como:  

```sql
INSERT
UPDATE
DELETE
```

### 1 - Tipos de Funções

No SQL Server existem dois grandes grupos:

- **Funções de Sistema**
- **Funções Definidas pelo Usuário (User Defined Functions - UDF)**

#### 1.1 - Funções de Sistema
 
São funções fornecidas pelo próprio SQL Server  
Elas executam tarefas comuns como obter data, converter valores ou manipular strings  

Exemplo:

```sql
SELECT GETDATE();
```

Resultado:  
- Retorna a **data e hora atual do servidor SQL Server**

Outros exemplos comuns:  

```sql
LEN()
ISNULL()
CAST()
CONVERT()
```

#### 1.2 - Funções Definidas pelo Usuário (UDF) 

São funções criadas por desenvolvedores ou DBAs  
Elas permitem encapsular lógica reutilizável dentro do banco de dados  

Existem dois tipos principais:
- **Scalar Functions**
- **Table-Valued Functions**

##### 1.2.1 - Scalar Function 

Uma **Scalar Function** retorna **apenas um valor**  
Mesmo que receba vários parâmetros, o retorno sempre será **um único valor escalar**  

**Exemplo Prático — Cálculo de Cubagem (Volume de Carga)**

O exemplo a seguir apresenta um cálculo utilizado em sistemas de logística, como no ERP **Protheus da TOTVS**, para determinar a **cubagem (volume ocupado)** de uma mercadoria  
Esse cálculo pode ser utilizado para verificar se **ainda existe espaço disponível dentro do baú de um caminhão** para armazenar novos itens  

→ O resultado normalmente é expresso em **metros cúbicos (m³)**  
→ A cubagem é obtida através da fórmula:  

```
Altura × Largura × Comprimento
```

**Criamos uma função escalar para cálculo de cubagem**

```sql
CREATE OR ALTER FUNCTION dbo.ufn_CalculateVolume
(
    @Height DECIMAL(10,2),
    @Width  DECIMAL(10,2),
    @Length DECIMAL(10,2)
)
RETURNS DECIMAL(18,2)
AS
BEGIN

    DECLARE @Volume DECIMAL(18,2);

    SET @Volume = @Height * @Width * @Length;

    RETURN @Volume;

END;
```

**Exemplo de uso — Sofá de 2 e 3 lugares**

Suponha que precisa ser feito o carregamento de dois itens em um caminhão:

| Item | Altura | Largura | Comprimento | Peso |
|-----|-----|-----|-----|-----|
| Sofá 2 lugares | 0.90 m | 1.60 m | 0.90 m | 40 kg |
| Sofá 3 lugares | 0.90 m | 2.10 m | 0.90 m | 55 kg |

**Cálculo da cubagem**

```sql
SELECT dbo.ufn_CalculateVolume(0.90,1.60,0.90) AS Sofa2Seats_m3,
       dbo.ufn_CalculateVolume(0.90,2.10,0.90) AS Sofa3Seats_m3;
```

Resultado esperado:

| Item | Volume (m³) |
|-----|-----|
| Sofá 2 lugares | 1.30 |
| Sofá 3 lugares | 1.70 |

Volume total ocupado:

```
3.00 m³
```

Nesse cenário:
- O **peso** é usado para verificar o limite de carga do caminhão
- A **cubagem (m³)** é usada para verificar se **ainda existe espaço físico disponível dentro do baú**

Em sistemas logísticos, ambos os fatores normalmente são analisados:

- **Peso máximo suportado**
- **Volume máximo disponível**

##### 1.2.2 - Table-Valued Function 

Uma **Table-Valued Function** retorna uma **tabela de resultados**  
Ela é semelhante a uma **Stored Procedure**, porém pode ser utilizada diretamente em consultas `SELECT`  
Alguns bancos de dados consideram esse tipo de função como uma **View parametrizada**  

Exemplo:
```sql
CREATE OR ALTER FUNCTION dbo.ufn_GetOrdersByCustomer
(
    @CustomerID INT
)
RETURNS TABLE
AS
RETURN
(
    SELECT
        OrderID,
        OrderDate,
        TotalAmount
    FROM Sales.Orders
    WHERE CustomerID = @CustomerID
);
```

Uso da função:
```sql
SELECT *
FROM dbo.ufn_GetOrdersByCustomer(10);
```

**Nesse caso, a função retorna uma tabela de resultados** 

Boas Práticas:
- Evitar colocar lógica muito pesada em funções
- Preferir **Table-Valued Functions** em cenários que exigem melhor desempenho
- Evitar uso excessivo de **Scalar Functions** dentro de consultas complexas
- Utilizar prefixos para identificar funções de usuário como por exemplo **`ufn`**
 
Exemplo de nomenclatura recomendada:

```sql
dbo.ufn_CalculateDiscount
dbo.ufn_GetOrdersByCustomer
```

**Resumo**

| Tipo | Retorno | Observação |
|-----|-----|-----|
| Sistema | valor ou tabela | fornecida pelo SQL Server |
| Scalar Function | 1 valor | recebe parâmetros e retorna um valor |
| Table-Valued Function | tabela | pode ser usada em consultas SELECT |

---

## Referências

- [Funções definidas pelo usuário](https://learn.microsoft.com/pt-br/sql/relational-databases/user-defined-functions/user-defined-functions?view=sql-server-ver16)
- [CREATE FUNCTION (Transact-SQL)](https://learn.microsoft.com/pt-br/sql/t-sql/statements/create-function-transact-sql?view=sql-server-ver16)
