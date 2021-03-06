---
title: Конфигурирование IaC
sidebar: applications_guide
guide_code: gitlab_python_django
permalink: gitlab_python_django/020_basic/20_iac.html
toc: false
---

{% filesused title="Файлы, упомянутые в главе" %}
- .helm/templates/deployment.yaml
- .helm/templates/ingress.yaml
- .helm/templates/service.yaml
- .helm/values.yaml
- .helm/secret-values.yaml
{% endfilesused %}

Для того, чтобы приложение заработало в Kubernetes — необходимо описать инфраструктуру приложения как код, т.е. в нашем случае объекты kubernetes: Pod, Service и Ingress.

Конфигурацию для Kubernetes нужно шаблонизировать. Один из популярных инструментов для такой шаблонизации — это Helm, и движок Helm-а встроен в Werf. Помимо этого, werf предоставляет возможности работы с секретными значениями, а также дополнительные Go-шаблоны для интеграции собранных образов.

В этой главе мы научимся описывать helm-шаблоны, используя возможности werf, а также освоим встроенные инструменты отладки.

### Составление конфигов инфраструктуры

На сегодняшний день [Helm](https://helm.sh/) один из самых удобных способов которым вы можете описать свой deploy в Kubernetes. Кроме возможности установки готовых чартов с приложениями прямиком из репозитория, где вы можете введя одну команду, развернуть себе готовый Redis, Postgres, Rabbitmq прямиком в Kubernetes, вы также можете использовать Helm для разработки собственных чартов с удобным синтаксисом для шаблонизации выката ваших приложений.

Потому для werf это был очевидный выбор использовать такую технологию.

{% offtopic title="Что делать, если вы не работали с Helm?" %}

Мы не будем вдаваться в подробности [разработки yaml манифестов с помощью Helm для Kubernetes](https://habr.com/ru/company/flant/blog/423239/). Осветим лишь отдельные её части, которые касаются данного приложения и werf в целом. Если у вас есть вопросы о том как именно описываются объекты Kubernetes, советуем посетить страницы документации по Kubernetes с его [концептами](https://kubernetes.io/ru/docs/concepts/) и страницы документации по разработке [шаблонов](https://helm.sh/docs/chart_template_guide/) в Helm.

Работа с Helm и конфигурацией для Kubernetes может быть очень сложной первое время из-за нелепых мелочей, таких, как опечатки или пропущенные пробелы. Если вы только начали осваивать эти технологии — постарайтесь найти ментора, который поможет вам преодолеть эти сложности и посмотрит на ваши исходники сторонним взглядом.

В случае затруднений, пожалуйста убедитесь, что вы:

- Понимаете, как работает [indent](https://helm.sh/docs/chart_template_guide/function_list/#indent)
- Понимаете, что такое конструкция [tuple](https://helm.sh/docs/chart_template_guide/control_structures/)
- Понимаете, как Helm работает с хэш-массивами 
- Очень внимательно следите за пробелами в Yaml
{% endofftopic %}

Для работы нашего приложения в среде Kubernetes понадобится описать сущности Deployment (который породит в кластере Pod), Service, направить трафик на приложение, донастроив роутинг в кластере с помощью сущности Ingress. И не забыть создать отдельную сущность Secret, которая позволит нашему Kubernetes скачивать собранные образа из registry.

#### Создание Pod-а

{% filesused title="Файлы, упомянутые в главе" %}
- .helm/templates/deployment.yaml
- .helm/values.yaml
{% endfilesused %}

Для того, чтобы в кластере появился Pod с нашим приложением — мы пропишем объект Deployment. У создаваемого Pod будет один контейнер `basicapp`. Укажем, **как этот контейнер будет запускаться**:

{% snippetcut name="deployment.yaml" url="#" %}
{% raw %}
```yaml
      containers:
      - name: basicapp
        command: ['gunicorn', 'django_example.wsgi:application', '--bind', '0.0.0.0:8001', '--access-logfile', '-', '--log-level', 'debug']
{{ tuple "basicapp" . | include "werf_container_image" | indent 8 }}
```
{% endraw %}
{% endsnippetcut %}

Обратите внимание на вызов [`werf_container_image`]({{ site.docsurl }}/documentation/reference/deploy_process/deploy_into_kubernetes.html#werf_container_image). Данная функция генерирует ключи `image` и `imagePullPolicy` со значениями, необходимыми для соответствующего контейнера пода и это позволяет гарантировать перевыкат контейнера тогда, когда это нужно.

{% offtopic title="А в чём проблема?" %}
Kubernetes не знает ничего об изменении контейнера — он действует на основании описания объектов и сам выкачивает образы из Registry. Поэтому Kubernetes-у нужно в явном виде сообщать, что делать.

Werf складывает собранные образы в Registry с разными именами, в зависимости от выбранной стратегии тегирования и деплоя — подробнее это мы разберём в главе про CI. И, как следствие, в описание контейнера нужно пробрасывать правильный путь до образа, а также дополнительные аннотации, связанные со стратегией деплоя.

Подробнее - можно посмотреть в [документации]({{ site.docsurl }}/documentation/reference/deploy_process/deploy_into_kubernetes.html#werf_container_image).
{% endofftopic %}

Для корректной работы нашего приложения ему нужно узнать **переменные окружения**.

Для Django это, например, `DEBUG` и `SECRET_KEY`

{% snippetcut name="deployment.yaml" url="#" %}
{% raw %}
```yaml
      env:
      - name: "DEBUG"
        value: "1" 
      - name: "SECRET_KEY"
        value: "seafij-hawefphiqwf-gqwiuf" 
{{ tuple "basicapp" . | include "werf_container_env" | indent 8 }}
```
{% endraw %}
{% endsnippetcut %}


Мы задали значение для `SECRET_KEY` в явном виде — и это абсолютно не безопасный путь для хранения таких критичных данных. Мы разберём более правильный путь ниже, в главе "Разное поведение в разных окружениях".

Обратите также внимание на функцию [`werf_container_env`]({{ site.docsurl }}/documentation/reference/deploy_process/deploy_into_kubernetes.html#werf_container_env) — с помощью неё Werf вставляет в описание объекта служебные переменые окружения.

<a name="helm-values-yaml" />

{% offtopic title="А как динамически подставлять в переменные окружения нужные значения?" %}

Helm — шаблонизатор, и он поддерживает множество инструментов для подстановки значений. Один из центральных способов — подставлять значения из файла `values.yaml`. Наша конструкция могла бы иметь вид

{% snippetcut name="deployment.yaml" url="#" %}
{% raw %}
```yaml
      env:
      - name: SECRET_KEY
        value: {{ .Values.app.secret_key }}
```
{% endraw %}
{% endsnippetcut %}

или даже более сложный, для того, чтобы значение основывалось на текущем окружении:

{% snippetcut name="deployment.yaml" url="#" %}
{% raw %}
```yaml
      env:
      - name: SECRET_KEY
        value: {{ pluck .Values.global.env .Values.app.secret_key | first | default .Values.app.secret_key._default }}
```
{% endraw %}
{% endsnippetcut %}

{% snippetcut name="values.yaml" url="#" %}
```yaml
app:
  secret_key:
    _default: 9eeddad83cebe240f55ae06ccdd95f8e
    production: 684e5cb73034052dc89e3055691b7ac4
    testing: 189af8ca60b04e529140ec114175f098
```
{% endsnippetcut %}

{% endofftopic %}


При запуске приложения в Kubernetes **логи необходимо отправлять в stdout и stderr** - это нужно для простого сбора логов например через `filebeat`, а так же чтобы не разрастались docker образы запущенных приложений.

Для того чтобы логи приложения отправлялись в stdout нам необходимо сконфигурировать это в коде приложения. Добавьте в файл настроек словарь:

```javascript
LOGGING = {
    'version': 1,
    'disable_existing_loggers': False,
    'handlers': {
        'console': {
            'class': 'logging.StreamHandler',
        },
    },
    'root': {
        'handlers': ['console'],
        'level': 'WARNING',
    },
}
```

[Подбробно про логирование](https://docs.djangoproject.com/en/3.0/topics/logging/)

#### Доступность Pod-а

{% filesused title="Файлы, упомянутые в главе" %}
- .helm/templates/deployment.yaml
- .helm/templates/service.yaml
- .helm/values.yaml
{% endfilesused %}

Для того чтобы запросы извне попали к нам в приложение нужно открыть порт у Pod-а, создать объект Service и привязать его к Pod-у, и создать объект Ingress.

{% offtopic title="Что за объект Ingress и как он связан с балансером?" %}
Возможна коллизия терминов:

* Есть Ingress — в смысле [NGINX Ingress Controller](https://github.com/kubernetes/ingress-nginx), который работает в кластере и принимает входящие извне запросы
* И есть [объект Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/), который фактически описывает настройки для NGINX Ingress Controller

В статьях и бытовой речи оба этих термина зачастую называют "Ingress", так что нужно догадываться по контексту.
{% endofftopic %}

Наше приложение работает на стандартном порту `3000` — **откроем порт Pod-у**:

{% snippetcut name="deployment.yaml" url="#" %}
```yaml
        ports:
        - containerPort: 3000
          name: http
          protocol: TCP
```
{% endsnippetcut %}

Затем, **пропишем Service**, чтобы к Pod-у могли обращаться другие приложения кластера.

{% snippetcut name="service.yaml" url="#" %}
{% raw %}
```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: {{ .Chart.Name }}
spec:
  selector:
    app: {{ .Chart.Name }}
  ports:
  - name: http
    port: 3000
    protocol: TCP
```
{% endraw %}
{% endsnippetcut %}

Обратите внимание на поле `selector` у Service: он должен совпадать с аналогичным полем у Deployment и ошибки в этой части — самая частая проблема с настройкой маршрута до приложения.

{% snippetcut name="deployment.yaml" url="#" %}
{% raw %}
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Chart.Name }}
spec:
  selector:
    matchLabels:
      app: {{ .Chart.Name }}
```
{% endraw %}
{% endsnippetcut %}

{% offtopic title="Как убедиться, что выше всё сделано правильно?" %}

Мы будем деплоить наше приложение в Kubernetes позже, но если после деплоя у вас возникли проблемы в этом месте, то вернитесь сюда и проведите проверку описанную ниже.

Попробуйте получить `endpoint` сервиса в нужном вам окружении.
Если в нем будет фигурировать ip пода, значит вы все правильно настроили. А если нет, то проверьте еще раз совпадают ли у вас поля selector в сервисе и деплойменте.
Название эндпоинта совпадает с названием сервиса.

Пример команды:

{% raw %}
`kubectl -n <название окружения> get ep {{ .Chart.Name }}`
{% endraw %}

{% endofftopic %}

После этого можно настраивать **роутинг на Ingress**. Укажем, на какой домен и путь, в какой сервис и на какой порт направлять запросы.

{% snippetcut name="ingress.yaml" url="#" %}
{% raw %}
```yaml
  rules:
  - host: mydomain.io
    http:
      paths:
      - path: /
        backend:
          serviceName: {{ .Chart.Name }}
          servicePort: 3000
```
{% endraw %}
{% endsnippetcut %}

#### Разное поведение в разных окружениях

Некоторые настройки хочется видеть разными в разных окружениях. К примеру, домен, на котором будет открываться приложение должен быть либо staging.mydomain.io, либо mydomain.io — смотря куда мы задеплоились.

В werf для этого существует три механики:

1. Подстановка значений из `values.yaml` по аналогии с Helm
2. Проброс значений через аттрибут `--set` при работе в CLI-режиме, по аналогии с Helm
3. Подстановка секретных значений из `secret-values.yaml`

**Вариант с `values.yaml`** рассматривался ранее в главе ["Создание Pod-а"](#helm-values-yaml).

Второй вариант подразумевает **задание переменных через CLI** `werf deploy --set "global.ci_url=mydomain.io"`, которое затем будет доступно в yaml-ах в виде {% raw %}`{{ .Values.global.ci_url }}`{% endraw %}.

Этот вариант удобен для проброски, например, имени домена для каждого окружения

{% snippetcut name="ingress.yaml" url="#" %}
{% raw %}
```yaml
  rules:
  - host: {{ .Values.global.ci_url }}
```
{% endraw %}
{% endsnippetcut %}

<a name="secret-values-yaml" />Отдельная проблема — **хранение и задание секретных переменных**, например, учётных данных аутентификации для сторонних сервисов, API-ключей и т.п.

Так как Werf рассматривает git как единственный источник правды — правильно хранить секретные переменные там же. Чтобы делать это корректно — мы [храним данные в шифрованном виде]({{ site.docsurl }}/documentation/reference/deploy_process/working_with_secrets.html). Подстановка значений из этого файла происходит при рендере шаблона, который также запускается при деплое.

Чтобы продолжать дальше

* [Сгенерируйте ключ]({{ site.docsurl }}/documentation/cli/management/helm/secret/generate_secret_key.html) (`werf helm secret generate-secret-key`)
* Задайте ключ в переменных приложения, в текущей сессии консоли (например, `export WERF_SECRET_KEY=504a1a2b17042311681b1551aa0b8931z`)
* Пропишите полученный ключ в Variables для вашего репозитория в Gitlab (раздел `Settings` - `CI/CD`), название переменной `WERF_SECRET_KEY`

![](/applications_guide_ru/images/applications-guide/020-werf-secret-key-in-gitlab.png)

После этого мы сможем задать секретную переменную `secret_key`. Зайдите в режим редактирования секретных значений:

```bash
$ werf helm secret values edit .helm/secret-values.yaml
```

Откроется консольный текстовый редактор с данными в расшифованном виде:

{% snippetcut name="secret-values.yaml в расшифрованном виде" url="#" %}
```yaml
app:
  s3:
    access_key:
      _default: bNGXXCF1GF
    secret_key:
      _default: zpThy4kGeqMNSuF2gyw48cOKJMvZqtrTswAQ
```
{% endsnippetcut %}

После сохранения значения в файле зашифруются и примут примерно такой вид:

{% snippetcut name="secret-values.yaml в зашифрованном виде" url="#" %}
```yaml
app:
  s3:
    access_key:
      _default: 100063ead9354e0259b4cd224821cc4c57adfbd63f9d10cb1e828207f294a958d0e9
    secret_key:
      _default: 1000307fe833d63f467597d10e00ef7b321badd5d02b2f9823b0babfa7265c03e0967e434c1a496274825885e6d3645764fe097e3f463cbde27b4552b9376a78ba5f
```
{% endsnippetcut %}

<a name="iac-debug-deploy" />

### Отладка конфигов инфраструктуры и деплой в Kubernetes

После того, как написана основная часть конфигов — хочется проверить корректность конфигов и задеплоить их в Kubernetes. Для того, чтобы отрендерить конфиги инфраструктуры нужны сведения об окружении, на которое будет произведён деплой, ключ для расшифровки секретных значений и т.п.

Если мы запускаем Werf вне Gitlab CI — нам нужно сделать несколько операций вручную прежде чем Werf сможет рендерить конфиги и деплоить в Kubernetes.

* Вручную подключиться к gitlab registry с помощью [`docker login`](https://docs.docker.com/engine/reference/commandline/login/) (если ранее это не сделано)
* Установить переменную окружения `WERF_IMAGES_REPO` с путём до Registry (вида `registry.mydomain.io/myproject`)
* Установить переменную окружения `WERF_SECRET_KEY` со значением, [сгенерированным ранее в главе "Разное поведение в разных окружениях"](#secret-values-yaml)
* Установить переменную окружения `WERF_ENV` с названием окружения, в которое будет осуществляться деплой. Вопроса разных окружений мы коснёмся подробнее, когда будем строить CI-процесс, сейчас — просто установим значение `staging`. **Важно удалить эту переменную в финальном варианте деплоя** — иначе деплой всегда будет идти в один и тот же namespace.

Если вы всё правильно сделали, то вы у вас корректно будут отрабатывать команды [`werf helm render`]({{ site.docsurl }}/documentation/cli/management/helm/render.html) и [`werf deploy`]({{ site.docsurl }}/documentation/cli/main/deploy.html). _Примечание: при локальном запуске эти команды могут жаловаться на нехватку данных, которые в ином случае были бы проброшены из CI. Например, на данные о теге собранного образа. Это нормально._

{% offtopic title="Как вообще работает деплой" %}

Werf (по аналогии с helm) берет yaml шаблоны, которые описывают объекты Kubernetes, и генерирует из них общий манифест. Манифест отдается API Kubernetes, который на его основе внесет все необходимые изменения в кластер. Werf отслеживает как Kubernetes вносит изменения и сигнализирует о результатах в реальном времени. Все это благодаря встроенной в werf библиотеке [kubedog](https://github.com/flant/kubedog). Уже сам Kubernetes занимается выкачиванием нужных образов из Registry и запуском их на нужных серверах с указанными настройками.

{% endofftopic %}

Запустите деплой и дождитесь успешного завершения

```bash
werf deploy --stages-storage :local
```

Проверить что приложение задеплоилось в кластер можно с помощью kubectl, вы увидите что-то вида:

```bash
$ kubectl get namespace
NAME                                 STATUS               AGE
default                              Active               161d
werf-guided-project-production       Active               4m44s
werf-guided-project-staging          Active               3h2m
```

{% offtopic title="Как формируется имя namespace-а?" %}

По шаблону `[[ project ]]-[[ env ]]`, где `[[ project ]]` — имя проекта, а `[[ env ]]` — имя окружения. Подробнее можно почитать [в документации]({{ site.docsurl }}/documentation/configuration/deploy_into_kubernetes.html#namespace-%D0%B2-kubernetes)

При необходимости namespace можно переназначить.
{% endofftopic %}

```bash
$ kubectl -n example-1-staging get po
NAME                                 READY                STATUS   RESTARTS  AGE
werf-guided-project-9f6bd769f-rm8nz  1/1                  Running  0         6m12s
```

```bash
$ kubectl -n example-1-staging get ingress
NAME                                 HOSTS                ADDRESS  PORTS     AGE
werf-guided-project                  staging.mydomain.io           80        6m18s
```

А также вы должны увидеть ваш сервис через браузер.

<div id="go-forth-button">
    <go-forth url="30_ci.html" label="Построение CI-процесса" framework="{{ page.label_framework }}" ci="{{ page.label_ci }}" guide-code="{{ page.guide_code }}" base-url="{{ site.baseurl }}"></go-forth>
</div>
