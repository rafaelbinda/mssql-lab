# A0010 – SQL Data Types

> **Author:** Rafael Binda  
> **Created:** 2026-02-28  
> **Version:** 4.0 

---

## Descrição
Este documento apresenta os tipos de Dados no SQL Server

---

## Hands-on  
[Q0005 - SQL Server String Data Types](../scripts/Q0005-sql-string-data-types.sql)  
[Q0006 - SQL Server Numeric and Bit Data Types](../scripts/Q0006-sql-numeric-and-bit-data-types.sql)  
[Q0007 - SQL Server Date and Time Data Types](../scripts/Q0007-sql-date-and-time-data-types.sql)  
[Q0008 - SQL Server Special Data Types](../scripts/Q0008-sql-special-data-types.sql)

---

## 1 - Tipos de Dados String

No SQL Server, os tipos de dados para texto (caracteres) são divididos em dois grandes grupos:

- **Não-Unicode (baseados em Code Page)**
- **Unicode (UTF-16)**

### 1.1 - Tipos Não-Unicode (Code Page)
→ Armazenam caracteres utilizando **1 byte por caractere** (dependendo da collation)   
→ Indicado quando você tem certeza de que não precisará armazenar múltiplos idiomas  

--- 

**CHAR(n)**
- **Tamanho fixo**
- Sempre reserva exatamente `n` bytes
- Recomendado utilizar `VARCHAR` ao invés de `CHAR` quando o tamanho não for fixo
  
Exemplo:
```sql
CHAR(10)
```
→ Sempre ocupará **10 bytes**, mesmo que seja armazenado 'ABC'  
→ Tamanho máximo: **8.000 bytes**  

**Quando usar?**
→ Quando o tamanho da informação é previsível  
Exemplo: códigos fixos, siglas, UF, etc

---

**VARCHAR(n)**
- **Tamanho variável**
- Armazena apenas o necessário + 2 bytes de controle
- Pode consumir um pouco mais além dos 2 bytes de controle por causa da arquitetura da página

Exemplo:
```sql
VARCHAR(10)
```
- 'ABC' → 3 bytes
- 'ABCDEFGHIJ' → 10 bytes 

**Limites**
- `VARCHAR(n)` → até **8.000 bytes**
- `VARCHAR(MAX)` → até **2 GB**

---

**TEXT** **(Obsoleto)**
- **Tamanho variável**
- Pode armazenar até **2 GB**
- Armazenamento fora da estrutura principal da linha
- Deprecated (não utilizar em novos projetos)
- Recomendado utilizar `VARCHAR` ao invés de `TEXT`

### 1.2 - Tipos Unicode (UTF-16)

→ Armazenam caracteres usando **2 bytes por caractere**   
→ Necessário quando o sistema precisa suportar múltiplos idiomas, acentos especiais, caracteres asiáticos  
→ Tamanho máximo: **4.000 caracteres** (8.000 bytes)

---

**NCHAR(n)**
- **Tamanho fixo**
- Sempre ocupa `n × 2 bytes`

Exemplo:
```sql
NCHAR(10)
```
→ Sempre ocupará **20 bytes**

---
**NVARCHAR(n)**
- **Tamanho variável**
- Armazena:
  - (Quantidade de caracteres × 2 bytes)
  - + 2 bytes de controle

**Limites**
- `NVARCHAR(n)` → até **4.000 caracteres**
- `NVARCHAR(MAX)` → até **2 GB** mas só pode receber até 1GB de dados pois o espaço de armazenamento é de 2 bytes por caracter

Exemplo:

```sql
NVARCHAR(10)
```
- 'ABC' → 6 bytes
- 'ABCDEFGHIJ' → 20 bytes

---

**NTEXT** **(Obsoleto)**
- Tamanho variável
- Até **2 GB**
- Tipo descontinuado
- Não utilizar em novos desenvolvimentos
- Recomendado utilizar `NVARCHAR(MAX)` ao invés de `NTEXT`

---

**Boas Práticas**

- Utilizar `VARCHAR` ao invés de `CHAR` quando possível  
- Utilizar `NVARCHAR` apenas quando houver necessidade real de suporte a múltiplos idiomas  
- Evitar `TEXT` e `NTEXT` (tipos obsoletos)  
- Utilizar `VARCHAR(MAX)` ou `NVARCHAR(MAX)` para grandes volumes de texto
- Considerar o impacto de armazenamento ao utilizar Unicode pois consome o dobro de espaço

---

**Resumo Tipos de Dados String:**

| Tipo | Descrição | Tamanho Máximo | Observação |
|-----|-----------|---------------|-----------|
| CHAR(n) | Texto de tamanho fixo | 8.000 bytes | Sempre ocupa o tamanho definido |
| VARCHAR(n) | Texto de tamanho variável | 8.000 bytes | Usa apenas o espaço necessário |
| VARCHAR(MAX) | Texto longo | ~2 GB | Substitui TEXT |
| NCHAR(n) | Unicode tamanho fixo | 4.000 caracteres | 2 bytes por caractere |
| NVARCHAR(n) | Unicode variável | 4.000 caracteres | Suporte a múltiplos idiomas |
| NVARCHAR(MAX) | Unicode longo | ~2 GB | Substitui NTEXT |


---

## 2 - Tipos de Dados Numéricos

Os tipos de dados numéricos no SQL Server são divididos em quatro categorias principais:

- Inteiros
- Decimais (Precisos)
- Aproximados (Ponto Flutuante)
- Monetários e Lógicos

---

### 2.1 - Tipos Inteiros
- **INT** é o tipo inteiro mais utilizado
- **BIGINT** deve ser usado apenas quando houver necessidade real de grande volume
- Escolher o tipo correto ajuda a economizar espaço e melhorar performance

Armazenam apenas números inteiros (sem casas decimais)

| Tipo      | Tamanho  | Intervalo                                              |
|-----------|----------|--------------------------------------------------------|
| TINYINT   | 1 byte   | 0 a 255 (não aceita negativos)                         |
| SMALLINT  | 2 bytes  | -32.768 a 32.767                                       |
| INT       | 4 bytes  | -2.147.483.648 a 2.147.483.647                         |
| BIGINT    | 8 bytes  | -9.223.372.036.854.775.808 a 9.223.372.036.854.775.807 |

### 2.2 - Tipos Decimais (Precisos)

Usados quando é necessário controle exato sobre casas decimais

- **DECIMAL(p, s)**  
- **NUMERIC(p, s)**

- **NUMERIC** é sinônimo de **DECIMAL**
- Exigem definição de:  
→ **Precisão (p)** → Total de dígitos  
→ **Escala (s)** → Quantidade de dígitos após a vírgula  

Exemplo:
```sql
DECIMAL(16,6)
```
Significa:  
→ Total de 16 dígitos  
→ 6 dígitos reservados para a parte decimal  

Exemplo de valor máximo:
```
9999999999.999999
```
**Armazenamento**
→ O tamanho em bytes varia conforme a precisão:
| Precisão | Bytes |
|----------|-------|
| 1–9      | 5 bytes |
| 10–19    | 9 bytes |
| 20–28    | 13 bytes |
| 29–38    | 17 bytes |

### 2.3 - Tipos Aproximados (Ponto Flutuante)

→ Utilizados para cálculos científicos ou estatísticos onde pequenas imprecisões são aceitáveis  
→ Nunca utilizar **FLOAT** ou **REAL** para:  
- Valores financeiros
- Controle contábil
- Cálculo tributário
- Comparações exatas

---

**FLOAT**
- Armazena valores aproximados
- Pode ter precisão de até 53 bits
- Ocupa 4 bytes (FLOAT(24)) ou 8 bytes (FLOAT(53))

Exemplo:
```sql
FLOAT(53)
```
---

**REAL**
- Equivalente a FLOAT(24)
- Ocupa 4 bytes (32 bits)

--- 
**Por que (24) e (53)?**  
Porque no SQL Server, os tipos **REAL** e **FLOAT** são baseados no padrão `IEEE 754 – Floating Point Arithmetic`  
- 24 bits = precisão de single precision (32 bits)  
- 53 bits = precisão de double precision (64 bits)  
→ REAL = ~7 dígitos decimais de precisão  
→ FLOAT(53) = ~15 a 16 dígitos decimais de precisão  

→ A sintaxe é:
```sql
FLOAT(n)
REAL(n)
```
Onde:  
→ n = número de bits de precisão mantissa  
→ Pode variar de 1 até 53  

**Regra do SQL Server**  
→ O SQL Server simplifica assim:  
|Valor de n |	Armazenamento |	Equivalente |
|-----------|---------------|-------------|
| 1 a 24    | 4 bytes       |    REAL     |
| 25 a 53   | 8 bytes       |  FLOAT(53)  |
 
→ Ou seja:
- FLOAT(24) → ocupa 4 bytes
- FLOAT(53) → ocupa 8 bytes

**E se não declarar nada?**    
- **REAL** (sem parâmetro) = FLOAT(24)  
→ 4 bytes  
→ Máxima precisão possível  
→ Single precision  

- **FLOAT** (sem parâmetro) = FLOAT(53)  
→ O SQL Server assume automaticamente **FLOAT(53)**  
→ 8 bytes  
→ Máxima precisão possível  
→ Double precision  

**Resumo prático**
|Declaração	|       Armazena	  |Bytes        |
|-----------|-------------------|-------------|
|REAL	      | FLOAT(24)	        |4 bytes      |
|FLOAT(24)  |	Single precision	|4 bytes      |
|FLOAT(53)  | Double precision	|8 bytes      |
|FLOAT	    | Assume 53	        |8 bytes      |   

--- 

**Exemplo de Imprecisão com FLOAT e REAL**  
Tipos aproximados como FLOAT e REAL utilizam representação binária (IEEE 754), o que pode gerar pequenas imprecisões em operações matemáticas  

Exemplo:
```sql
DECLARE @a FLOAT = 0.1
DECLARE @b FLOAT = 0.2

IF (@a + @b = 0.3)
    SELECT 'Verdadeiro'
ELSE
    SELECT 'FALSO'

--ou

DECLARE @a REAL = 0.1
DECLARE @b REAL = 0.2

IF (@a + @b = 0.3)
    SELECT 'Verdadeiro'
ELSE
    SELECT 'FALSO'

```  

**Resultado: FALSO**

Por que isso acontece?  

**REAL = FLOAT(24)**    
→ Usa apenas 24 bits de mantissa    
→ Tem cerca de 7 dígitos de precisão  

**FLOAT = FLOAT(53)**  
→ Tem cerca de 15–16 dígitos de precisão  

→ 0.1 e 0.2 não possuem representação binária exata  
→ Eles são armazenados como valores aproximados  
→ A soma resulta em algo próximo de 0.3, mas não exatamente 0.3  
→ Como a comparação exige igualdade exata, **o IF retorna FALSO**  

**O que acontece internamente?**  

**REAL**
→ O valor armazenado para 0.1 vai ser `0.10000000149011612`  
→ O valor armazenado para 0.2 vai ser `0.20000000298023224`  
→ Somando: `0.30000000447034836`  
**Então: 0.30000000447034836 ≠ 0.3**  

**FLOAT**
→ O valor armazenado para 0.1 vai ser `0.1000000000000000055511151231257827021181583404541015625`   
→ O valor armazenado para 0.2 vai ser `0.200000000000000011102230246251565404236316680908203125`    
→ Somando: `0.3000000000000000444089209850062616169452667236328125`    
**Então: 0.3000000000000000444089209850062616169452667236328125 ≠ 0.3**  

**REAL** gera erro maior porque:  
→ Tem menos bits  
→ Arredonda mais cedo  

**FLOAT(53):**  
→ Tem mais bits  
→ Aproxima melhor  
→ Mas ainda não é exato  

**Conclusão técnica**  
→ REAL ≠ FLOAT(53)  
→ Ambos seguem IEEE 754  
→ Ambos são aproximados  
→ FLOAT(53) é mais preciso  
→ Nenhum representa 0.1 exatamente  

### 2.4 - Tipos Monetários

**MONEY**
- Tamanho de armazenamento 8 bytes
- 4 casas decimais fixas
- Intervalo de `-922.337.203.685.477,5808` até `922.337.203.685.477,5807`

---

**SMALLMONEY**
- Tamanho de armazenamento 4 bytes  
- 4 casas decimais fixas  
- Intervalo de `-214.748,3648` até `214.748,3647`

---

**Atenção**
- **MONEY** e **SMALLMONEY** são limitados a 4 casas decimais
- Podem causar arredondamentos indesejados
- **Recomendação:** Preferir `DECIMAL` para valores financeiros

---
**Resumo Tipos de Dados Numéricos:**

| Categoria | Tipo | Bytes | Precisão / Intervalo | Observação |
|---|---|---|---|---|
| Inteiro | TINYINT | 1 | 0 a 255 | Não aceita valores negativos |
| Inteiro | SMALLINT | 2 | -32.768 a 32.767 | Inteiro de pequeno intervalo |
| Inteiro | INT | 4 | -2.147.483.648 a 2.147.483.647 | Inteiro mais utilizado |
| Inteiro | BIGINT | 8 | -9.223.372.036.854.775.808 a 9.223.372.036.854.775.807 | Inteiros muito grandes |
| Decimal (exato) | DECIMAL(p,s) | 5–17 | Precisão definida | Número decimal com precisão e escala |
| Decimal (exato) | NUMERIC(p,s) | 5–17 | Igual ao DECIMAL | Sinônimo de DECIMAL |
| Aproximado | REAL | 4 | ~7 dígitos de precisão | FLOAT(24) |
| Aproximado | FLOAT(53) | 8 | ~15–16 dígitos de precisão | Ponto flutuante |
| Monetário | MONEY | 8 | 4 casas decimais | Pode causar arredondamentos |
| Monetário | SMALLMONEY | 4 | 4 casas decimais | Intervalo menor que MONEY ||

---

## 3 - Tipo Lógico

**BIT**
- Aceita apenas `0`, `1` ou `NULL`
- Utilizado para representar verdadeiro/falso
- O SQL Server otimiza armazenamento de múltiplas colunas BIT internamente
- Ele não ocupa 1 byte por coluna necessariamente pois o SQL Server faz uma otimização interna chamada **Bit Packing**

### O que é Bit Packing?

Em vez de armazenar cada coluna BIT ocupando 1 byte inteiro, o SQL Server:  
→ Agrupa várias colunas BIT  
→ Compacta elas dentro de um único byte  

**Regra de armazenamento**
→ A cada 8 colunas BIT consomem apenas 1 byte  

|Quantidade de colunas BIT|Espaço usado  |
|-------------------------|--------------|
|1 a 8 colunas BIT	      |1 byte        |
|9 a 16 colunas BIT	      |2 bytes       |
|17 a 24 colunas BIT	    |3 bytes       |
|        ...	            |   ...        |


**Exemplo 1**  
```sql
CREATE TABLE Exemplo2 (
    Flag1 BIT,
    Flag2 BIT,
    Flag3 BIT,
    Flag4 BIT,
    Flag5 BIT,
    Flag6 BIT,
    Flag7 BIT,
    Flag8 BIT
)
```
→ Resultado: **Espaço usado 1 Byte**  

**Exemplo 2**  
```sql
CREATE TABLE Exemplo2 (
    Flag1 BIT,
    Flag2 BIT,
    Flag3 BIT,
    Flag4 BIT,
    Flag5 BIT,
    Flag6 BIT,
    Flag7 BIT,
    Flag9 BIT
)
```
→ Resultado: **Espaço usado 2 Bytes**  

**Por que isso acontece?**  
Porque:  
→ Um byte tem 8 bits  
→ Cada coluna BIT usa apenas 1 bit  
→ Então o SQL Server armazena até 8 colunas dentro do mesmo byte  

**Importante**  
Isso só funciona porque:  
→ BIT precisa apenas de 1 ou 0  
→ O engine gerencia isso internamente  
→ Não é visível para o usuário  

---

**E o NULL?**  
Se a coluna permitir NULL:  
→ O controle de NULL é feito na null bitmap da linha  
→ Isso é separado do armazenamento do valor  

**O que é a Null Bitmap?**  
→ Ela existe sempre que a tabela tem pelo menos uma coluna que permite NULL  
→ Toda linha no SQL Server possui uma estrutura interna parecida com isso:  

**| Row Header | Fixed-Length Data | Null Bitmap | Variable-Length Data | Variable Column Offset Array |**  

→ A Null Bitmap é uma área da linha que:   
- Indica quais colunas permitem NULL  
- Indica quais colunas estão NULL naquela linha  

**Se uma coluna BIT permite NULL:**  
Ela tem duas estruturas:  
- O valor 0 ou 1 → armazenado no grupo compactado de bits  
- O estado NULL → armazenado na null bitmap  
Ou seja:  
→ O NULL não ocupa espaço dentro do byte compactado dos BIT  
→ Ele é controlado separadamente  

**Conclusão**  
Quando dizemos `O controle de NULL é feito na null bitmap`, significa que:  
→ O SQL Server não armazena NULL como valor físico  
→ Ele usa um bit indicando ausência de valor  
→ Esse bit fica numa área específica da linha  

**Resumo do Tipo de Dado Lógico**
| Tipo | Valores |
|-----|---------|
| BIT | 0, 1 ou NULL |

---

## 4 - Tipos de Dados de Data e Hora

Os tipos de dados de **data e hora** no SQL Server permitem armazenar datas, horários ou ambos  
Alguns tipos existem desde as primeiras versões do SQL Server, enquanto outros foram introduzidos no **SQL Server 2008**

--- 

**Tipos Disponíveis em Todas as Versões**   

**DATETIME**

- Intervalo de 1753-01-01 até 9999-12-31
- Precisão aproximada: **3,33 milissegundos**
- Armazenamento: **8 bytes**
- Armazena **data e hora juntos**
- Foi o tipo mais utilizado historicamente
- Hoje é recomendado usar **DATETIME2**, que possui maior precisão
  
---

**SMALLDATETIME**

- Intervalo de 1900-01-01 até 2079-06-06
- Precisão: **1 minuto**
- Armazenamento: **4 bytes**
- Armazena **data e hora**
- Segundos e milissegundos são sempre **00**
- Hoje é pouco utilizado devido às limitações de precisão e intervalo

---

**A partir do SQL Server 2008 foram introduzidos novos tipos mais precisos e flexíveis**

**DATE**

- Armazena **apenas a data**
- Intervalo de 0001-01-01 até 9999-12-31
- Precisão: **1 dia**
- Armazenamento: **3 bytes**
- Ideal quando **não é necessário armazenar horário**

---

**TIME**
- Armazena **apenas horário**
- Intervalo de 00:00:00.0000000 até 23:59:59.9999999
- Precisão máxima: **100 nanossegundos**
- Armazenamento: **3 a 5 bytes** dependendo da precisão definida

Exemplo:

```sql
TIME(7)
```
- O número indica a quantidade de casas decimais da fração de segundo

---

**DATETIME2**
- Armazena **data e hora**
- Intervalo de 0001-01-01 até 9999-12-31
- Precisão: até **100 nanossegundos**
- Armazenamento: **6 a 8 bytes** dependendo da precisão

Vantagens sobre `DATETIME`:
- Maior precisão
- Maior intervalo de datas
- Armazenamento mais eficiente
- Tipo recomendado para novos projetos

---

**DATETIMEOFFSET**
- Muito comum em sistemas **distribuídos ou globais**
- Igual ao **DATETIME2**, mas inclui **informação de fuso horário (timezone)**
- Intervalo de timezone: -14:00 até +14:00  

Exemplo:
```sql
2026-03-03 21:30:00 -03:00
```

Utilizado quando é necessário registrar:  
→ Hora  
→ Data  
→ Fuso horário  

---

**Boas Práticas**

- Preferir **DATETIME2** em novos projetos
- Usar **DATE** quando precisar apenas da data
- Usar **TIME** quando precisar apenas do horário
- Usar **DATETIMEOFFSET** quando o sistema precisar armazenar **timezone**
- Evitar **SMALLDATETIME** devido à baixa precisão

---

**Resumo Tipos de Dados de Data e Hora**
| Tipo | Data | Hora | Precisão | Intervalo | Bytes | Observação |
|-----|-----|-----|-----|-----|-----|-----|
| DATE | ✔ | ❌ | 1 dia | 0001-01-01 a 9999-12-31 | 3 | Armazena apenas data |
| TIME | ❌ | ✔ | até 100 ns | 00:00:00 a 23:59:59.9999999 | 3–5 | Armazena apenas hora |
| DATETIME | ✔ | ✔ | 3,33 ms | 1753-01-01 a 9999-12-31 | 8 | Tipo tradicional |
| SMALLDATETIME | ✔ | ✔ | 1 minuto | 1900-01-01 a 2079-06-06 | 4 | Baixa precisão |
| DATETIME2 | ✔ | ✔ | até 100 ns | 0001-01-01 a 9999-12-31 | 6–8 | Recomendado para novos sistemas |
| DATETIMEOFFSET | ✔ | ✔ | até 100 ns | 0001-01-01 a 9999-12-31 | 8–10 | Inclui timezone (-14:00 a +14:00) |


---

## 5 - Tipos de Dados Especiais no SQL Server

O SQL Server possui alguns tipos de dados específicos utilizados para cenários especiais como:

- Identificação global
- Controle de concorrência
- Armazenamento binário
- Estruturas hierárquicas
- Dados espaciais
- Estruturas XML

---

**UNIQUEIDENTIFIER**
- Armazena um identificador único global (**GUID – Global Unique Identifier**)  
- Tamanho: **16 bytes**
- Único mesmo entre servidores diferentes
- Muito utilizado em **replicação**
- Pode ser gerado automaticamente
- O algoritmo utiliza diferentes componentes (tempo, aleatoriedade e outros dados) para garantir unicidade
 
Exemplo:

```sql
SELECT NEWID()
```

Resultado de exemplo:

```
550e8400-e29b-41d4-a716-446655440000
```

**Funções relacionadas:**
| Função | Descrição |
|------|------|
| NEWID() | Gera um GUID aleatório |
| NEWSEQUENTIALID() | Gera GUID sequencial (melhor para índices) |

---

**ROWVERSION**
- Tipo utilizado para **controle de versão de linha**
- Tamanho: **8 bytes**
- Valor gerado automaticamente pelo SQL Server
- Sempre que uma linha é modificada, o valor do `ROWVERSION` é atualizado automaticamente
- É frequentemente utilizado em controle de **concorrência otimista**, onde o valor da versão da linha é verificado durante o UPDATE para garantir que o registro não foi alterado por outra transação desde a última leitura

**O que é concorrência otimista?**  
→ Concorrência acontece quando duas ou mais transações tentam alterar o mesmo registro ao mesmo tempo  
→ Existem duas estratégias comuns:  

|Estratégia|Ideia|
|-----------|------|
|Concorrência pessimista	| Bloqueia o registro para evitar conflitos |
|Concorrência otimista	| Assume que conflitos são raros e verifica apenas no momento do UPDATE |

### Exemplo prático em ordem cronológica 

**1** - Estado inicial da tabela
|Id|Preco|RowVersion| 
|-----------|------|	------|	
1	| 50	| 0x00000000000007D3| 

**2** - Aplicação lê o registro
```sql
SELECT Id, Preco, RowVersion
FROM Produtos
WHERE Id = 1
```

**3** - Resultado:  
→ Preço = `50`  
→ RowVersion = `0x00000000000007D3`  
→ A aplicação guarda esse valor  

**4** - Acontece o UPDATE

```sql
UPDATE Produtos
SET Preco = 100
WHERE Id = 1
AND RowVersion = @RowVersionOriginal
```
→ Nesse momento @RowVersionOriginal = 0x00000000000007D3

**4.1 - Cenário 1** 
Ninguém alterou o registro  
→ O valor ainda é o mesmo      
→ `UPDATE` funciona  
→ E o SQL Server gera um novo `ROWVERSION`  

**4.2 - Cenário 2** 
Outro usuário alterou antes executando um `UPDATE` como este abaixo  
```sql
UPDATE Produtos
SET Preco = 80
WHERE Id = 1
```
**4.2.1** - Agora o ROWVERSION se torna `0x00000000000007D4`  
**4.2.2** - Então, o UPDATE com RowVersion = `0x00000000000007D3` não encontra nenhuma linha e o resultado é `0 rows affected`  
Isso indica conflito, assim, a aplicação pode entender que o registro foi alterado por outro usuário e pode:  
→ Avisar o usuário  
→ Recarregar o registro  
→ Tentar novamente  

**Vantagem desse método**  
- Não precisa bloquear registro
- Funciona bem em sistemas web  
- Muito usado em:  
→ Entity Framework  
→ APIs REST  
→ Sistemas multiusuário  

---

**TIMESTAMP** (nome antigo)
- `TIMESTAMP` é apenas um **sinônimo de ROWVERSION** no SQL Server
- Isso gera confusão porque:  
→ Na **ANSI SQL**, `TIMESTAMP` significa **Data + Hora**  
→ No **SQL Server** ele representa apenas **um número binário sequencial**  
- Por isso a Microsoft recomenda utilizar `ROWVERSION`  

---

**IMAGE** (Deprecated)
- Tipo utilizado para armazenar dados binários grandes
- Capacidade: **até 2 GB**
- A recomendação atual da Microsoft é utilizar `VARBINARY(MAX)`

---

**BINARY**
- Armazena dados binários de **tamanho fixo**
- No exemplo abaixo sempre ocupará exatamente **10 bytes**

```sql
BINARY(10)
```

---

**VARBINARY**
- Armazena dados binários de **tamanho variável**
- Capacidade: até 2 GB
- Versão para grandes dados: `VARBINARY(MAX)`
- Armazena apenas a quantidade real de bytes do valor armazenado, acrescido de 2 bytes utilizados pelo SQL Server para registrar o tamanho do dado
- Muito utilizado para:  
→ imagens  
→ documentos  
→ arquivos  
→ hashes criptográficos  

Exemplo:  

```sql
VARBINARY(100)
```

---

**XML**
- Tipo de dado utilizado para armazenar documentos **XML**
- Capacidade máxima: **2 GB**
- Possui suporte a métodos específicos para manipulação de XML

Exemplo:

```sql
SELECT XmlCol.query('/clientes/cliente')
FROM TabelaXML
```

Vantagens:  
→ Validação de estrutura XML  
→ Métodos nativos para consulta e manipulação  
→ Integração com XQuery  


---

**SQL_VARIANT**
- Tipo que pode armazenar **diferentes tipos de dados**  
- Pode conter valores como:  
→ INT  
→ VARCHAR  
→ DATETIME  
→ DECIMAL  
- Mas cada valor armazena também metadados indicando qual tipo foi utilizado
- **Recomendação:** evitar usar quando possível

Motivos:  
→ Maior complexidade  
→ Impacto em performance  
→ Dificuldade de indexação   

---

**TABLE**
- Tipo utilizado para declarar **variáveis do tipo tabela**
- Funciona de forma semelhante a uma **tabela temporária**, mas:  
→ Existe apenas dentro do escopo da variável  
→ Não é visível fora do batch ou procedure  

Exemplo:  

```sql
DECLARE @TabelaExemplo TABLE (
    Id INT,
    Nome VARCHAR(50)
)
```

---

**HIERARCHYID**  
- Tipo especializado para armazenar **estruturas hierárquicas**  

Exemplo de uso:  
→ Estrutura organizacional  
→ Árvore de categorias  
→ Estruturas de diretórios  

Permite navegar facilmente entre:  
→ pai  
→ filho  
→ descendentes  

---

**GEOMETRY**  
- Tipo utilizado para armazenar **dados espaciais em plano cartesiano (flat-earth)**  
- Usado em cenários como:  
→ mapas locais  
→ plantas  
→ coordenadas em plano  

---

**GEOGRAPHY**  
- Tipo utilizado para armazenar **dados espaciais considerando a curvatura da Terra (round-earth)**  
- Usado para:  
→ coordenadas GPS  
→ latitude e longitude  
→ cálculos de distância geográfica  

---

**Resumo Tipos de Dados Especiais**
| Tipo | Uso principal |
|-----|---------------|
| UNIQUEIDENTIFIER | Identificador único global |
| ROWVERSION | Controle de versão de linha |
| BINARY | Dados binários fixos |
| VARBINARY | Dados binários variáveis |
| XML | Armazenamento de documentos XML |
| SQL_VARIANT | Armazenar múltiplos tipos |
| TABLE | Variável do tipo tabela |
| HIERARCHYID | Estruturas hierárquicas |
| GEOMETRY | Dados espaciais em plano |
| GEOGRAPHY | Dados espaciais globais |

---

## Referências

- [Tipos de dados (Transact-SQL)](https://learn.microsoft.com/pt-br/sql/t-sql/data-types/data-types-transact-sql?view=sql-server-ver16)
- [uniqueidentifier (Transact-SQL)](https://learn.microsoft.com/pt-br/sql/t-sql/data-types/uniqueidentifier-transact-sql?view=sql-server-ver16)

