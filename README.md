## Referência para o meu blog.

Feito com [Hugo](https://gohugo.io/) e o tema [Beautiful Hugo](https://github.com/halogenica/beautifulhugo) (como git submodule em `themes/beautifulhugo`).

## Setup

```
brew install hugo
git clone --recurse-submodules git@github.com:anaschwendler/blog.git
```

Se já clonou sem os submodules:

```
git submodule update --init
```

## Flow geral

1. Adiciona a postagem para `content/en` (se inglês) ou `content/br` (se português)
2. Commita e pusha para esse repositório
3. Github Actions vai fazer o build e o commit para `anaschwendler.github.io` que é onde está o Github Pages

## Como roda o blog:

```
hugo server
```

A pasta `public/` é gerada pelo build e não é commitada.

## Como atualiza:

- Tema: o Dependabot abre PR mensal atualizando o submodule (ou manualmente: `git submodule update --remote`)
- Hugo: atualiza `hugo-version` em `.github/workflows/deploy.yml` (e `brew upgrade hugo` local)

Customizações de layout vão em `layouts/` na raiz (sobrescrevem o tema); não edita nada dentro de `themes/`.

<!-- Ana do passado para a Ana do futuro: em 2026 o tema virou submodule e o Hugo foi de 0.64 para 0.169. Antes disso o tema inteiro estava copiado em layouts/, i18n/, data/ e static/. -->
