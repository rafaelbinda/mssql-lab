# A0013 – SQL Server Programming Objects

> **Author:** Rafael Binda  
> **Created:** 2026-03-17  
> **Version:** 1.0 

---

## Descrição  

Este documento apresenta uma visão geral sobre Stored Procedures no SQL Server, incluindo conceitos, estrutura, parâmetros, controle de fluxo, tratamento de erros, uso de transações e cursores, além de cenários práticos de utilização
 
---

## Hands-on  

[Q0011 - Stored Procedures](../scripts/Q0011-sql-procedures.sql)  
[PROC-Q0001 - Procedures Metadata Objects](../../../dba-scripts/SQL-programming-objects/PROC-Q0001-procedures-metadata.sql) 

---

## SQL Server Stored Procedures 

**O que é uma Stored Procedure?**

- Uma **Stored Procedure** é um conjunto de comandos escritos em **Transact-SQL (T-SQL)** que são armazenados e executados diretamente no **SQL Server**  
- Ela permite encapsular lógica de banco de dados que pode ser executada sempre que necessário  

As stored procedures podem:  
- Executar múltiplas instruções SQL
- Receber **parâmetros**
- Retornar **resultados**
- Executar **operações administrativas ou de manipulação de dados**

---

### 1 - Tipos de Stored Procedures

#### 1.1 - Stored Procedures de Sistema 

São procedures internas do SQL Server usadas para executar **tarefas administrativas**  
Características:  
- Criadas automaticamente pelo SQL Server
- Possuem prefixo **`sp_`**

Exemplo:

```sql
EXEC sp_helpdb;
```

#### 1.2 - Stored Procedures Estendidas 

São procedures desenvolvidas utilizando **linguagem C ou C++** e carregadas no SQL Server como bibliotecas externas  
Características:  
- Possuem prefixo **`xp_`**
- Executam operações externas ao SQL Server
- Mantidas apenas por **compatibilidade**
- **Não é recomendado utilizar**
- Muitas estão **descontinuadas ou desabilitadas por padrão**

Exemplo:

```sql
EXEC xp_cmdshell 'dir';
```

#### 1.3 - Stored Procedures Locais 

São procedures **criadas pelo próprio desenvolvedor ou DBA**  
Características:  
- Criadas dentro de um banco de dados  
- Podem manipular objetos de **outros bancos de dados**  
- Utilizadas para encapsular regras de negócio ou automações  

Exemplo:

```sql
CREATE PROCEDURE dbo.usp_GetCustomers
AS
BEGIN
    SELECT *
    FROM Sales.Customers;
END;
```

Execução:

```sql
EXEC dbo.usp_GetCustomers;
```

#### 1.4 - Stored Procedures com Parâmetros 

Uma stored procedure pode receber parâmetros para tornar sua execução dinâmica

Exemplo:

```sql
CREATE PROCEDURE dbo.usp_GetCustomerById
    @CustomerID INT
AS
BEGIN
    SELECT *
    FROM Sales.Customers
    WHERE CustomerID = @CustomerID;
END;
```

Execução:

```sql
EXEC dbo.usp_GetCustomerById @CustomerID = 10;
```

#### 1.5 - Stored Procedures Temporárias 

São procedures criadas no banco de dados **`tempdb`**  
Características:  
- Quando o nome começa com **`#`**, a procedure é **temporária local**
- Procedures temporárias locais existem apenas na sessão em que foram criadas
- Quando o nome começa com **`##`**, a procedure é **temporária global**
- Procedures temporárias globais podem ser acessadas por outras sessões enquanto existirem
- Esse recurso é pouco usado no dia a dia 

Exemplo de procedure temporária local:

```sql
CREATE PROCEDURE #usp_TempProcedure
AS
BEGIN
    SELECT 'Temporary procedure' AS Message;
END;
```

Exemplo de execução:

```sql
EXEC #usp_TempProcedure;
```

---

**Resumo dos Tipos de Stored Procedures**
| Tipo | Prefixo / Identificação | Uso |
|---|---|---|
| Sistema | `sp_` | Procedures internas do SQL Server usadas para tarefas administrativas |
| Estendida | `xp_` | Procedures desenvolvidas em C/C++ para executar operações externas (mantidas por compatibilidade, não recomendado) |
| Local | definido pelo usuário | Procedures criadas por desenvolvedores ou DBAs para encapsular lógica de negócio |
| Temporária | `#` (local) / `##` (global) | Criadas no banco **tempdb** para uso temporário; pouco utilizadas na prática |

### 2 - Vantagens e Benefícios do Uso de Stored Procedures

Algumas **vantagens** de utilizar **Stored Procedures** no SQL Server incluem:

- Reutilização de código
- Melhor organização da lógica do banco de dados
- Maior controle de permissões
- Redução do tráfego entre aplicação e banco
- Execução diretamente no servidor

---

Além dessas vantagens citadas anteriormente, existem alguns **benefícios práticos** importantes no uso de Stored Procedures, sendo possível listar:

#### 2.1 - Redução no Tráfego de Rede entre a Aplicação e o SQL Server   

Um dos principais benefícios das Stored Procedures é a **redução da quantidade de comunicação entre a aplicação e o banco de dados**  
Exemplo:  

##### 2.1.1 - Cenário sem o uso de Stored Procedure   

Suponhamos que para executar uma regra de negócio na aplicação seja necessário:  
→ Executar **10 consultas SQL**  
→ Analisar os resultados  
→ Realizar um **cálculo para geração de crédito financeiro ao cliente**  

Sem o uso de Stored Procedures, o fluxo seria semelhante a:  
1. A aplicação envia o primeiro `SELECT` ao SQL Server, o comando trafega pela rede e o resultado retorna pela rede  
2. A aplicação envia o segundo `SELECT`, o comando trafega pela rede e o resultado retorna pela rede  
3. A aplicação envia o terceiro `SELECT`, o comando trafega pela rede e o resultado retorna pela rede  
4. O processo continua até o **décimo `SELECT`**, sempre com envio e retorno de dados pela rede
   
**Conclusão:**  
Cada consulta gera **uma nova comunicação entre aplicação e banco de dados** aumentando o tráfego de rede 

##### 2.1.2 - Cenário com o uso de Stored Procedure 

Considere o mesmo cenário **utilizando uma Stored Procedure**
1. Criamos uma **Stored Procedure** contendo todas as consultas e a lógica de cálculo  
2. A aplicação chama a execução da Stored Procedure passando os parâmetros necessários  

```sql
EXEC dbo.usp_CalculateCustomerCredit @CustomerID = 10352;
```
3. No SQL Server, a Stored Procedure executa todas as consultas internamente  
4. Ao final da execução, o SQL Server retorna **apenas o resultado final** para a aplicação
   
**Conclusão:**  
Utilizando Stored Procedure, o tráfego de rede ocorre apenas em dois momentos:  
→ Primeiro: **Envio da chamada da Stored Procedure**  
→ Segundo: **Recebimento do resultado da execução**  

**Resultado:** Isso reduz significativamente a quantidade de comunicação entre aplicação e banco de dados

---

#### 2.2 - Compartilhamento de Código Facilita a Manutenção  

→ Stored Procedures permitem **centralizar regras de negócio no banco de dados**  
→ Isso facilita a manutenção porque:  
- Várias aplicações podem utilizar a mesma procedure
- Alterações na lógica precisam ser feitas apenas em um lugar
- Evita duplicação de código em diferentes aplicações

---

#### 2.3 - Redução da Complexidade no Acesso ao Banco de Dados 

→ Sem Stored Procedures, a aplicação precisaria executar diversas consultas e manipular os resultados  
→ Aplicações podem executar operações complexas com **apenas uma chamada**  

Exemplo:
```sql
EXEC dbo.usp_ProcessCustomerOrder;
```

---

#### 2.4 - Aumento da Segurança 

→ Stored Procedures permitem controlar melhor o acesso aos dados  
→ Em vez de conceder permissões diretamente nas tabelas, é possível conceder permissão apenas na procedure:

Exemplo:
```sql
GRANT EXECUTE ON dbo.usp_GetCustomerData TO AppUser;
```

→ Dessa forma, o usuário pode executar a procedure sem ter acesso direto às tabelas

---

#### 2.5 - Redução do Risco de SQL Injection 

→ SQL Injection ocorre quando um usuário mal-intencionado insere **código SQL malicioso** em entradas da aplicação  

Exemplo conceitual de código malicioso:
```sql
DROP TABLE Customers;
```

→ Se a aplicação construir consultas SQL dinamicamente usando concatenação de strings, esse código pode ser executado  
→ Stored Procedures ajudam a reduzir esse risco quando utilizadas com **parâmetros**  

Exemplo:

```sql
CREATE PROCEDURE dbo.usp_GetCustomer
    @CustomerID INT
AS
BEGIN
    SELECT *
    FROM Customers
    WHERE CustomerID = @CustomerID;
END;
```

→ Como o parâmetro é tratado separadamente do comando SQL, o risco de injeção é reduzido

---

#### 2.6 - Possível Melhoria de Desempenho 

→ Stored Procedures podem melhorar o desempenho porque:  
- São **pré-compiladas e armazenadas no servidor**  
- Utilizam **planos de execução reutilizáveis**  
- Reduzem o tráfego de rede entre aplicação e banco

→ Isso pode resultar em execuções mais eficientes em cenários com alto volume de operações  

---

### 3 - Estrutura de uma Stored Procedure

Uma **Stored Procedure** no SQL Server possui uma estrutura padrão utilizada para definir:  

- O **nome da procedure**
- O **schema ao qual ela pertence**
- Os **parâmetros de entrada ou saída**
- O **bloco de código SQL que será executado**

A criação normalmente utiliza `CREATE OR ALTER`, que permite criar a procedure caso ela não exista ou alterá-la caso já exista

#### 3.1 - Estrutura básica 

```sql
CREATE OR ALTER PROCEDURE SchemaName.ProcedureName
    @Parametro1 INT,
    @Parametro2 VARCHAR(50)
AS
BEGIN
    SET NOCOUNT ON;
    -- Instruções SQL executadas pela procedure
END;
GO
```

#### 3.2 - Elementos da estrutura 

- **CREATE OR ALTER PROCEDURE**  
  Cria uma nova Stored Procedure ou altera uma existente

- **SchemaName**  
  Schema ao qual a procedure pertence (ex: `dbo`, `Sales`)

- **ProcedureName**  
  Nome da Stored Procedure

- **@Parametro**  
  Parâmetros utilizados para receber valores de entrada

- **AS**  
  Indica o início da definição do corpo da procedure

- **BEGIN ... END**  
  Delimita o bloco de instruções SQL executadas pela procedure

 - **SET NOCOUNT ON**  
  Evita o retorno de mensagens como `(X rows affected)`, melhorando a performance e reduzindo tráfego desnecessário entre o SQL Server e a aplicação

- **GO**  
  Separador de batches utilizado por ferramentas como **SSMS** e **Azure Data Studio**  

Stored Procedures podem conter praticamente qualquer instrução SQL, incluindo:  
- Consultas (`SELECT`)
- Manipulação de dados (`INSERT`, `UPDATE`, `DELETE`)
- Estruturas de controle (`IF`, `WHILE`)
- Controle de transações
- Tratamento de erros
- Cursores

Os exemplos práticos estão disponíveis no hands-on **scripts de exemplos** do projeto 

---

**Boas práticas:**  

- Utilizar prefixos próprios como **`usp_`** (user stored procedure)  
- Evitar utilizar **`sp_`** em procedures criadas pelo usuário  

**Exemplo recomendado:**  

```sql
CREATE PROCEDURE dbo.usp_UpdateCustomerStatus
AS
BEGIN
    -- lógica da procedure
END;
```
---

### 4 - Parâmetros em Stored Procedures

- Os **parâmetros** permitem que uma Stored Procedure receba valores externos no momento da execução, tornando-a reutilizável e dinâmica  
- Eles funcionam de forma semelhante a parâmetros em funções de linguagens de programação  

#### 4.1 - Declaração de parâmetros   
Os parâmetros são definidos logo após o nome da procedure:

```sql
@Parametro1 INT,
@Parametro2 VARCHAR(50)
```

Cada parâmetro possui:

- Um nome (sempre iniciado com `@`)
- Um tipo de dado
- Opcionalmente um valor padrão

---

#### 4.2 - Tipos de parâmetros 

##### 4.2.1 - Parâmetros de entrada (Input)  
São utilizados para enviar valores para dentro da procedure
- São os mais comuns
- Podem ser utilizados em filtros, cálculos e regras de negócio

##### 4.2.2 - Parâmetros com valor padrão   
Permitem tornar o parâmetro opcional na execução  

```sql
@Status VARCHAR(20) = 'Active'
```

Se nenhum valor for informado, o valor padrão será utilizado  

---

#### 4.3 - Execução com parâmetros 
Os parâmetros podem ser passados de duas formas:  

##### 4.3.1 - Forma posicional 

```sql
EXEC SchemaName.ProcedureName 10, 'Value'
```

##### 4.3.2 - Forma nomeada (recomendada) 

```sql
EXEC SchemaName.ProcedureName
    @Parametro1 = 10,
    @Parametro2 = 'Value'
```

A forma nomeada é mais segura e legível, principalmente quando há muitos parâmetros

---

**Boas práticas**

- Utilizar nomes claros e descritivos para os parâmetros
- Evitar excesso de parâmetros na mesma procedure
- Preferir sempre a execução com parâmetros nomeados
- Definir valores padrão quando fizer sentido

---

### 5 - Parâmetros de Saída (OUTPUT)

Os **parâmetros de saída (OUTPUT)** permitem que uma Stored Procedure retorne valores para quem a executou  
Diferente de um `SELECT`, que retorna um conjunto de resultados, os parâmetros OUTPUT retornam **valores específicos através de variáveis**

---

#### 5.1 - Declaração de parâmetros OUTPUT  
Para definir um parâmetro de saída, utilizamos a palavra-chave `OUTPUT`:  
```sql
@TotalPedidos INT OUTPUT
```

**Como funciona**
- A procedure recebe parâmetros de entrada normalmente
- Durante a execução, valores são atribuídos aos parâmetros OUTPUT
- Esses valores podem ser recuperados após a execução

---

#### 5.1.1 - Execução com OUTPUT

Para capturar o valor de saída, é necessário:
1. Declarar uma variável
2. Executar a procedure informando `OUTPUT`
3. Utilizar a variável após a execução

```sql
DECLARE @Resultado INT;

EXEC SchemaName.ProcedureName
    @Parametro1 = 10,
    @TotalPedidos = @Resultado OUTPUT;
```

---

**Características importantes**

- É obrigatório utilizar a palavra `OUTPUT` tanto na definição quanto na execução
- Permite retornar múltiplos valores de forma estruturada
- Muito utilizado para status, contadores e valores calculados

---

#### 5.1.2 - OUTPUT vs SELECT

| Característica | OUTPUT | SELECT |
|---|---|---|
| Retorna valor único | ✔ | ✔ |
| Retorna múltiplas linhas | ✖ | ✔ |
| Pode ser capturado em variável | ✔ | ✖ (diretamente) |
| Usado em integração com aplicações | ✔ | ✔ |

**Comparação simplificada**

|Situação	|Melhor usar|
|---------|-----------|
|Listar pedidos de um cliente|	SELECT|
|Retornar total de pedidos|	OUTPUT|
|Retornar ID de um INSERT|	OUTPUT|
|Retornar dados para relatório|	SELECT|

---

**Boas práticas**

- Utilizar OUTPUT para retornar valores simples (status, totais, identificadores)
- Evitar retornar grandes volumes de dados via OUTPUT
- Nomear claramente os parâmetros de saída (ex: `@OUT_Status`, `@OUT_Total`)

Em muitos cenários, é comum combinar:
- parâmetros de entrada
- parâmetros OUTPUT
- e `SELECT`

Isso permite que a Stored Procedure seja flexível tanto para aplicações quanto para consultas diretas

---

### 6 - Tratamento de Erros (TRY...CATCH)

O **tratamento de erros** em Stored Procedures permite capturar e tratar falhas que ocorrem durante a execução, evitando comportamentos inesperados na aplicação
No SQL Server, isso é feito utilizando a estrutura `TRY...CATCH`

Estrutura básica:

```sql
BEGIN TRY
    -- Código que pode gerar erro
END TRY
BEGIN CATCH
    -- Código executado em caso de erro
END CATCH
```

**Como funciona**
- O bloco `TRY` contém as instruções que podem gerar erro
- Se ocorrer uma falha durante a execução, o fluxo é transferido para o `CATCH`
- O bloco `CATCH` permite tratar o erro de forma controlada

---

#### 6.1 - Funções de erro disponíveis

Dentro do `CATCH`, é possível obter detalhes do erro utilizando funções específicas:

- `ERROR_NUMBER()` → código do erro
- `ERROR_MESSAGE()` → mensagem descritiva
- `ERROR_SEVERITY()` → nível de gravidade
- `ERROR_STATE()` → estado do erro
- `ERROR_LINE()` → linha onde ocorreu o erro
- `ERROR_PROCEDURE()` → nome da procedure

---

**Boas práticas**  
- Utilizar `TRY...CATCH` em operações críticas
- Retornar mensagens claras para facilitar troubleshooting
- Evitar que erros sejam propagados sem controle para a aplicação

---

#### 6.2 - Integração com transações

O tratamento de erros é frequentemente utilizado junto com transações:  
- Em caso de sucesso → `COMMIT`
- Em caso de erro → `ROLLBACK`

→ Isso garante a consistência dos dados  

---

**Observação**  
Nem todos os erros são capturados pelo `TRY...CATCH`, especialmente:  
- Erros de compilação
- Erros de sintaxe

O `TRY...CATCH` atua principalmente em **erros de execução (runtime)**

---

### 7 - Transações em Stored Procedures

As **transações** garantem que um conjunto de operações SQL seja executado de forma **atômica**, ou seja, todas as operações são concluídas com sucesso ou nenhuma delas é aplicada  
Em Stored Procedures, as transações são utilizadas para manter a **consistência dos dados** durante operações críticas  

---

#### 7.1 - Conceitos básicos
Uma transação é controlada por três comandos principais:  
- `BEGIN TRAN` → inicia a transação
- `COMMIT` → confirma as alterações
- `ROLLBACK` → desfaz as alterações

**Como funciona**
- A transação é iniciada com `BEGIN TRAN`
- Todas as operações executadas após isso fazem parte da transação
- Se tudo ocorrer corretamente, utiliza-se `COMMIT`
- Em caso de erro, utiliza-se `ROLLBACK`

---

#### 7.2 - Uso com Stored Procedures
Dentro de uma Stored Procedure, as transações são utilizadas para garantir que múltiplas operações relacionadas sejam tratadas como uma única unidade lógica  
Isso é comum em cenários como:  
- Inserção de pedidos e itens
- Atualização de múltiplas tabelas relacionadas
- Operações financeiras

---

#### 7.3 - Integração com tratamento de erros  
O uso de transações normalmente é combinado com `TRY...CATCH`:  
- `TRY` → executa a transação
- `CATCH` → realiza o `ROLLBACK` em caso de erro

Isso evita inconsistências no banco de dados  

---

**Boas práticas**  
- Sempre utilizar `COMMIT` ou `ROLLBACK` para finalizar a transação  
- Evitar transações longas (podem causar bloqueios)  
- Utilizar transações apenas quando necessário  
- Combinar transações com tratamento de erros  

---

**Observação**  
Enquanto uma transação estiver aberta:  
- Alterações podem não estar visíveis para outras sessões (dependendo do isolamento)
- Recursos podem ficar bloqueados

---
**Dica de DBA (importante pra vida real)**  
Se esquecer de dar COMMIT ou ROLLBACK, você pode:  
- Gerar locks
- Travar sessões
- Causar problemas de performance

---

### 8 - Controle de Fluxo (Control-of-Flow)

O **controle de fluxo** permite definir a ordem de execução das instruções dentro de uma Stored Procedure, possibilitando a criação de regras e decisões lógicas  
Essas estruturas são utilizadas para controlar o comportamento da procedure com base em condições  

---

#### 8.1 - Principais estruturas

##### IF...ELSE
Permite executar diferentes blocos de código com base em uma condição  
- `IF` → executa quando a condição é verdadeira
- `ELSE` → executa quando a condição é falsa

##### WHILE
Permite repetir um bloco de código enquanto uma condição for verdadeira  
- Utilizado para loops
- Deve ser usado com cuidado para evitar execuções longas

##### BREAK
Interrompe a execução de um loop (`WHILE`) antes que a condição seja encerrada naturalmente  

##### CONTINUE
Interrompe a iteração atual do loop e passa para a próxima execução  

---
**Como funciona**
- As estruturas de controle de fluxo permitem que a procedure tome decisões
- São utilizadas para implementar regras de negócio diretamente no banco
- Podem ser combinadas entre si para criar lógicas mais complexas

---
**Boas práticas**
- Evitar lógica excessivamente complexa dentro da procedure
- Utilizar nomes de variáveis claros para facilitar entendimento
- Garantir que loops (`WHILE`) tenham condição de parada adequada
- Preferir operações baseadas em conjuntos (set-based) sempre que possível

---

**Observação**  
O SQL Server é otimizado para trabalhar com **operações em conjunto (set-based)**  
O uso excessivo de controle de fluxo pode impactar a performance quando comparado a consultas bem estruturadas  

---

### 9 - Cursores (Cursors)

Os **cursores** permitem percorrer um conjunto de resultados linha por linha dentro de uma Stored Procedure  
Eles são utilizados quando é necessário executar operações que não podem ser facilmente realizadas com consultas baseadas em conjunto (set-based)  

---

#### 9.1 - Conceito  
Um cursor funciona como um ponteiro sobre o resultado de uma consulta, permitindo acessar cada linha individualmente  
O fluxo básico de uso é:  
- Declarar o cursor
- Abrir o cursor
- Ler os dados (FETCH)
- Fechar o cursor
- Liberar os recursos

---

#### 9.2 - Estrutura geral

```sql
DECLARE CursorName CURSOR FOR
    -- consulta

OPEN CursorName;

FETCH NEXT FROM CursorName INTO -- variáveis

WHILE @@FETCH_STATUS = 0
BEGIN

    -- processamento linha a linha

    FETCH NEXT FROM CursorName INTO -- variáveis

END

CLOSE CursorName;
DEALLOCATE CursorName;
```

---

#### 9.3 - Como funciona  
- O cursor executa uma consulta e armazena o resultado
- Cada `FETCH` retorna uma linha
- `@@FETCH_STATUS` indica o status da leitura
- O loop continua até que não existam mais linhas

---

#### 9.4 - Tipos comuns de cursor
- `FAST_FORWARD` → somente leitura e avanço para frente (mais leve e performático)  
- `STATIC` → cria uma cópia dos dados  
- `DYNAMIC` → reflete alterações nos dados durante a execução  

---

**Boas práticas**  
- Utilizar cursores apenas quando necessário  
- Sempre fechar (`CLOSE`) e desalocar (`DEALLOCATE`) o cursor  
- Preferir cursores `FAST_FORWARD` quando possível  
- Evitar cursores em grandes volumes de dados  

---

**Observação**
Cursores são geralmente mais lentos do que operações baseadas em conjunto (set-based), pois processam dados linha por linha  
Em vez de cursores, sempre que possível, deve-se priorizar:  
- `SELECT`
- `JOIN`
- `GROUP BY`

---

**Importância**
Os cursores são úteis para:
- Processamento linha a linha
- Integração com regras complexas
- Cenários onde lógica procedural é necessária  
Mesmo assim, devem ser utilizados com cautela devido ao impacto em performance

---

## Referências

- [Stored Procedures (Mecanismo de Banco de Dados)](https://learn.microsoft.com/pt-br/sql/relational-databases/stored-procedures/stored-procedures-database-engine?view=sql-server-ver16)
- [CREATE PROCEDURE (Transact-SQL)](https://learn.microsoft.com/pt-br/sql/t-sql/statements/create-procedure-transact-sql?view=sql-server-ver16)
