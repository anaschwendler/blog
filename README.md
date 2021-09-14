## Referência para o meu blog.

## Como usa:

[Base para como usar theme](https://gohugo.io/getting-started/quick-start/)

[Base para deploy via Github Actions](https://ruddra.com/hugo-deploy-static-page-using-github-actions/)

As mudanças são feitas aqui nesse repositório de blog, vide flow

## Flow geral

1. Adiciona a postagem para `en` (se inglês) ou `br` (se português)
2. Commita e pusha para esse repositório
3. Github Actions vai fazer o commit para `anaschwendler.github.io` que é onde está o Github Pages

## Como roda o blog:

```
hugo server
```

Ainda se comita as mudanças pra cá

## Como faz deploy:

Atualmente o deploy é feito via Github Actions :tada:

E o que o Github Actions faz é fazer o deploy para o repositório principal do blog `anaschwendler.github.io`

<!-- Ana do passado para a Ana do futuro: tu mudou isso em 2021 para fazer via Github Actions, deixou o comentário apenas para referência. Tu não vai lembrar disso no futuro. -->
<!-- Isso faz o deploy do blog (pasta `/public`) para meu repositório do github.io:

```
./deploy.sh "Mensagem de commit"
``` -->
