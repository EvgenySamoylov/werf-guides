---
title: Как использовать гайд
sidebar: applications_guide
guide_code: gitlab_python_django
permalink: gitlab_python_django/000_task.html
toc: false
---

Этот гайд расскажет, как Python разработчику развернуть своё приложение в Kubernetes с помощью утилиты Werf.

![](/applications_guide_ru/images/applications-guide/navigation.svg){:width="800px"}

Обязательны к прочтению главы "Подготовка к работе" и "Базовые настройки" — в них будут разобраны вопросы настройки окружения и основы работы с Werf, сборки и деплоя приложения в production. Однако, чтобы построить серьёзное приложение понадобится чуть больше навыков, раскрытых в других главах.

## Работа с исходными кодами

Для прохождения гайда предоставляется много исходного кода: как самого приложения, которое будет переносится в Kubernetes, так и кода инфраструктуры, связанного с каждой главой. В тексте будут контрольные точки, где вы можете сверить состояние своих исходников с образцом.

Мы рекомендуем сперва пройти гайд с предложенным приложением, разобравшись в механиках сборки и деплоя, и только затем — пробовать перенести в Kubernetes свой код.

## Условные обозначения

В начале каждой главы мы показываем, **какие файлы будут затронуты**:

{% filesused title="Файлы, упомянутые в главе" %}
- just/an/example.yaml
- of/files.yaml
- like.yml
- many.yml
- others.yml
{% endfilesused %}

Для вещей, выходящих за рампки повествования, но полезных для саморазвития, предусмотрены **схлопытвающиеся блоки**, например:

{% offtopic title="Нажми сюда чтобы узнать больше" %}

Это просто пример блока, который может раскрываться. Здесь, внутри, будет дополнительная информация для самых любознательных и желающих разобраться в матчасти.

{% endofftopic %}

В коде можно регулярно встретить **блоки с кодом**. Обратите внимание, что они **почти всегда отображают только часть файла**. Куда вставлять этот кусок текста — объясняется в тексте гайда, а также вы можете нажать на ссылку (в приведённом ниже примере - `deployment.yaml`) и перейти к github с полным исходным кодом файла.

{% snippetcut name="deployment.yaml" url="#" %}
{% raw %}
```yaml
      containers:
      - name: basicapp
        command:
         - echo "OK"
{{ tuple "basicapp" . | include "werf_container_image" | indent 8 }}
```
{% endraw %}
{% endsnippetcut %}


<div id="go-forth-button">
    <go-forth url="010_preparing.html" label="Подготовка к работе" framework="{{ page.label_framework }}" ci="{{ page.label_ci }}" guide-code="{{ page.guide_code }}" base-url="{{ site.baseurl }}"></go-forth>
</div>
