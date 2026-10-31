---
title: "Configurando testes para Plugins Django"
date: 2020-04-04
tags: ["django", "python", "testing"]
comments: true
---

Teve uma ideia bacana de Plugin para Django? Não sabe como configurar testes sem criar uma aplicação Django no seu projeto? Seus problemas acabaram. Esse artigo foi escrito a partir de um[pull request](https://github.com/cuducos/django-public-admin/pull/11) para [Django Public Admin](https://github.com/cuducos/django-public-admin), usando [django-stdimage](https://github.com/codingjoe/django-stdimage) como base.

Django Public Admin é um plugin que ajusta o admin de sua aplicação Django como pública e em modo de apenas leitura. Ele usa Poetry para gerenciar pacotes e dependencias,`pytest` para rodar os testes em conjunto com `pytest-django`.

### O que é Django

Django é um web framework written in Python, scrito in Python, e é projetado para desenvolvedores escreverem [aplicações reutilizáveis](https://docs.djangoproject.com/en/3.0/intro/reusable-apps/) (apps in this context means application, which in this case is the project designed using Django):

> “Reusability is the way of life in Python. […] Django itself is also a normal Python package. This means that you can take existing Python packages or Django apps and compose them into your own web project. You only need to write the parts that make your project unique. The idea of these apps.”

Em outras palavras, podemos agrupar parte reusáveis de qualquer aplicação Django e usar este mesmo pedaço de código em outras aplicações que escrevemos.É por causa disso que precisamos testar nossa aplicação. No fim do dia, teremos um pedaço de código que não faz parte da aplicação Django original.

Então, como testar essa parte de código sem criar uma aplicação do zero? 🤔

### Entendendo o que está faltando

Com o ambiente já configurado, o primeiro erro que apareceu no terminal ao rodarmos os testes localmente, dizia que o Django estava mal configurado (`ImproperlyConfigured`) e que precisávamos definir a variável `DJANGO_SETTINGS_MODULE`:

```sh
You must either define the environment variable DJANGO_SETTINGS_MODULE or call settings.configure() before accessing settings.
```

### Configurando `tests/settings.py`

Depois de conversar com a [Ana Paula Gomes](https://twitter.com/AnaPaulaGomess), sela comentou que já tinha visto uma implementação em outro projeto que, e que isso era feito criando um arquivo chamado `tests/settings.py` que serve como arquivo de configuração para o Django rodar os testes. Para isso, eu criei um arquivo que se parece com o abaixo:

```python
import os

BASE_DIR = os.path.dirname(os.path.abspath(__file__))

INSTALLED_APPS = (
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.sites',
    'public_admin',
    'tests'
)

SECRET_KEY = 'foobar'
```

It does contain the [default Django configuration variables](https://docs.djangoproject.com/en/3.0/ref/settings/) as below:

* `BASE_DIR`: Essa variável aponta para a pasta onde o projeto Django está.

* [`INSTALLED_APPS`](https://docs.djangoproject.com/en/3.0/ref/settings/#installed-apps): Contém todas as aplicações extra necessárias para rodar seu ambiente (não esqueça de adicionar o seu projeto, nesse caso `public_admin` e a pasta onde estão os testes, `tests`), e varia de acordo com o que está sendo testado.

* [`SECRET_KEY`](https://docs.djangoproject.com/en/3.0/ref/settings/#secret-key): Esta chave precisa ser sempre configurada.

### Configurando `setup.cfg`

Depois de configurar o arquivo `settings.py` precisamos dizer a nossa aplicação que use este arquivo para rodar os testes, configurando a variável `DJANGO_SETTINGS_MODULE` em um arquivo especial chamado `setup.cfg`, no nosso caso, dizemos que quando `[tool:pytest]` é chamado, devemos usar esse conjunto de configurações, que devem se parecer com:

```
[tool:pytest]
DJANGO_SETTINGS_MODULE=tests.settings
```

E é isso, agora você já pode configurar seus testes para Plugins Django sem usar uma aplicação inteira, usando `pytest`.

![](/img/gifs/l0HlGS7yQ9AUwlY52.gif)
