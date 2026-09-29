---
title: "Apostila, Introducao a engenharia de dados e diagnostico"
date: 2026-07-31
type: apostila
status: draft
project: zambotto-mentoria
tags: [engenharia_de_dados, diagnostico]
---

# Apostila, Introducao a engenharia de dados e diagnostico

> Trilha de Engenharia de Dados. Conduzida por Iuri Zambotto e Paulo Shindi.

## Sumário

- [0. Como usar esta apostila](#0-como-usar-esta-apostila)
- [1. Objetivo pedagógico](#1-objetivo-pedagógico)
- [2. Contexto de negócio](#2-contexto-de-negócio)
- [3. O projeto que atravessa a trilha](#3-o-projeto-que-atravessa-a-trilha)
- [4. Ciclo de vida dos dados](#4-ciclo-de-vida-dos-dados)
- [5. O que é um data product](#5-o-que-é-um-data-product)
- [6. Métricas de negócio contra métricas técnicas](#6-métricas-de-negócio-contra-métricas-técnicas)
- [7. Hipóteses mensuráveis e critérios de sucesso](#7-hipóteses-mensuráveis-e-critérios-de-sucesso)
- [8. Exemplos do domínio](#8-exemplos-do-domínio)
- [9. Exercícios e entregáveis](#9-exercícios-e-entregáveis)
- [10. Mini-desafio com solução](#10-mini-desafio-com-solução)
- [11. Rubrica de validação da aprendizagem](#11-rubrica-de-validação-da-aprendizagem)
- [12. Erros comuns e como corrigir](#12-erros-comuns-e-como-corrigir)
- [13. Plano de continuidade](#13-plano-de-continuidade)
- [14. Glossário](#14-glossário)
- [Referências](#referências)
- [Fontes verificadas (2026-07-31)](#fontes-verificadas-2026-07-31)

## 0. Como usar esta apostila

**Leitura linear.** Este é o primeiro módulo da trilha, e ele existe para instalar
o vocabulário que todos os outros usam sem reexplicar. As seções 3 a 7 constroem
esse vocabulário na ordem em que ele é usado depois.

**Revisão pontual.** Se você voltou aqui atrás de um assunto: ciclo de vida na 4,
a diferença entre dado e data product na 5, métricas na 6, metas mensuráveis na 7.

**Como praticar.** Os exercícios da seção 9 são de escrita, não de código. O
entregável de cada um é uma tabela, e as três tabelas juntas formam o diagnóstico
inicial que você vai revisitar no fim da trilha para ver o quanto sua leitura
mudou.

**Pré-requisitos.** Nenhum. Este módulo não pressupõe SQL, Python nem nuvem.

**Este módulo não tem laboratório em container.** Ele declara `lab: false` no
`trilha.yml`, e a razão é o assunto: aqui não se executa pipeline, se decide o que
o pipeline deveria fazer. A primeira ferramenta aparece no módulo de SQL.

**Sobre os números desta apostila.** Todas as tabelas das seções 7 e 8 são
ilustrativas e autocontidas: os valores foram escolhidos para o exemplo, não
extraídos de sistema nenhum. O que foi verificado é a aritmética, porcentagem por
porcentagem, e a seção de fontes verificadas registra o comando. Um material que
ensina a desconfiar de número tem a obrigação de acertar os próprios.

## 1. Objetivo pedagógico

Ao terminar este módulo, você consegue:

1. **Descrever** as quatro etapas do ciclo de vida de um dado, e dizer em qual
   delas um problema concreto se origina.
2. **Distinguir** um dado bruto de um data product, usando os quatro atributos da
   seção 5.
3. **Classificar** uma métrica como de negócio ou técnica, e explicar como a
   segunda sustenta a primeira.
4. **Reescrever** um objetivo vago como hipótese mensurável, indicando métrica,
   valor-alvo, prazo e a tabela que serve de medida.
5. **Mapear** uma pergunta de negócio até as fontes de dados necessárias para
   respondê-la.

O verbo de cada item é o que será cobrado. "Descrever" e "explicar" são orais, na
call. "Reescrever" e "mapear" são entregáveis escritos, e estão na seção 9.

## 2. Contexto de negócio

A trilha inteira gira em torno de uma startup fictícia de marketing e e-commerce.
Ela gerencia campanhas de aquisição paga em vários canais e tem uma base de
usuários em crescimento. O negócio precisa responder perguntas como:

- Qual campanha está gerando mais receita este mês?
- Quais canais trazem o melhor retorno sobre o investimento?
- Quais usuários estão em risco de deixar de usar o produto?
- Onde no funil de conversão estamos perdendo mais usuários?

Nenhuma dessas perguntas se responde com uma tabela só. Elas exigem integrar
fontes de naturezas diferentes: um banco relacional com cadastros, uma API externa
de mídia que reporta custo de anúncio, e um sistema de eventos que captura o
comportamento no produto. Cada natureza pede uma estratégia de ingestão
diferente, e é por isso que a trilha tem módulos separados para elas.

**O time é pequeno**, e isso não é um detalhe de enredo. É o critério que elimina
metade das arquiteturas possíveis: cada ferramenta escolhida precisa ser operada
por poucos engenheiros. Solução que exige equipe dedicada para manter não cabe
aqui, mesmo quando é tecnicamente superior. O módulo de fontes, arquitetura e
contratos retoma esse critério na discussão de construir contra comprar.

**As cinco entidades do domínio.** O modelo usado em todos os módulos:

| Entidade | O que representa |
|---|---|
| `users` | Cadastro dos usuários da plataforma |
| `campaigns` | Campanhas de marketing criadas e gerenciadas |
| `events` | Eventos de comportamento dos usuários no funil |
| `costs` | Custos diários por campanha e canal de mídia |
| `crm` | Relacionamento e score de risco de churn por usuário |

Duas chaves conectam tudo: `user_id` une `users`, `events` e `crm`; `campaign_id`
une `campaigns`, `costs` e `events`. A entidade `events` é o centro do modelo,
porque é ela que registra o encontro entre um usuário e uma campanha ao longo do
funil. Guarde esse desenho: ele volta no módulo de SQL como as tabelas do JOIN, e
no módulo de dbt como os modelos.

## 3. O projeto que atravessa a trilha

Um material de engenharia de dados pode ser organizado por ferramenta ou por
problema. Esta trilha escolhe por problema, e usa um projeto único como fio
condutor, por um motivo prático: ferramenta aprendida fora de contexto vira
sintaxe decorada, e sintaxe decorada não sobrevive à primeira decisão de
arquitetura que você precisa justificar para outra pessoa.

O projeto acumula, módulo a módulo, até virar um pipeline completo:

```
[PostgreSQL / API de midia / eventos]
         |
   [ingestao, batch e CDC]
         |
    [bronze, dado bruto]
         |
   [silver, dado curado]
         |
    [gold, metricas]
         |
     [BI e analise]
```

Os nomes bronze, silver e gold vêm da arquitetura medalhão, e a documentação da
Databricks define as três camadas assim: bronze guarda o dado bruto e não
validado, no formato original da fonte, crescendo de forma incremental; silver
guarda a versão validada, limpa e enriquecida, onde acontecem deduplicação e
normalização; gold guarda o dado agregado e alinhado à regra de negócio, otimizado
para consulta e dashboard.

Duas observações honestas sobre esse desenho. A primeira: ele é uma convenção de
nomes, não uma exigência técnica. Nada impede quatro camadas, ou duas. A segunda:
a fronteira entre silver e gold é a que gera mais discussão em time real, porque
"enriquecido" e "agregado" não têm definição objetiva. O que resolve na prática é
escrever quem consome cada camada, e isso é assunto de contrato de dados.

Este módulo é o único da trilha que não constrói nenhum pedaço desse pipeline. Ele
decide o que o pipeline precisa responder, o que é a etapa que costuma ser
pulada.

## 4. Ciclo de vida dos dados

Antes de projetar qualquer pipeline, é preciso saber como o dado se move. O ciclo
tem quatro etapas, e a utilidade de nomeá-las é diagnóstica: quando algo dá
errado, o primeiro trabalho é descobrir em qual etapa.

### Ingestão

O dado nasce em sistemas de origem que não são seus: o banco da aplicação, a API
de um fornecedor, um sistema de eventos. Ingestão é extrair de lá e trazer para o
ambiente de dados.

A natureza da fonte determina a estratégia, e a trilha cobre as três:

| Natureza da fonte | Estratégia | Onde na trilha |
|---|---|---|
| Banco transacional | CDC, captura de cada alteração | módulo de CDC |
| API externa | lote periódico | módulo de fontes e arquitetura |
| Sistema de eventos | streaming contínuo | módulo de streaming |

### Transformação

Dado bruto raramente responde pergunta. A transformação cobre limpeza, com
remoção de duplicata e tratamento de nulo; enriquecimento, com junção contra
tabela de referência e campo calculado; e agregação, com a métrica resumida por
período.

Isso precisa rodar de forma confiável, repetível e monitorada, e é o que justifica
um orquestrador. O módulo de orquestração com Airflow é onde esse fluxo ganha
agendamento, dependência e nova tentativa em caso de falha.

### Modelagem

Modelagem é estruturar o dado transformado para que ele responda a pergunta com
eficiência. As decisões são de grão, de chave e relação, de formato de
armazenamento e de estratégia de particionamento. As duas últimas têm módulo
próprio, porque são as que mais mexem na conta no fim do mês.

### Consumo

O dado modelado é lido por analista, dashboard, modelo ou API de produto. O
consumo é o teste final: se o dado chega correto, fresco e barato ao consumidor, o
pipeline cumpriu o objetivo. Se não chega, nenhuma elegância nas três etapas
anteriores compensa.

**Onde os problemas nascem.** Vale gravar o padrão, porque ele se repete:

| Sintoma no consumo | Etapa que costuma ser a causa |
|---|---|
| Número mudou sem ninguém mexer | ingestão, mudança de schema na origem |
| Métrica duplicada | transformação, junção que multiplicou linha |
| Dashboard lento e caro | modelagem, partição ou formato errados |
| Relatório chega tarde demais | orquestração, agendamento errado |

## 5. O que é um data product

Um data product é um ativo de dados construído com a intenção de ser consumido
por alguém, interno ou externo, para tomar decisão. A palavra que carrega o peso é
intenção: não é o dataset que existe, é o dataset com destinatário e garantia.

### Os quatro atributos

**Utilidade.** Resolve um problema real. Uma tabela de eventos que ninguém abre
não é data product, é dado sem destinatário. No domínio do projeto, um exemplo
com utilidade clara é uma tabela gold com ROI diário por campanha, lida pelo time
de marketing toda manhã para ajustar o orçamento do dia.

**Confiabilidade.** O consumidor pode confiar sem conferir. Isso significa ausência
de duplicata, valor dentro do intervalo esperado, chave primária sem nulo e
cobertura temporal completa. Dado que "parece certo" e não pode ser verificado não
é confiável, é suposição com aparência de fato.

**Tempo de entrega.** Está disponível quando a decisão acontece. Um relatório de
ROI que fica pronto às 15h, num dia em que o orçamento é decidido às 9h, tem
conteúdo correto e valor zero. O prazo é parte da especificação, não um detalhe
operacional.

**Qualidade percebida.** O consumidor entende o que recebeu. Schema documentado,
definição de campo, histórico de mudança e aviso antes de quebrar. Esse atributo é
o único dos quatro que não se resolve com técnica: ele se constrói com
transparência ao longo do tempo, e se perde de uma vez.

### Dado bruto contra data product

| Dado bruto | Data product |
|---|---|
| Sem destinatário definido | Tem consumidor e caso de uso |
| Sem prazo acordado | Tem prazo de entrega |
| Sem documentação | Schema documentado |
| Sem monitoramento | Alerta de qualidade configurado |
| Sem contrato | Contrato de dados definido |

A coluna da direita é mais caro de manter, e é o ponto: nem todo dado precisa ser
data product. Promover tudo a data product é uma forma comum de o time pequeno
travar. O módulo de fontes, arquitetura e contratos trata o contrato de dados, que
é o mecanismo formal de dizer o que o data product entrega e com qual garantia.

## 6. Métricas de negócio contra métricas técnicas

Uma confusão comum no começo de carreira é tratar métrica técnica como objetivo.
Ela é instrumento. O objetivo está sempre do lado do negócio.

### Métricas de negócio

São as que sustentam decisão. No domínio do projeto:

| Métrica | O que é | Como se calcula |
|---|---|---|
| CAC | Custo de aquisição de cliente | gasto em mídia dividido por novos clientes no período |
| ROI | Retorno sobre investimento | `(receita - custo) / custo` |
| Taxa de conversão | Avanço de etapa no funil | usuários que completaram a etapa sobre os que entraram |
| Churn | Saída de usuários | usuários que pararam de usar sobre o total, no período |
| LTV | Valor do cliente ao longo da relação | receita média por cliente projetada pelo tempo de vida |

Duas ressalvas que material introdutório costuma omitir. CAC e LTV têm mais de uma
definição em uso: entra só mídia paga ou o custo do time de vendas também, o LTV é
receita ou margem. E churn depende de uma definição arbitrária de "parou de usar",
que precisa estar escrita. Métrica sem definição escrita produz duas pessoas
discutindo números diferentes com o mesmo nome.

### Métricas técnicas

Indicam a saúde do que produz as métricas de negócio:

| Métrica | O que indica |
|---|---|
| Latência de ingestão | tempo entre o fato na origem e a disponibilidade |
| Frescor | quão recente é o dado mais novo da tabela |
| Taxa de erro | registros rejeitados por schema ou regra de qualidade |
| Cobertura | se tudo que era esperado chegou, e onde há lacuna |
| Tamanho médio de arquivo | indício de arquivo pequeno demais e de desequilíbrio |

### A relação entre as duas

A métrica técnica só importa pelo efeito na de negócio, e o efeito é rastreável:

- Latência de ingestão sobe, e o ROI do relatório da manhã fica desatualizado.
- Taxa de erro sobe, e a taxa de conversão vem subestimada por evento perdido.
- Cobertura cai num dia, e a série histórica ganha um vale que parece queda real
  de demanda.

O terceiro caso é o mais perigoso dos três, porque produz uma conclusão de negócio
plausível a partir de uma falha técnica silenciosa. Engenharia de dados existe
para manter a métrica técnica em nível que torne a de negócio confiável.

## 7. Hipóteses mensuráveis e critérios de sucesso

Antes de construir, é preciso definir o que significa ter dado certo. Hipótese
mensurável é o que traduz objetivo em afirmação verificável.

### O formato SMART, e uma correção comum

O acrônimo foi publicado por George T. Doran em 1981, na Management Review, e no
original as letras são:

| Letra | Original de Doran | O que significa |
|---|---|---|
| S | Specific | mira uma área determinada de melhoria |
| M | Measurable | quantifica, ou ao menos sugere, um indicador de progresso |
| A | Assignable | define de quem é a responsabilidade |
| R | Realistic | resultado alcançável com os recursos disponíveis |
| T | Time-related | tem prazo declarado |

A forma que circula hoje troca três letras: **Achievable** no lugar de Assignable,
**Relevant** no lugar de Realistic e **Time-bound** no lugar de Time-related. Essa
variante é posterior a Doran, e é a mais usada na prática. Vale saber das duas por
um motivo concreto: o **Assignable** original cobra dono, e a variante moderna
perdeu isso. Em projeto de dados, meta sem dono é a que ninguém persegue.

Esta apostila usa a variante moderna nos exemplos, e recomenda acrescentar o dono.

### Exemplos aplicados ao domínio

| Objetivo vago | Hipótese mensurável |
|---|---|
| "Melhorar a conversão" | Aumentar a conversão do funil de 1,5% para 2,0% até o fim do trimestre, medida pela razão entre compras e visitas na tabela `events`, agrupada por semana. Dono: growth |
| "Reduzir o CAC" | Reduzir o CAC do canal de maior gasto de 45 para 38 na moeda local em 60 dias, calculado como a soma de `costs` dividida pelos novos registros em `users`. Dono: mídia |
| "Detectar churn" | Identificar usuários com score acima de 0,7 em `crm` com 7 dias de antecedência e acerto de ao menos 70%, medido contra as saídas reais do período seguinte. Dono: CRM |

Repare no que cada reescrita acrescentou: a métrica, o ponto de partida, o alvo, o
prazo, a tabela que serve de medida e o dono. Falta qualquer um desses e a
hipótese volta a ser discutível.

### Critérios de sucesso do pipeline da trilha

Para o pipeline construído ao longo dos módulos, sucesso técnico é:

1. As cinco entidades disponíveis em bronze e silver, com schema documentado.
2. A camada gold com ao menos uma tabela por grupo de consumidor.
3. Prazos de entrega acordados e cumpridos, cada um escrito antes de ser medido.
4. O pipeline idempotente: reexecutar a mesma carga não duplica dado nem gera
   inconsistência.

O quarto item é o que mais se descobre tarde. Ele aparece cedo, no módulo de
orquestração, porque nova tentativa automática é inútil quando reexecutar
corrompe.

## 8. Exemplos do domínio

Os números desta seção são ilustrativos, escolhidos para o exemplo. A aritmética
foi conferida, e é ela que sustenta a leitura.

### Desempenho por campanha

| campanha | visitas | compras | conversão | custo | receita | ROI |
|---|---|---|---|---|---|---|
| camp_001 | 450 | 6 | 1,33% | 2.000 | 480 | -0,76 |
| camp_002 | 456 | 15 | 3,29% | 2.000 | 1.125 | -0,44 |

A `camp_002` converte mais que o dobro da `camp_001`, e as duas têm ROI negativo:
o custo de aquisição ainda supera a receita. A leitura útil não é "camp_002 é
melhor", é que **conversão e retorno são perguntas diferentes**. Uma campanha pode
converter bem e continuar dando prejuízo, e é exatamente o que a tabela mostra.

### Funil de conversão

Cada etapa é um registro em `events`:

```
visita -> signup -> checkout -> purchase
```

| Etapa | Usuários | Avanço da etapa anterior |
|---|---|---|
| visita | 1.000 | 100% |
| signup | 250 | 25% |
| checkout | 80 | 32% |
| purchase | 25 | 31% |

A queda mais forte está entre visita e signup: 75% saem ali. É o ponto de
investigação prioritário, e a coluna de avanço é sempre em relação à etapa
anterior, não ao topo. Confundir as duas leituras é o erro mais comum em análise de
funil, e ele muda a conclusão: 25 compras sobre 1.000 visitas é 2,5% de conversão
de ponta a ponta, não os 31% da última linha.

### Risco de churn por segmento

| Segmento | Usuários | Em risco | Taxa de risco |
|---|---|---|---|
| premium | 500 | 45 | 9% |
| free | 2.000 | 680 | 34% |

O segmento `free` tem taxa de risco quase quatro vezes maior, 3,8 vezes para ser
exato. Isso pode justificar retenção direcionada, e pode também ser o
comportamento esperado de um plano gratuito. O dado localiza onde olhar, e não
decide sozinho: essa fronteira entre o que o dado mostra e o que ele não diz é o
assunto que fecha este módulo.

## 9. Exercícios e entregáveis

### Exercício 1, metas mensuráveis

Objetivo: traduzir objetivo genérico em hipótese verificável.

Reescreva os três objetivos abaixo no formato da seção 7, indicando métrica,
ponto de partida, valor-alvo, prazo, a tabela do domínio que serve de medida e o
dono:

1. "Quero aumentar a receita das campanhas."
2. "Quero diminuir o custo por clique."
3. "Quero melhorar a retenção dos usuários premium."

Entregável: três hipóteses, cada uma com os seis elementos.

### Exercício 2, mapa de perguntas de negócio

Objetivo: praticar o raciocínio que liga pergunta de stakeholder a fonte de dado.

Monte uma tabela com 10 perguntas de negócio do domínio. Para cada uma, indique
quem faria a pergunta, quais das cinco entidades são necessárias para responder, e
com que frequência o dado precisa ser atualizado.

Entregável: tabela com 10 perguntas, priorizadas, com consumidor, entidades e
frequência.

### Exercício 3, inventário inicial de fontes

Objetivo: mapear as fontes necessárias antes de escolher ferramenta.

Monte uma tabela com as cinco entidades. Para cada uma, preencha o tipo de fonte,
a frequência de atualização esperada, o time dono do dado na origem e a
criticidade para as perguntas do exercício 2.

Entregável: tabela de inventário com as quatro colunas preenchidas.

A coluna de dono na origem é a que costuma ficar em branco, e é a mais útil das
quatro. Ela é o nome da pessoa para quem você vai escrever quando o schema mudar
sem aviso.

## 10. Mini-desafio com solução

**O enunciado.** O CEO abre o dashboard numa segunda-feira e diz: "a receita de
campanha caiu 40% no fim de semana, precisamos cortar o investimento em mídia
hoje".

Você tem acesso às cinco entidades. Antes de concordar ou discordar, escreva as
verificações que faria, em ordem, e diga qual decisão cada resultado sustenta.

**Dica 1.** Uma das etapas do ciclo de vida da seção 4 explica esse sintoma sem
que nada tenha acontecido no negócio.

**Dica 2.** Duas métricas da seção 6 respondem à pergunta "esse número é
confiável?" antes de qualquer análise de causa.

**Solução comentada.**

A ordem correta separa falha de dado de fato de negócio, e ela é sempre a mesma.

**Primeiro, o dado é confiável?** Antes de explicar a queda, confirme que ela
existe. Duas métricas técnicas respondem:

- **Frescor:** qual é o registro mais recente em `events` e em `costs`? Se a
  ingestão do fim de semana não rodou, ou rodou parcialmente, a queda é a ausência
  do dado, não a ausência de venda. Este é o desfecho mais provável de um sintoma
  que aparece exatamente na segunda-feira.
- **Cobertura:** a contagem de registros por dia tem o mesmo perfil dos fins de
  semana anteriores? Uma lacuna localizada aponta falha de carga; uma queda
  proporcional em todas as entidades aponta problema na origem, não no seu
  pipeline.

**Segundo, a comparação é justa?** Se o dado está íntegro, verifique contra o quê
os 40% foram calculados. Fim de semana comparado com dia útil cai em quase todo
e-commerce, e isso é sazonalidade, não tendência. A comparação que sustenta
decisão é com os fins de semana anteriores.

**Terceiro, a queda é de receita ou de conversão?** Aqui entra a leitura da seção
8. Se as visitas caíram e a conversão ficou estável, o problema é topo de funil, e
cortar mídia agrava. Se as visitas se mantiveram e a conversão caiu, o problema
está no produto ou no checkout, e cortar mídia não resolve nada.

**Quarto, a decisão proposta responde à causa?** Cortar mídia só faz sentido se a
causa for gasto sem retorno, o que exige ROI por campanha no período, com o custo
do mesmo período. Nas outras três hipóteses, cortar mídia é agir sobre o sintoma
errado.

**O que este desafio ensina.** A pergunta que abre a semana não é "por que caiu",
é "caiu?". Um engenheiro de dados que responde a segunda pergunta primeiro evita a
categoria mais cara de erro: a decisão de negócio tomada com confiança sobre um
pipeline que falhou em silêncio.

## 11. Rubrica de validação da aprendizagem

| Critério | Insuficiente | Suficiente | Excelente |
|---|---|---|---|
| Ciclo de vida | Lista as etapas | Localiza em qual etapa um problema nasce | Antecipa qual etapa vai falhar primeiro num desenho novo |
| Data product | Confunde com dataset | Aplica os quatro atributos | Argumenta quando **não** promover um dado a data product |
| Métricas | Trata técnica e negócio como o mesmo | Classifica corretamente as duas | Rastreia o efeito de uma métrica técnica numa decisão de negócio |
| Hipótese mensurável | Reescreve sem métrica ou sem prazo | Cobre os seis elementos | Percebe quando a métrica pedida não é mensurável com as fontes existentes |
| Leitura de funil | Lê a porcentagem sem saber a base | Distingue avanço de etapa de conversão ponta a ponta | Identifica a etapa de maior perda e o que ela implica |
| Diagnóstico | Explica a causa do número | Confirma o número antes de explicar | Ordena as verificações da mais barata para a mais cara |
| Comunicação | Descreve o dado | Traduz o dado em decisão possível | Diz com clareza o que o dado **não** permite concluir |

Checklist para a call:

- [ ] Nomeou as quatro etapas do ciclo de vida e deu um exemplo de falha em cada.
- [ ] Explicou a diferença entre dado bruto e data product sem citar a tabela.
- [ ] Reescreveu ao menos um objetivo vago no formato mensurável, com dono.
- [ ] Leu a tabela de campanhas e percebeu que conversão e ROI discordam.
- [ ] Resolveu o mini-desafio começando pela confiabilidade do dado.

## 12. Erros comuns e como corrigir

**Escolher a ferramenta antes da pergunta**

Sintoma: a arquitetura está desenhada, e ninguém sabe dizer qual decisão de
negócio ela sustenta.

Causa: começar pelo que é interessante de construir, e não pelo que é preciso
responder.

Correção: o exercício 2 antes do exercício 3. A pergunta define a fonte, a fonte
define a estratégia de ingestão, e só então a ferramenta aparece.

**Confundir avanço de etapa com conversão de ponta a ponta**

Sintoma: alguém apresenta 31% de conversão num funil que converte 2,5%.

Causa: ler a porcentagem da última etapa sem olhar a base sobre a qual ela foi
calculada.

Correção: declarar a base em toda porcentagem de funil. Escrever "31% do
checkout" em vez de "31% de conversão" resolve a ambiguidade no próprio texto.

**Tratar métrica técnica como objetivo**

Sintoma: relatório de saúde do pipeline sem nenhuma menção ao efeito no negócio.

Causa: as métricas técnicas são as que o time controla, e por isso viram o
assunto.

Correção: para cada métrica técnica monitorada, escrever a frase "se isto piorar,
a decisão X fica errada". Métrica que não completa a frase não precisa de alerta.

**Aceitar métrica sem definição escrita**

Sintoma: duas áreas apresentam CAC diferentes na mesma reunião, e as duas estão
certas.

Causa: CAC, LTV e churn têm mais de uma definição legítima, e nenhuma foi
acordada.

Correção: escrever a fórmula e o que entra nela, no mesmo lugar onde a métrica é
publicada. Discussão de definição resolvida uma vez, e não a cada reunião.

**Explicar a queda antes de confirmar a queda**

Sintoma: uma hora de análise de causa, e no fim a carga do fim de semana não tinha
rodado.

Causa: confiança implícita no pipeline.

Correção: frescor e cobertura primeiro, sempre. É a verificação mais barata da
lista, e é a que mais economiza tempo.

**Prometer prazo de entrega que ninguém mediu**

Sintoma: prazo acordado na reunião, descumprido na primeira semana.

Causa: prazo definido pela necessidade do consumidor, sem confrontar com a
latência real da fonte.

Correção: medir a latência atual antes de acordar prazo. Se a origem só
disponibiliza o custo de mídia no dia seguinte, nenhum orquestrador entrega o dado
de hoje hoje.

## 13. Plano de continuidade

**Antes da próxima call**

Faça os três exercícios da seção 9. O segundo é o que mais rende conversa, porque
ele expõe as perguntas que a empresa faz e não consegue responder.

**O que estudar em seguida, dentro da trilha**

O próximo degrau é SQL com foco em JOINs, e ele usa exatamente as entidades da
seção 2. Depois dele, o bloco de armazenamento mostra onde essas tabelas moram e
por que o formato e a partição decidem o custo de cada pergunta.

A ordem dos blocos da trilha não é arbitrária: ela segue o ciclo de vida da seção
4. Você aprende a consultar, depois a guardar, depois a transformar, depois a
orquestrar.

**O que aprofundar por conta**

Pegue a empresa onde você trabalha, ou uma que você conheça bem, e faça o
exercício 2 com o domínio dela. O exercício vale mais com dado que você entende, e
o resultado é utilizável no trabalho.

**O que não perseguir agora**

Comparação entre ferramentas de mercado, e desenho de arquitetura de referência.
Os dois assuntos ficam melhores depois de você ter operado ao menos um pipeline
completo, e a trilha chega lá.

## 14. Glossário

| Termo | Significado |
|---|---|
| Arquitetura medalhão | Convenção de camadas por qualidade do dado, bronze, silver e gold |
| Batch | Processamento em lote periódico, em oposição a contínuo |
| CAC | Custo de aquisição de cliente |
| CDC | Change Data Capture, captura de cada alteração de um banco de origem |
| Churn | Saída de usuários num período, segundo uma definição acordada |
| Cobertura | Se todos os registros esperados chegaram, e onde há lacuna |
| Contrato de dados | Acordo escrito do que uma tabela entrega e com qual garantia |
| Data product | Ativo de dados com consumidor, garantia e prazo definidos |
| Frescor | Quão recente é o dado mais novo disponível numa tabela |
| Grão | O que cada linha de uma tabela representa |
| Idempotência | Propriedade de reexecutar sem duplicar nem corromper |
| Ingestão | Extrair dado do sistema de origem e trazer para o ambiente de dados |
| Latência de ingestão | Tempo entre o fato na origem e a disponibilidade para consulta |
| LTV | Valor do cliente ao longo da relação com o produto |
| Orquestração | Agendamento, dependência e nova tentativa de tarefas de dados |
| Partição | Divisão física do dado por valor de coluna, para ler menos |
| ROI | Retorno sobre investimento, `(receita - custo) / custo` |
| Sazonalidade | Variação esperada e recorrente, por dia da semana ou período |
| Stakeholder | Quem consome o dado e decide com ele |
| Streaming | Processamento contínuo, evento a evento |

## Referências

Arquitetura medalhão, documentação da Databricks, consultada em 2026-07-31:

- https://docs.databricks.com/aws/en/lakehouse/medallion

Origem do acrônimo SMART, consultada em 2026-07-31:

- https://en.wikipedia.org/wiki/SMART_criteria
- Doran, G. T. "There's a S.M.A.R.T. way to write management's goals and
  objectives". Management Review, novembro de 1981.

## Fontes verificadas (2026-07-31)

- O acrônimo SMART foi publicado por George T. Doran na Management Review em
  novembro de 1981, e no original as letras são Specific, Measurable,
  **Assignable**, **Realistic** e **Time-related**. As formas Achievable, Relevant
  e Time-bound são variantes posteriores, não a formulação de Doran. A versão
  anterior desta apostila apresentava a variante moderna como se fosse o
  framework, sem ressalva. https://en.wikipedia.org/wiki/SMART_criteria
- As definições de bronze, silver e gold da seção 3 são as da documentação da
  Databricks: bronze com dado bruto e não validado no formato original, crescendo
  incrementalmente; silver com o dado validado, limpo e enriquecido, incluindo
  deduplicação e normalização; gold com dado agregado e alinhado à regra de
  negócio, otimizado para consulta. A mesma documentação **não** atribui a autoria
  do termo a ninguém, e por isso esta apostila também não atribui.
  https://docs.databricks.com/aws/en/lakehouse/medallion
- Toda a aritmética das tabelas das seções 7 e 8 foi conferida por execução, e não
  por leitura: as duas taxas de conversão e os dois ROI da tabela de campanhas, os
  três avanços de etapa do funil, as duas taxas de risco de churn e a conversão de
  ponta a ponta de 2,5%. As nove porcentagens publicadas conferem com o cálculo.
- A relação de 34% contra 9% no churn por segmento é de 3,8 vezes, não de 4. A
  versão anterior escrevia "quatro vezes maior", e o texto agora diz "quase quatro
  vezes" com o número exato ao lado.
- A referência ao módulo `cdc_generator` como origem dos dados desta apostila foi
  removida. O módulo **não existe** no repositório: a auditoria por `git ls-tree`
  não encontra nenhum arquivo ou diretório com esse nome, embora sete arquivos de
  material o citem. Os números desta apostila são declarados como ilustrativos,
  que é o que eles são.
- As referências a "Capítulo 2", "Capítulo 3", "Capítulo 4" e "Capítulo 5" foram
  trocadas por nomes de módulo. Elas vinham da apostila consolidada anterior, e
  apontavam para uma numeração que não existe mais na trilha modular.
- Os números das tabelas das seções 7 e 8 são ilustrativos e não foram extraídos
  de sistema nenhum. Isso está declarado na seção 0 e repetido na abertura da
  seção 8, em vez de ficar implícito.
