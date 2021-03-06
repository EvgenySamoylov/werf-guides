---
title: Базовые настройки
sidebar: applications_guide
guide_code: gitlab_python_django
permalink: gitlab_python_django/020_basic.html
toc: false
---

В этой главе мы возьмём приложение, которое будет выводить сообщение "hello world" по http и опубликуем его в kubernetes с помощью Werf. Сперва мы разберёмся со сборкой и добьёмся того, чтобы образ оказался в Registry, затем — разберёмся с деплоем собранного приложения в Kubernetes, и, наконец, организуем CI/CD-процесс силами Gitlab CI.

Если у вас мало опыта с описанием объектов Kubernetes — учтите, что это может занять у вас больше времени, чем организация сборки и CI-процесса с помощью werf. Это нормально и мы постарались облегчить для вас этот процесс.

Наше приложение будет состоять из одного docker образа собранного с помощью werf. В этом образе будет работать один основной процесс, который запустит gunicorn. Управлять маршрутизацией запросов к приложению будет Ingress в Kubernetes кластере. Мы реализуем два стенда: [production]({{ site.docsurl }}/documentation/reference/ci_cd_workflows_overview.html#production) и [staging]({{ site.docsurl }}/documentation/reference/ci_cd_workflows_overview.html#staging).

<div id="go-forth-button">
    <go-forth url="020_basic/10_build.html" label="Сборка" framework="{{ page.label_framework }}" ci="{{ page.label_ci }}" guide-code="{{ page.guide_code }}" base-url="{{ site.baseurl }}"></go-forth>
</div>
