---
title: "Setting up tests for Django Plugins"
date: 2020-04-04
tags: ["django", "python", "testing"]
comments: true
---

Did you have a good idea for a nice Django plugin? Don’t know how to set up tests for it without creating a whole Django app for your project? Here is the solution for you. This article is written after a [pull request](https://github.com/cuducos/django-public-admin/pull/11) for [Django Public Admin](https://github.com/cuducos/django-public-admin), using [django-stdimage](https://github.com/codingjoe/django-stdimage) as a base.

Django Public Admin is a plugin that makes a public and read-only version of the Django Admin. It uses Poetry to manage packages and dependencies, `pytest` to run tests in addition to `pytest-django`.

### What is Django

Django is a web framework written in Python, and it’s designed for developers to write [reusable apps](https://docs.djangoproject.com/en/3.0/intro/reusable-apps/) (apps in this context means application, which in this case is the project designed using Django):

> “Reusability is the way of life in Python. […] Django itself is also a normal Python package. This means that you can take existing Python packages or Django apps and compose them into your own web project. You only need to write the parts that make your project unique. The idea of these apps.”

In other words, we can wrap reusable parts of any Django app and use them in other Django apps we write. This is the idea and for that we also need tests. At the end of the day, we have a reusable bit of code without the accompanying Django application.

So, how to test a part of a Django app without the full Django app? 🤔

### Understanding what is missing

With the environment already configured, the first issue that appeared was that when running the tests outside of a Django app was that it was complaining that our Django was `ImproperlyConfigured` and that we need to set up the `DJANGO_SETTINGS_MODULE`:

```sh
You must either define the environment variable DJANGO_SETTINGS_MODULE or call settings.configure() before accessing settings.
```

### Setting up `tests/settings.py`

After talking to [Ana Paula Gomes](https://twitter.com/AnaPaulaGomess), she pointed out that she already saw this being implemented in a nice way on another project, and what was being made was to create a `tests/settings.py` file that will act as configuration for Django to run the tests. So for that, I created a file that looks like that:

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

* `BASE_DIR`: This is pointed to the Django project directory.

* [`INSTALLED_APPS`](https://docs.djangoproject.com/en/3.0/ref/settings/#installed-apps): Which contains all the apps necessary to run your set up (don’t forget to add your project here `public_admin` in our target project, and `tests` ), and might vary depending on what you are testing.

* [`SECRET_KEY`](https://docs.djangoproject.com/en/3.0/ref/settings/#secret-key): The `SECRET_KEY` setting must not be empty.

### Setting up `setup.cfg`

After configuring up the `settings.py` we need to tell our application to use this file to run the tests, setting up the `DJANGO_SETTINGS_MODULE` variable in a special file called `setup.cfg`, in our case, we tell that when `[tool:pytest]` is called we should use this settings environment, this might look like that:

```
[tool:pytest]
DJANGO_SETTINGS_MODULE=tests.settings
```

And that is it, now you can set up your tests for Django Plugins without creating an app for testing using `pytest`.

![](https://media.giphy.com/media/l0HlGS7yQ9AUwlY52/giphy.gif)
