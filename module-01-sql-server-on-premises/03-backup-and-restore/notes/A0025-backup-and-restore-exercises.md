# A0025 – Backup and Restore Exercises  
> **Author:** Rafael Binda  
> **Created:** 2026-04-12  
> **Version:** 2.0  

---

## Descrição  

Este material apresenta cenários práticos de backup e restore no SQL Server com foco em situações reais de falha  
O objetivo é desenvolver raciocínio técnico para recuperação de dados considerando Recovery Model cadeia de backups estratégia adotada e impacto operacional em caso de incidente  
Todos os cenários seguem o mesmo padrão com estratégia linha do tempo sequência correta de restore resultado esperado e análise final  

---
## Hands-on

[Q0022 - Tail Log Backup (NO_TRUNCATE)](../scripts/Q0022-sql-tail-log-backup.sql)

---

## Cenário 1 - Estratégia de backup FULL  

### Estratégia  

- RECOVERY MODEL SIMPLE  
- BACKUP FULL diário às 18:00  
- Utilizando banco de teste desenvolvimento e data warehouse  

### Linha do tempo  

- Segunda-feira 18:00 backup FULL  
- Terça-feira 11:30 falha com perda do arquivo MDF  

### Como recuperar o banco?  

1 - Restore do backup FULL de segunda-feira 18:00 WITH RECOVERY  
2 - Não há possibilidade de aplicar backup de LOG pois o banco está em RECOVERY MODEL SIMPLE  

### Resultado  

- O banco retorna ao último ponto consistente salvo no backup FULL de segunda-feira 18:00  
- Perdeu todos os dados de segunda-feira 18:00:01 até terça-feira 11:30  

### Análise final  

- O RECOVERY MODEL SIMPLE não permite backup de LOG  
- Isso significa que toda a recuperação fica limitada ao último backup FULL disponível  
- Essa estratégia pode ser aceitável em ambientes onde alguma perda de dados é tolerável  
- Em contrapartida o RPO tende a ser maior porque qualquer alteração após o último FULL será perdida em caso de falha  
- O ponto principal desse cenário é entender que não existe alternativa para recuperar o intervalo após o último FULL quando não existe cadeia de LOG  

---

## Cenário 2 - Estratégia de backup FULL e LOG  

### Estratégia  

- RECOVERY MODEL FULL  
- BACKUP FULL diário às 18:00  
- BACKUP LOG diário em intervalos regulares  
- Utilizando banco pequeno em ambiente de produção  

### Linha do tempo  

- Segunda-feira 18:00 backup FULL  
- Terça-feira backups LOG às 08:00 09:00 e 10:00  
- Terça-feira 11:30 falha com perda do arquivo MDF  

### Como recuperar o banco?  

1 - Realizar BACKUP do TAIL LOG WITH NO_TRUNCATE  
2 - Restore do backup FULL de segunda-feira 18:00 WITH NORECOVERY  
3 - Restore do backup LOG de terça-feira 08:00 WITH NORECOVERY  
4 - Restore do backup LOG de terça-feira 09:00 WITH NORECOVERY  
5 - Restore do backup LOG de terça-feira 10:00 WITH NORECOVERY  
6 - Restore do TAIL LOG WITH RECOVERY  

### Resultado  

- O banco é recuperado até o momento mais próximo possível da falha  
- A perda de dados é mínima desde que o TAIL LOG seja executado com sucesso  

### Análise final  

- Esse é um dos cenários clássicos de recuperação em ambiente de produção  
- O TAIL LOG é essencial para preservar as transações ocorridas após o último backup LOG regular  
- A ordem cronológica do restore deve ser rigorosamente respeitada  
- Se o TAIL LOG falhar a recuperação ficará limitada ao último backup LOG íntegro disponível  
- Esse modelo oferece melhor RPO do que SIMPLE porque permite recuperar praticamente até o ponto da falha  

---

## Cenário 3 - Estratégia de backup FULL, DIFFERENTIAL e LOG  

### Estratégia  

- RECOVERY MODEL FULL  
- BACKUP FULL semanal aos domingos às 10:00  
- BACKUP DIFFERENTIAL diário às 18:00  
- BACKUP LOG diário a cada 1 hora das 08:00 às 17:00  
- Utilizando bancos médios e grandes em ambiente de produção  

### Linha do tempo  

- Domingo 10:00 backup FULL  
- Segunda-feira backups LOG às 08:00 09:00 10:00 11:00 12:00 13:00 14:00 15:00 16:00 e 17:00  
- Segunda-feira 18:00 backup DIFFERENTIAL  
- Terça-feira backups LOG às 08:00 09:00 10:00 11:00 12:00 13:00 14:00 15:00 16:00 e 17:00  
- Terça-feira 18:00 backup DIFFERENTIAL  
- Quarta-feira backups LOG às 08:00 09:00 10:00 e 11:00  
- Quarta-feira 11:30 falha com perda do arquivo MDF  

### Como recuperar o banco?  

1 - Realizar BACKUP do TAIL LOG WITH NO_TRUNCATE  
2 - Restore do backup FULL de domingo 10:00 WITH NORECOVERY  
3 - Restore do backup DIFFERENTIAL de terça-feira 18:00 WITH NORECOVERY  
4 - Restore do backup LOG de quarta-feira 08:00 WITH NORECOVERY  
5 - Restore do backup LOG de quarta-feira 09:00 WITH NORECOVERY  
6 - Restore do backup LOG de quarta-feira 10:00 WITH NORECOVERY  
7 - Restore do backup LOG de quarta-feira 11:00 WITH NORECOVERY  
7 - Restore do TAIL LOG WITH RECOVERY  

### Resultado  

- O banco é recuperado com perda mínima de dados  
- O uso do DIFFERENTIAL reduz a quantidade de LOGs necessários para o restore  
- O tempo total de recuperação é menor do que em uma estratégia baseada apenas em FULL e LOG  

### Análise final  

- O backup DIFFERENTIAL contém todas as alterações desde o último FULL base  
- Isso reduz significativamente o trabalho de restore porque elimina a necessidade de aplicar todos os LOGs desde o FULL de domingo  
- A sequência correta é FULL base seguido do último DIFFERENTIAL válido depois os LOGs posteriores a esse DIFFERENTIAL e por fim o TAIL LOG  
- Essa estratégia oferece bom equilíbrio entre RPO e RTO em bancos médios e grandes  
- O ponto central aqui é entender que o DIFFERENTIAL acelera a recuperação sem substituir a necessidade dos LOGs posteriores a ele  

---

## Cenário 4 - Estratégia de backup FILE e FILEGROUP  

### Estratégia  

- RECOVERY MODEL FULL  
- BACKUP FULL semanal aos domingos às 10:00  
- BACKUP FILE ou FILEGROUP diário às 18:00  
- BACKUP LOG diário a cada 1 hora das 08:00 às 17:00  
- Utilizando banco grande com dados distribuídos entre múltiplos arquivos ou FILEGROUPS  

### Linha do tempo  

- Domingo 10:00 backup FULL  
- Domingo backups LOG às 09:00 10:00 11:00 12:00 13:00 14:00 15:00 16:00 e 17:00  
- Segunda-feira 18:00 backup do DATA FILE 1  
- Segunda-feira backups LOG às 09:00 10:00 11:00 12:00 13:00 14:00 15:00 16:00 e 17:00  
- Terça-feira 18:00 backup do DATA FILE 2  
- Terça-feira backups LOG às 09:00 10:00 11:00 12:00 13:00 14:00 15:00 16:00 e 17:00  
- Quarta-feira backups LOG às 08:00 09:00 10:00 e 11:00  
- Quarta-feira 11:30 falha com perda do MDF do DATA FILE 2  

### Como recuperar o banco?  

1 - Realizar BACKUP do TAIL LOG WITH NO_TRUNCATE  
2 - Restore do backup do DATA FILE 2 de terça-feira 18:00 WITH NORECOVERY  
3 - Restore do backup LOG de quarta-feira 08:00 WITH NORECOVERY  
4 - Restore do backup LOG de quarta-feira 09:00 WITH NORECOVERY  
5 - Restore do backup LOG de quarta-feira 10:00 WITH NORECOVERY  
6 - Restore do backup LOG de quarta-feira 11:00 WITH NORECOVERY   
7 - Restore do TAIL LOG WITH RECOVERY  

### Resultado  

- Apenas o arquivo afetado é restaurado  
- O restante do banco não precisa passar por restore completo  
- O tempo de indisponibilidade tende a ser menor do que em um restore integral do banco  

### Análise final  

- Esse cenário mostra uma estratégia avançada para ambientes de grande porte  
- FILE e FILEGROUP backup permitem recuperação granular de partes específicas do banco  
- O restore do arquivo ou FILEGROUP afetado precisa ser complementado pela cadeia de LOG para que ele alcance o mesmo ponto transacional do restante do banco
- O banco pode permanecer online durante o processo de restore (piecemeal restore)  
- Apenas os objetos e dados localizados no FILE ou FILEGROUP em processo de recuperação ficam inacessíveis  
- FILEGROUPS já recuperados permanecem disponíveis e podem ser acessados normalmente  
- Essa abordagem é muito útil quando o volume total do banco é grande e um restore completo seria demorado demais  
- O ponto principal é entender que a arquitetura física do banco influencia diretamente a estratégia de recuperação  

---

## Cenário 5 - Estratégia com backup FULL fora do padrão  

### Estratégia  

- RECOVERY MODEL FULL  
- BACKUP FULL diário às 18:00  
- BACKUP LOG diário a cada 1 hora das 08:00 às 17:00  
- Ambiente de produção com execução manual de backup FULL fora da rotina oficial  

### Linha do tempo  

- Segunda-feira 18:00 backup FULL  
- Terça-feira backup LOG às 08:00  
- Terça-feira backup LOG às 09:00  
- Terça-feira backup LOG às 10:00  
- Terça-feira 11:00 backup FULL executado manualmente fora do padrão  
- Terça-feira 11:30 falha com perda do arquivo MDF  

### Como recuperar o banco?  

1 - Realizar BACKUP do TAIL LOG WITH NO_TRUNCATE  
2 - Restore do backup FULL de terça-feira 11:00 WITH NORECOVERY  
3 - Restore do TAIL LOG WITH RECOVERY  

### Resultado  

- O banco pode ser recuperado utilizando o FULL mais recente  
- O restore tende a ser mais rápido do que começar a partir do FULL de segunda-feira  
- A perda de dados é mínima desde que o TAIL LOG seja obtido com sucesso  

### Análise final  

- Um backup FULL adicional não quebra a cadeia de LOG, ou seja, os backups de LOG continuam válidos e podem ser aplicados normalmente no processo de restore  
- Isso acontece porque a sequência de LSN utilizada pelos LOGs permanece contínua, independentemente da execução de novos backups FULL  
- Nesse cenário específico não existem LOGs regulares após o FULL manual, pois a falha ocorreu logo em seguida, limitando a sequência de restore ao próprio FULL e ao eventual TAIL LOG  
- O FULL manual cria um novo ponto possível de início para a recuperação, podendo reduzir o tempo total de restore em comparação com o uso de um FULL mais antigo  
- Esse comportamento é importante para entender que a execução de FULL fora da estratégia não invalida a cadeia de LOG, mas pode alterar o ponto mais eficiente para iniciar o processo de recuperação  
- Em ambientes de produção, a execução de FULL fora do padrão definido pode gerar confusão operacional e aumentar o risco de erro durante o restore se não houver controle e documentação adequados  

---

## Cenário 6 - Mudança de SIMPLE para FULL pouco antes da falha  

### Estratégia  

- Banco operava inicialmente em RECOVERY MODEL SIMPLE  
- BACKUP FULL diário às 18:00  
- A equipe alterou o banco para RECOVERY MODEL FULL durante o horário de operação  
- Após a alteração foram executados backups LOG  
- Nenhum novo backup FULL foi executado após a mudança do modelo  

### Linha do tempo  

- Segunda-feira 18:00 backup FULL  
- Terça-feira 07:30 banco ainda operando em RECOVERY MODEL SIMPLE  
- Terça-feira 08:00 ALTER DATABASE para FULL  
- Terça-feira backup LOG às 09:00  
- Terça-feira backup LOG às 10:00  
- Terça-feira backup LOG às 11:00  
- Terça-feira 11:30 falha com perda do arquivo MDF  

### Como recuperar o banco?  

1 - Restore do backup FULL de segunda-feira 18:00 WITH RECOVERY  
2 - Os backups LOG de terça-feira 09:00 10:00 e 11:00 não podem ser utilizados para restore  
3 - Não existe possibilidade de recuperar as alterações feitas após segunda-feira 18:00 porque a cadeia de LOG não foi iniciada corretamente  

### Resultado  

- O banco retorna ao estado do backup FULL de segunda-feira 18:00  
- Perdeu todos os dados entre segunda-feira 18:00:01 e terça-feira 11:30  

### Análise final  

- Alterar o banco de SIMPLE para FULL não inicializa automaticamente uma cadeia válida de LOG para restore  
- Após a mudança para FULL é obrigatório executar um novo backup FULL para iniciar corretamente essa cadeia  
- Os backups LOG até podem existir fisicamente mas não serão utilizáveis até que a cadeia tenha sido iniciada  
- O ponto central do cenário é entender que o Recovery Model sozinho não basta sem a sequência correta de backups  
- O cenário também mostra que executar backups LOG logo após a mudança do modelo não resolve o problema sem o novo FULL base  

---

## Cenário 7 - Banco em estado RESTORING após restore incompleto ou interrompido  

### Estratégia  

- Restore do backup FULL foi iniciado com NORECOVERY  
- O processo pode ter sido deixado em espera propositalmente para aplicação de DIFFERENTIAL e LOG  
- Ou pode ter sido interrompido no meio por falha operacional, cancelamento (kill) ou incidente no ambiente  

### Linha do tempo  

- Segunda-feira 18:00 backup FULL disponível para restore  
- Terça-feira 08:00 restore do backup FULL iniciado com NORECOVERY  
- Terça-feira 08:20 o restore FULL foi concluído com sucesso mas o banco permaneceu em RESTORING aguardando próximos backups  
- Terça-feira 09:00 existia a possibilidade de aplicar backup DIFFERENTIAL ou LOG subsequente caso fizessem parte da estratégia  
- Terça-feira 09:15 o ambiente foi analisado e o banco permaneceu indisponível em RESTORING  
- Em um segundo cenário possível o processo pode ter sido interrompido no meio antes do término correto do restore FULL (kill) 

### Como recuperar o banco?  

Caso 1 - O restore FULL foi concluído corretamente e o banco apenas aguarda a continuação da cadeia  

1 - Se não houver mais backups a serem aplicados executar `RESTORE DATABASE <databasename> WITH RECOVERY`  
2 - Se ainda houver backup DIFFERENTIAL ou LOG pendentes aplicar os backups na ordem correta todos WITH NORECOVERY  
3 - Finalizar com RECOVERY apenas no último passo da sequência  

Caso 2 - O restore FULL foi interrompido no meio e não existe garantia de consistência  

1 - Reiniciar o processo de restore desde o backup FULL  
2 - Reaplicar toda a cadeia necessária de backups subsequentes com NORECOVERY  
3 - Finalizar com RECOVERY somente ao término completo da sequência  

### Resultado  

- No caso válido o banco pode ser colocado online com RECOVERY ou após aplicação da cadeia restante  
- No caso interrompido o caminho seguro é reiniciar todo o processo de restore  
- O banco permanece indisponível enquanto estiver em NORECOVERY ou em restore inconsistente  
- A decisão correta depende de saber se o restore anterior chegou ou não ao final com sucesso  

### Análise final  

- Esse cenário exige distinguir duas situações que podem parecer iguais visualmente mas são tecnicamente diferentes  
- Um banco em RESTORING pode estar apenas aguardando a continuação normal do restore ou pode estar em estado não confiável por interrupção do processo  
- Executar RECOVERY cedo demais encerra a sequência e impede aplicação de backups adicionais podendo causar perda de dados  
- Por outro lado tentar continuar um restore interrompido no meio pode ser inseguro porque não há garantia de consistência  
- O ponto principal é avaliar se o restore anterior foi concluído com sucesso até o final ou não antes de decidir o próximo passo  
- Esse é um cenário de nível hard porque depende de interpretação correta do estado do banco e da fase em que o restore se encontra  

---

## Cenário 8 - Backup DIFFERENTIAL aparentemente válido mas falha no restore  

### Estratégia  

- RECOVERY MODEL FULL  
- BACKUP FULL semanal aos domingos às 10:00  
- BACKUP DIFFERENTIAL diário às 18:00  
- BACKUP LOG diário a cada 1 hora das 08:00 às 17:00  
- Um backup FULL adicional foi executado fora da rotina oficial sem que a equipe considerasse o impacto sobre os diferenciais  

### Linha do tempo  

- Domingo 10:00 backup FULL  
- Segunda-feira 18:00 backup DIFFERENTIAL  
- Terça-feira 18:00 backup DIFFERENTIAL  
- Quarta-feira 08:00 backup LOG  
- Quarta-feira 09:00 backup LOG  
- Quarta-feira 10:00 backup FULL executado manualmente fora do padrão  
- Quarta-feira 11:00 backup LOG  
- Quarta-feira 11:30 falha com perda do arquivo MDF  

### Como recuperar o banco?  

1 - Realizar BACKUP do TAIL LOG WITH NO_TRUNCATE  
2 - Restore do backup FULL de quarta-feira 10:00 WITH NORECOVERY  
3 - Restore do backup LOG de quarta-feira 11:00 WITH NORECOVERY  
4 - Restore do TAIL LOG WITH RECOVERY  

### Resultado  

- O backup DIFFERENTIAL de terça-feira 18:00 não pode ser utilizado  
- A recuperação continua possível mas agora precisa começar pelo FULL mais recente  
- Os LOGs posteriores ao FULL de quarta-feira continuam válidos para restore  

### Análise final  

- Quando um novo backup FULL é executado ele redefine a base dos backups DIFFERENTIAL  
- Isso significa que diferenciais anteriores deixam de ser compatíveis com o FULL base anterior para aquela estratégia de recuperação  
- A cadeia de LOG não é quebrada por esse novo FULL mas a lógica do restore muda completamente  
- Esse é um cenário clássico para testar o entendimento sobre differential base  
- O ponto principal é perceber que FULL adicional impacta DIFFERENTIAL de forma diferente do impacto que ele tem sobre LOG chain  
- A armadilha comum é tentar usar o DIFFERENTIAL de terça-feira com o FULL de domingo ignorando o FULL manual de quarta-feira  

---

## Cenário 9 - Corrupção em FILEGROUP com necessidade de Piecemeal Restore  

### Estratégia  

- RECOVERY MODEL FULL  
- Banco dividido em múltiplos FILEGROUPS  
- FILEGROUP1 PRIMARY  
- FILEGROUP2 FG_SALES  
- FILEGROUP3 FG_HISTORY  
- BACKUP FULL semanal aos domingos às 10:00  
- BACKUP FILEGROUP diário às 18:00  
- BACKUP LOG diário a cada 1 hora das 08:00 às 17:00  
- Ambiente crítico com necessidade de reduzir downtime ao máximo  

### Linha do tempo  

- Domingo 10:00 backup FULL  
- Segunda-feira 18:00 backup do FILEGROUP2 FG_SALES  
- Terça-feira 18:00 backup do FILEGROUP3 FG_HISTORY  
- Quarta-feira backup LOG às 08:00  
- Quarta-feira backup LOG às 09:00  
- Quarta-feira backup LOG às 10:00
- Quarta-feira backup LOG às 11:00  
- Quarta-feira 11:30 corrupção detectada no FILEGROUP3 FG_HISTORY  
- Quarta-feira 11:30 o banco começa a falhar em consultas que dependem desse FILEGROUP  

### Como recuperar o banco?  

1 - Realizar BACKUP do TAIL LOG WITH NO_TRUNCATE  
2 - Restore do backup do FILEGROUP3 FG_HISTORY de terça-feira 18:00 WITH NORECOVERY  
3 - Restore do backup LOG de quarta-feira 08:00 WITH NORECOVERY  
4 - Restore do backup LOG de quarta-feira 09:00 WITH NORECOVERY  
5 - Restore do backup LOG de quarta-feira 10:00 WITH NORECOVERY  
6 - Restore do backup LOG de quarta-feira 11:00 WITH NORECOVERY  
7 - Restore do TAIL LOG WITH RECOVERY  

### Resultado  

- O restore é executado apenas sobre o FILEGROUP afetado  
- FILEGROUP1 PRIMARY e FILEGROUP2 FG_SALES podem permanecer disponíveis conforme a estratégia adotada e o tipo de acesso realizado  
- Objetos dependentes do FILEGROUP3 FG_HISTORY permanecem indisponíveis até a conclusão da recuperação  
- O downtime total tende a ser menor do que em um restore completo do banco  

### Análise final  

- Esse cenário demonstra recuperação avançada por FILEGROUP e o conceito de Piecemeal Restore  
- Nem sempre é necessário restaurar o banco inteiro quando a falha está isolada em uma parte específica da estrutura física  
- O FILEGROUP restaurado precisa ser levado ao mesmo ponto transacional do restante do banco por meio da aplicação correta da cadeia de LOG  
- Essa estratégia é especialmente valiosa em bancos muito grandes onde um restore completo demoraria demais  
- O ponto principal é entender que a arquitetura física do banco pode ser usada a favor da recuperação quando bem planejada  
- Também reforça a importância de separar dados por FILEGROUP de forma estratégica quando o banco exige alta disponibilidade operacional

---

## Cenário 10 - Interrupção recorrente de backup FULL e impacto na recuperação  

### Estratégia  

- RECOVERY MODEL FULL  
- BACKUP FULL semanal aos sábados iniciando às 04:00  
- BACKUP DIFFERENTIAL diário às 22:00  
- BACKUP LOG diário das 08:00 às 21:00 a cada 1 hora  
- Horário comercial: 08:00 às 20:00  
- Operação logística: 20:00 às 02:00  

### Linha do tempo  

- Sábado 04:00 início do BACKUP FULL  
- Domingo 08:00 início do horário comercial  
- Domingo 15:00 BACKUP FULL ainda em execução com 73%  
- Domingo 15:10 usuários relatam lentidão e travamentos  
- Domingo 15:20 BACKUP FULL é interrompido manualmente via KILL  
- Situação se repete pelos últimos 3 finais de semana consecutivos  

Durante a semana:  

- BACKUP LOG ocorre diariamente às 08:00 09:00 10:00 11:00 12:00 13:00 14:00 15:00 16:00 17:00 18:00 19:00 20:00 e 21:00  
- BACKUP DIFFERENTIAL ocorre diariamente às 22:00  

Evento de falha:  

- Sexta-feira 15:00 falha generalizada de rede  
- Servidor de produção é derrubado  
- Corrupção no arquivo MDF primário  

---

### Como recuperar o banco?  

1 - NÃO existe backup FULL recente válido disponível para restore  
2 - Os backups DIFFERENTIAL não podem ser utilizados pois dependem de um FULL base válido  
3 - A cadeia de LOG não pode ser aplicada sem um FULL base consistente  

#### Caminho possível de recuperação

1 - Tentar realizar BACKUP do TAIL LOG WITH NO_TRUNCATE  
2 - Caso o TAIL LOG seja realizado com sucesso, ele deverá ser utilizado ao final da sequência de restore  
3 - Identificar o último BACKUP FULL válido concluído com sucesso (anterior aos últimos 3 finais de semana)  
4 - Restore desse BACKUP FULL WITH NORECOVERY  
5 - Restore sequencial de TODOS os BACKUP LOG a partir desse FULL até o último LOG disponível antes da falha  
   - Restore LOG 08:00 WITH NORECOVERY  
   - Restore LOG 09:00 WITH NORECOVERY  
   - Restore LOG 10:00 WITH NORECOVERY  
   - ... seguir sequência cronológica até o último LOG disponível
     
6 - Restore do TAIL LOG WITH RECOVERY  

#### Caso o TAIL LOG não possa ser realizado

1 - Restore do BACKUP FULL válido WITH NORECOVERY  
2 - Restore sequencial dos BACKUP LOG até o último disponível WITH NORECOVERY  
3 - Finalizar com RESTORE WITH RECOVERY  

---

### Resultado  

- O banco será recuperado a partir do último FULL válido disponível  
- Todos os dados entre esse FULL e o último LOG aplicado poderão ser recuperados  
- Existe risco de perda de dados dependendo da disponibilidade do TAIL LOG  
- O tempo de restore será elevado devido à grande quantidade de LOGs acumulados  
- O RTO será significativamente alto  

---

### Análise final  

- O principal erro desse cenário é a interrupção recorrente do BACKUP FULL sem garantir um novo FULL válido  
- Sem um FULL base consistente não é possível utilizar DIFFERENTIAL nem garantir uma cadeia de recuperação eficiente  
- Os backups DIFFERENTIAL tornam-se inúteis se não houver um FULL base válido recente  
- A cadeia de LOG continua sendo gerada, porém depende de um FULL válido para ser utilizada no restore  
- Esse cenário gera um acúmulo muito grande de LOGs, aumentando drasticamente o tempo de recuperação  
- Em ambientes reais isso impacta diretamente o RTO e pode tornar o recovery inviável dentro da janela de negócio  
- A lentidão causada pelo FULL indica problema de estratégia de backup, podendo envolver falta de tuning, limitação de I/O ou ausência de uma janela adequada  
- A ação correta não é interromper o FULL, mas sim ajustar a estratégia, como uso de compressão, backup em múltiplos arquivos, otimização de I/O ou mudança de janela  
- Esse cenário é crítico porque demonstra como decisões operacionais incorretas ao longo do tempo comprometem a capacidade de recuperação em um incidente real  

---

## Cenário 11 - Recuperação até ponto específico (STOPAT)

### Estratégia  

- RECOVERY MODEL FULL  
- BACKUP FULL diário às 18:00  
- BACKUP LOG em intervalos regulares  

### Linha do tempo  

- Segunda-feira 18:00 backup FULL  
- Terça-feira backups LOG às 08:00 09:00 10:00 e 11:00  
- Terça-feira 11:15 execução incorreta de DELETE  
- Terça-feira 11:30 identificação do erro  

### Como recuperar o banco?  

1 - Realizar BACKUP do TAIL LOG WITH NO_TRUNCATE  
2 - Restore do backup FULL de segunda-feira 18:00 WITH NORECOVERY  
3 - Restore do backup LOG de terça-feira 08:00 WITH NORECOVERY  
4 - Restore do backup LOG de terça-feira 09:00 WITH NORECOVERY  
5 - Restore do backup LOG de terça-feira 10:00 WITH NORECOVERY  
6 - Restore do backup LOG de terça-feira 11:00 WITH NORECOVERY  
7 - Restore do TAIL LOG WITH STOPAT 'Terça-feira 11:14:59' WITH RECOVERY  

### Resultado  

- O banco é restaurado para o momento imediatamente anterior ao erro  
- A operação incorreta de DELETE não é aplicada  

### Análise final  

- STOPAT permite controle preciso baseado em data e hora  
- Depende da identificação correta do momento do erro  
- O TAIL LOG garante a recuperação das últimas transações antes da falha  
- Esse cenário é comum em erros operacionais e falhas de aplicação  

---

## Cenário 12 - Recuperação com transação marcada (STOPATMARK / STOPBEFOREMARK)

### Estratégia  

- RECOVERY MODEL FULL  
- Uso de transações marcadas para controle de deploy  
- BACKUP FULL e LOG conforme estratégia padrão  

### Linha do tempo  

- Segunda-feira 18:00 backup FULL  
- Terça-feira 10:00 início de deploy com transação marcada (Deploy_V1)  
- Terça-feira 10:05 execução de alterações no banco  
- Terça-feira 10:10 finalização do deploy  
- Terça-feira 10:30 identificação de problema no deploy  

### Como recuperar o banco (STOPATMARK)?  

1 - Realizar BACKUP do TAIL LOG WITH NO_TRUNCATE  
2 - Restore do backup FULL WITH NORECOVERY  
3 - Restore dos LOGs necessários WITH NORECOVERY  
4 - Restore do LOG WITH STOPATMARK 'Deploy_V1' WITH RECOVERY  

### Resultado  

- O banco é restaurado até o ponto da execução da transação marcada  
- As alterações do deploy são incluídas  

---

### Como recuperar o banco (STOPBEFOREMARK)?  

1 - Realizar BACKUP do TAIL LOG WITH NO_TRUNCATE  
2 - Restore do backup FULL WITH NORECOVERY  
3 - Restore dos LOGs necessários WITH NORECOVERY  
4 - Restore do LOG WITH STOPBEFOREMARK 'Deploy_V1' WITH RECOVERY  

### Resultado  

- O banco é restaurado antes da execução da transação marcada  
- As alterações do deploy não são aplicadas  

---

### Análise final  

- STOPATMARK e STOPBEFOREMARK permitem controle preciso baseado em transações  
- São mais confiáveis que STOPAT em cenários de deploy controlado  
- Não dependem de horário exato, mas sim de um ponto lógico definido  
- O uso de transações marcadas é uma prática avançada e recomendada em processos críticos  

---

## Observações gerais  

- Sempre manter a cadeia de backup íntegra  
- Sempre validar a ordem cronológica dos backups utilizados no restore  
- FULL não quebra LOG chain  
- FULL redefine a base dos backups DIFFERENTIAL  
- SIMPLE não permite LOG backup  
- Mudança para MODEL FULL exige novo bakup FULL para iniciar corretamente a cadeia de LOG  
- Sempre que possível realizar BACKUP do TAIL LOG WITH NO_TRUNCATE antes do restore  
- Backup íntegro sozinho não garante recovery se a cadeia estiver incompleta  
- Testes periódicos de restore são indispensáveis para validar a estratégia  
- RPO e RTO devem orientar a escolha do modelo de backup e recuperação

---

## Referências

- [Backups do log de cauda (tail-log)](https://learn.microsoft.com/pt-br/sql/relational-databases/backup-restore/tail-log-backups-sql-server?view=sql-server-ver16)
- [RESTORE Statements (Transact-SQL)](https://learn.microsoft.com/pt-br/sql/t-sql/statements/restore-statements-transact-sql?view=sql-server-ver16)
- [Piecemeal Restores (SQL Server)](https://learn.microsoft.com/pt-br/sql/relational-databases/backup-restore/piecemeal-restores-sql-server?view=sql-server-ver16)
- [Exemplo: Piecemeal Restore de banco de dados (modelo de recuperação completo)](https://learn.microsoft.com/pt-br/sql/relational-databases/backup-restore/example-piecemeal-restore-of-database-full-recovery-model?view=sql-server-ver16)
