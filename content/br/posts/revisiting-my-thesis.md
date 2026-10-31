---
title: "Revisitando meu TCC, quase dez anos depois"
date: 2026-10-31
tags: ["thesis", "nlp", "summarization", "reproducibility", "python"]
comments: true
---

Em 2016 eu terminei meu TCC em Ciência da Computação: um sumarizador automático
de notícias em português chamado **ConceptRank**. A ideia era pegar o LexRank,
um método clássico que monta um grafo de frases e escolhe as mais "centrais", e
trocar a similaridade entre frases por uma similaridade baseada em conceitos,
que deveria lidar melhor com sinônimos e variações de escrita. Avaliei no
[CSTNews](https://sites2.icmc.usp.br/taspardo/sucinto/cstnews.html), um corpus
de 50 grupos de notícias, com a métrica ROUGE. O resultado: o ConceptRank ganhava
do LexRank e do CSTSumm, um sistema da literatura.

O repositório ficou parado desde então, no estado em que todo TCC fica: notebooks,
Python 2, caminhos fixos no código e um README que dizia "[UNDER REFACTOR]". Esse
mês eu resolvi finalmente arrumar a casa. O código agora está em
[github.com/anaschwendler/tcc](https://github.com/anaschwendler/tcc), e o que eu
aprendi no caminho foi bem mais interessante do que eu esperava.

## Primeiro congelar, depois consertar

Antes de mexer em qualquer coisa, eu guardei os resultados de 2016 numa pasta
própria e escrevi o que se sabia sobre como eles foram gerados. A regra era: cada
correção só entra se eu conseguir medir quanto ela muda os números. Para isso, o
primeiro passo foi portar o código para Python 3 **sem mudar o comportamento**, e
escrever um teste que compara os sumários novos com os de 2016.

O teste não passou. O LexRank portado só reproduzia 24 dos 50 sumários.

## As bibliotecas mudam por baixo de você

Descobrir o porquê virou arqueologia. Os sumários eram de outubro de 2016, então
montei um ambiente descartável com as versões da época (`sumy` 0.4.1 e `nltk`
3.2.1) e rodei o mesmo código: 49 de 50 iguais byte a byte, e o 50º com o mesmo
texto e uma sobra de um arquivo que nunca foi fechado.

A diferença estava no tokenizador. O `sumy` de 2016 só aceitava palavras feitas
apenas de letras, então descartava tudo que tinha hífen: "segunda-feira",
"porta-voz", "vice-presidente". Em notícia em português, isso é muita coisa. As
versões novas mantêm essas palavras, e a seleção de frases muda.

Aprendizado: **sem as versões travadas, um resultado não é reprodutível**, mesmo
com o código idêntico. Hoje o projeto usa `uv` com um `uv.lock`, e o modo "como
na tese" reproduz o filtro antigo de propósito.

## Bugs que estavam escondidos pelo Python 2

O ConceptRank tinha outro problema: rodando duas vezes o mesmo código em Python 3,
39 dos 50 sumários saíam diferentes. A similaridade percorria um conjunto de
palavras guardando um máximo acumulado, então o resultado dependia da **ordem** de
iteração. No Python 2 essa ordem era fixa (mas arbitrária); no Python 3 ela muda a
cada execução. O bug sempre esteve lá, só não aparecia.

E olhando de perto, a fórmula também não era a do artigo original: em vez do peso
de cada palavra na frase, ela usava o **tamanho da palavra** (`len(term)`).

## A métrica estava medindo letras

Essa foi a descoberta que mudou tudo. Os números da tese eram altos demais: precisão
de ROUGE-1 entre 0,90 e 0,97. Fui ler o código da biblioteca que eu usava para
chamar o ROUGE, um fork do `pyrouge`, e a função `score_summary()` esperava uma
**lista de frases**. Eu passava uma **string**. Iterar sobre uma string em Python
percorre caracteres, então cada letra virou uma "frase", e o ROUGE comparou
**letras e pares de letras**, não palavras.

Para ter certeza, reimplementei esse comportamento e reproduzi os 150 valores
arquivados (3 sistemas × 50 grupos): 149 batem até a quinta casa decimal. O único
que não bate é justamente o arquivo que não foi fechado.

Medindo os mesmos sumários de 2016 com ROUGE por palavras, a vantagem do ConceptRank
sobre o LexRank continua na média, mas deixa de ser estatisticamente significativa,
e a vantagem sobre o CSTSumm some.

Aprendizado: **teste a sua avaliação**, não só o seu modelo. Uma checagem simples
(dois textos sem nenhuma palavra em comum deveriam ter ROUGE perto de zero) teria
pegado isso em 2016.

## O corpus também tinha surpresas

Ao reescrever a leitura do corpus, apareceram mais coisas:

- Os **títulos** das notícias entravam como se fossem frases, colados na primeira
  frase do texto.
- A **referência** usada no ROUGE tinha uma etiqueta no fim de cada frase
  (`<D1_C1_Folha; p1-s1>`), então "D1", "Folha" e "p1" contavam como palavras que
  o sumário deveria ter.
- Em 9 grupos essas etiquetas tinham o número do documento ou o jornal errado.
- Nomes de pasta gravados num codepage do DOS e lidos como cirílico (`Polбtica`).
- A lista de stopwords em português do `sumy` tem 36 das 195 palavras gravadas como
  escape literal (`n\xe3o` em vez de "não"), então "não", "já" e "são" nunca eram
  removidas.

## Uma correção por vez, e cuidado com o tamanho

Corrigi cada problema separadamente e medi cada um com teste de Wilcoxon pareado.
No meio do caminho, apareceu um efeito que contaminava tudo: o **tamanho do
sumário**. A tese escolhia um número fixo de frases, então qualquer mudança que
fizesse o sistema escolher frases mais curtas baixava o recall, sem que a escolha
fosse pior. Passei a cortar todos os sumários no tamanho da referência e refiz as
comparações.

Com isso, tirar os títulos, manter as palavras com hífen e controlar o tamanho
ajudaram de verdade. Stemmer em português, a segmentação oficial do corpus e a
fórmula "corrigida" do ConceptRank não fizeram diferença mensurável.

## E o ConceptRank, afinal?

Com todas as correções, medindo por palavras nos 50 grupos:

| Sistema | ROUGE-1 recall | ROUGE-2 recall |
|---|---|---|
| Lead (primeiras frases de cada notícia) | **0,587** | **0,404** |
| ConceptRank | 0,579 | 0,376 |
| LexRank | 0,569 | 0,364 |
| Frases aleatórias | 0,514 | 0,286 |

Todos os sistemas melhoraram em relação a 2016. O ConceptRank fica cerca de 0,01 à
frente do LexRank, mas sem significância estatística. Os dois ganham com folga de
frases aleatórias, então o grafo funciona, e nenhum dos dois supera simplesmente
pegar as primeiras frases de cada notícia, algo bem conhecido em sumarização de
notícias que eu não tinha medido na época. As duas outras afirmações da tese
(o ConceptRank seria melhor em textos curtos e em textos com pouca variedade de
vocabulário) também não se sustentam.

E o ponto mais honesto de todos: os "conceitos" do ConceptRank nunca foram além
de palavras com stem. A categorização por regras, com sinônimos e termos
relacionados, que a tese descreve, nunca chegou a ser implementada.

## O que eu levo disso

- **Reproduza antes de melhorar.** Sem um ponto de partida medido, não dá para
  saber se uma mudança ajudou.
- **Trave as versões.** O mesmo código com bibliotecas diferentes deu outro resultado.
- **Desconfie de números bons demais** e teste a avaliação com casos óbvios.
- **Sempre compare com baselines triviais.** O Lead teria mudado a conclusão em 2016.
- **Controle os fatores que confundem**, como o tamanho do sumário.
- **Teste estatístico com 50 exemplos é humilde.** Diferenças de 0,01 quase sempre
  são ruído.

Não é o resultado que eu queria ter encontrado, mas é o que os dados dizem, e
agora qualquer pessoa consegue rodar e conferir: cada número do repositório tem um
script que o gera, e os testes garantem que o código antigo foi portado fielmente.
Fiz essa revisão em par com o Claude Code, o que tornou viável cavar tão fundo em
tão pouco tempo.

Se um dia eu voltar ao tema, o caminho é implementar de verdade a parte dos
conceitos, com sinônimos ou embeddings de frases em português. A estrutura para
medir isso direito, pelo menos, já está pronta.
