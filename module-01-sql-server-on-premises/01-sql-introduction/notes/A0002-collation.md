# A0002 – Collation
>
> **Author:** Rafael Binda  
> **Created:** 2026-02-11  
> **Version:** 4.0 

## Descrição:
Documento técnico resumido sobre Code Page, Unicode e Collation no SQL Server, incluindo impactos de desempenho e boas práticas de padronização

## Hands-on
[Q0001 - SQL Collation Hands-on](../scripts/Q0001-collation.sql)  
[INST-Q0001 - SQL Collation DBA Scripts](../../../dba-scripts/SQL-instance-information/INST-Q0001-collation.sql)  

---

## 1. Code Page vs Unicode

O SQL Server armazena dados como valores numéricos, não como letras propriamente ditas.  
Para exibir caracteres, ele utiliza uma tabela de conversão.

Exemplo simplificado:

- 35 → 'a'
- 36 → 'c'

Existem dois principais mecanismos de conversão de caracteres:

- Code Page
- Unicode

---

# Code Page

→ Mais antiga  
→ Cada caractere utiliza 1 byte para ser armazenado  
→ Cada caractere é armazenado como um número  
→ Máximo de 256 caracteres (não cabem todos os caracteres do mundo)  
→ Os primeiros 128 caracteres são sempre idênticos para todas as Code Pages (letras de "a" a "z" maiúsculas e minúsculas sem  
  acentuação, números de "0" a "9")  
→ A faixa acima de 128 contém os caracteres especiais (essa muda de Code Page para Code Page)  
→ Criaram mais de uma Code Page mudando os caracteres de acordo com a língua  
→ Existem varias Code Pages dependendo da lingua  

## Exemplos

→ Code Page 437 → Voltada para Inglês, não possui acentuação  
→ Code Page 850 → Chamada de multilíngue (atende várias línguas inclusive o português), tem acentuação  

## Tipos de dados que usam Code Page

- CHAR  
- VARCHAR  
- TEXT  

Exemplo:

CHAR(10) = 10 bytes  

---

## Problema clássico (B.O.)

→ Na troca de informações entre bancos com Code Pages diferentes há risco de perda de caracteres.  
→ Se eu tenho um banco com Code Page 850 e transportar os dados para outro banco com Code Page 437, o "á" pode virar um caractere  
  estranho (ex: carinha sorrindo), pois o "á" não existe naquela Code Page.

---

## 3 - Unicode

→ Codificação única  
→ Mais recente  
→ Cada caractere utiliza 2 bytes para ser armazenado  
→ Cada caractere é armazenado como número  
→ Máximo de 65.536 caracteres (2^16); com caracteres suplementares (SC) até 1.114.112  
→ Não há risco de perda de acentos, símbolos ou caracteres internacionais  
→ Cabe TODOS os caracteres/símbolos/letras/números existentes no mundo  
→ Unicode é o padrão global  
→ UTF-8 e UTF-16 são formas de representá-lo  
→ UTF-16: Forma de codificação usada por NVARCHAR  
→ UTF-8: Forma de codificação usada por VARCHAR (SQL 2019+)  

## Tipos de dados que usam Unicode

- NCHAR  
- NVARCHAR  
- NTEXT  

Exemplo:  
NCHAR(10) = 20 bytes (ocupa o dobro do espaço comparado ao Code Page)

---

## Impacto

→ Maior volume de dados armazenados  
→ Mais I/O → maior consumo de recursos  
→ Possível impacto em desempenho  

---

## 4 - UTF-8 (SQL Server 2019+)

→ Disponivel a partir do SQL Server 2019  
→ Ele substitui a limitação histórica de Code Page  
→ UTF-8 está relacionado a Unicode, mas não é a mesma coisa que os tipos NCHAR / NVARCHAR  
→ UTF-8 é uma forma de codificação do padrão Unicode  

Exemplo:  
Latin1_General_100_CI_AS_SC_UTF8  

---

## Vantagens

→ Permite armazenar Unicode usando VARCHAR  
→ Economiza espaco comparado ao NVARCHAR  
→ Ideal para sistemas modernos multilíngues  

---

## Diferença prática

### Antes (Unicode clássico)

Ex: NVARCHAR(100)

→ Usa UTF-16  
→ Usa 2 bytes por caractere  
→ Sempre ocupa o dobro do espaço comparado a VARCHAR tradicional  

---

### Agora com UTF-8 usando Collation _UTF8

Ex: VARCHAR(100)

→ Pode armazenar caracteres internacionais  
→ Usa menos espaço para caracteres latinos  
→ Melhor aproveitamento de storage  
→ Reduz IO  
	
---

## 5 - Collation

→ Define as regras de:  
- Armazenamento  
- Manipulação de caracteres  
- Comparacao  
- Ordenacao de caracteres  

Exemplo:  
Latin1_General_CI_AS  
→ Code Page 1252  

→ Quando não está explícito "CP" na Collation, significa que usa a Code Page padrão definida pela padronização ANSI  

---

## Sufixos

- CI = Case Insensitive  
- CS = Case Sensitive  
- AS = Accent Sensitive  
- AI = Accent Insensitive  

---

## 6 - Tipos de Collation

### Windows Collation (recomendado)

→ Sempre que possível utilizar essa (não começa com "SQL_").  

---

### SQL Collation

→ Começa com "SQL_" (ex: SQL_Latin1_General_CP1_CI_AS)  
→ Mantida para compatibilidade com SQL 7, SQL 2000  

---

→ Quanto mais bancos com a mesma Collation, menor a chance de problemas.

---
## 7 - Hierarquia da Collation

Ordem de prioridade:  
Coluna > Banco > Servidor  

---

### 7.1 - Collation do Servidor

→ Definida durante a instalação do SQL Server  
→ Define a Collation padrão do banco model  
→ Impacta a criação do TEMPDB  

---

### 7.2 - Collation do Banco de Dados

→ Por padrão herda a Collation do servidor  
→ Pode ser diferente da Collation do servidor  
→ Define a Collation padrão das colunas criadas sem especificação explícita  

---

### 7.3 - Collation da Coluna

→ Pode ser definida individualmente  
→ Pode ser diferente da Collation do banco  
→ Tem prioridade sobre a Collation do banco na comparação de dados  

---

### 7.4 - Collation na Query (COLLATE)

→ Pode ser definida diretamente na consulta  
→ Tem prioridade máxima  
→ Usada para resolver conflitos entre bancos com Collations diferentes  

---

## 8 - Ordem da Collation

### Ordem Dicionário

Exemplo:  
Latin1_General_CI_AS  

→ Ordena seguindo regras linguísticas (AaBbCc...)  
→ Considera regras culturais do idioma  
→ Mais comum em sistemas corporativos  
→ Dá um pouco mais de trabalho para o SQL Server (overhead), mas hoje temos muito poder computacional  
→ É mais utilizada atualmente  

Exemplo:

Quando fazer um ORDER BY é mais confortável para gente pois vai trazer junto  
"A" maiúsculo, "a" minúsculo, "Á" maiúsculo acentuado, "a" minúsculo acentuado  

---

### Ordem Binária (termina com BIN ou BIN2)

Exemplo:  
Latin1_General_BIN2  

→ Ordena pelos valores numericos internos (código do caractere)  
→ Mais rápida  
→ Sensível a maiusculas/minusculas e acentos  
→ Utilizada em aplicações antigas e cenarios técnicos especificos  
→ Hoje já não é mais tão utilizada para criação de novos sistemas computacionais  

Exemplo:  
O "maiúsculo" tem um número menor de codificação  
O "minúsculo" tem um número um pouco maior de codificação  
Os "acentos" vão estar numa posição maior que 128  

Quando rodar o ORDER BY vai vir primeiro as letras maiúsculas, depois as minúsculas e lá no final o "Á" e "á" com acento agudo,  
então é uma ordem que teoricamente não estamos muito acostumados a usar  

---

No passado fazia sentido na versão 6, 7, SQL 2000 não chegava nem perto do poder computacional atual  
ai o que ocorria é que colocavam essa ordem binária  
mas a aplicação só aceitava caracteres sem acentuação  

Um caso real é o Protheus da TOTVS que eu trabalho desde 2010  
ele até hoje se mantem com Latim1_General_BIN  

---

## 9 - Alteração de Collation

→ Processo extremamente complexo e de alto risco em ambiente de produção  

---

### Se apenas alterar a Collation do banco

→ A alteração NÃO modifica automaticamente as colunas já existentes  
→ Apenas novas colunas criadas herdarão a nova Collation  
→ Colunas antigas continuam com a Collation original  
→ Pode continuar ocorrendo conflito de Collation  

---

### Para alterar corretamente

→ Remover TODAS as constraints  
→ Remover Primary Keys  
→ Remover Foreign Keys  
→ Remover índices  
→ Alterar a Collation das colunas manualmente  
→ Realizar rebuild das tabelas (quando necessário)  
→ Recriar constraints, chaves, índices e demais objetos  

---

### Observação

→ Processo pode gerar indisponibilidade  
→ Deve ser planejado com janela de manutenção  
→ Sempre realizar backup completo antes  

---

## 10 - Problemas clássicos

### Primeiro: Exemplo de Problema com Collation Diferente da TempDB

1 → Banco criado com Code Page 437  
2 → TEMPDB é criado automaticamente com a mesma Collation do Servidor (437)  
3 → Outro banco é criado com Collation 1252 (com suporte a acentos)  
4 → Executo um SELECT grande com ORDER BY  
5 → A consulta necessita ordenar grande volume de dados  
6 → A memória concedida (Memory Grant) não é suficiente para concluir a ordenação totalmente em RAM  
7 → O SQL Server realiza spill para TempDB (grava dados temporários em disco)  
8 → Dados originalmente em 1252 passam a ser manipulados dentro da TempDB 437  
9 → Pode ocorrer conversão implícita de Collation  
10 → Resultado pode apresentar comportamento inesperado ou erro de comparação  
11 → Impacto direto de performance devido ao uso adicional de I/O  

#### Observação importante sobre "falta de memória"

→ Não significa que o servidor ficou sem RAM física  
→ Significa que a memória concedida para execução da consulta não foi suficiente  
→ O SQL Server então utiliza a TempDB para continuar o processamento  
→ Esse processo é chamado de "Spill to TempDB"  
→ Pode ser identificado no plano de execução como Warning de Sort ou Hash  

---

### Segundo: Exemplo de Problema Bancos com Collations Diferentes

1 → Dois bancos possuem Collations diferentes  
2 → Colunas de mesmo tipo (ex: VARCHAR) possuem regras distintas de comparação  
3 → Ao realizar um JOIN entre esses bancos pode ocorrer erro de conflito de Collation  

Exemplo de erro:  
Cannot resolve the collation conflict between 'SQL_Latin1_General_CP1_CI_AS' and 'Latin1_General_CI_AS' in the equal to operation


4 → Necessário tratar explicitamente na query utilizando COLLATE  

Exemplo:

``` Script
ON TabelaA.Coluna COLLATE Latin1_General_CI_AS = TabelaB.Coluna
```


5 → O uso de COLLATE na consulta pode gerar conversão implícita  
6 → Conversão implícita pode impedir uso eficiente de índices  
7 → Impacto direto no desempenho  
8 → Pode causar aumento de CPU e I/O  
9 → Quanto maior o volume de dados, maior o impacto  

---

## 11 - Boas práticas recomendadas

→ Definir Collation padrão no início do projeto  
→ Manter mesma Collation em Servidor, TEMPDB e Bancos  
→ Evitar misturar SQL Collation com Windows Collation  
→ Evitar alterar Collation em ambiente produtivo  
→ Documentar Collation no README do projeto  

---

## Referências

- [Suporte a ordenações e Unicode – SQL Server](https://learn.microsoft.com/pt-br/sql/relational-databases/collations/collation-and-unicode-support?view=sql-server-ver16)
- [sys.fn_helpcollations – listar ordenações disponíveis](https://learn.microsoft.com/pt-br/sql/relational-databases/system-functions/sys-fn-helpcollations-transact-sql?view=sql-server-ver16)
- [Nome de ordenação do Windows](https://learn.microsoft.com/pt-br/sql/t-sql/statements/windows-collation-name-transact-sql?view=sql-server-ver16)
- [Nome de ordenação do SQL Server](https://learn.microsoft.com/pt-br/sql/t-sql/statements/sql-server-collation-name-transact-sql?view=sql-server-ver16)
