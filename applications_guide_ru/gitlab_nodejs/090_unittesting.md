---
title: Юнит-тесты и линтеры
permalink: gitlab_nodejs/090_unittesting.html
---

{% filesused title="Файлы, упомянутые в главе" %}
- .gitlab-ci.yml
{% endfilesused %}

В этой главе мы настроим в нашем базовом приложении выполнение тестов/линтеров. Запуск тестов и линтеров - это отдельная стадия в пайплайне Gitlab CI, для выполнения которой может требоваться соблюдение определенных условий. Рассмотрим на примере [линтера ESLint](https://eslint.org/) для языка программирования JavaScript (написан на Node.js).

Требуется добавить эту зависимость в `package.json`, создать к нему конфигурационный файл `.eslintrc.json` и прописать выполнение задания отдельной стадией на GitLab Runner командой [werf run]({{ site.docsurl }}/documentation/cli/main/run.html).

{% snippetcut name=".gitlab-ci.yml" url="https://github.com/werf/werf-guides/blob/master/examples/gitlab-nodejs/090-unittesting/.gitlab-ci.yml" %}
{% raw %}
```yaml
Run Tests:
  stage: test
  script:
    - werf run basicapp -- npm run pretest
  except:
    - schedules
  tags:
    - werf
  dependencies:
    - Build
```
{% endraw %}
{% endsnippetcut %}

Созданную стадию нужно добавить в список стадий:

{% snippetcut name=".gitlab-ci.yml" url="https://github.com/werf/werf-guides/blob/master/examples/gitlab-nodejs/090-unittesting/.gitlab-ci.yml" %}
{% raw %}
```yaml
stages:
- build
- test
- deploy
```
{% endraw %}
{% endsnippetcut %}

Обратите внимание, что процесс будет выполняться на runner'е, внутри собранного контейнера, но без доступа к базе данных и каким-либо ресурсам Kubernetes-кластера.

{% offtopic title="А если нужно больше?" %}
Если нужен доступ к ресурсам кластера или база данных — это уже не линтер: потребуется собрать отдельный образ и прописать сложный сценарий деплоя объектов Kubernetes. Эти случаи выходят за рамки нашего самоучителя для начинающих.
{% endofftopic %}

<div id="go-forth-button">
    <go-forth url="110_multipleapps.html" label="Несколько приложений в одном репозитории" framework="{{ page.label_framework }}" ci="{{ page.label_ci }}" guide-code="{{ page.guide_code }}" base-url="{{ site.baseurl }}"></go-forth>
</div>
