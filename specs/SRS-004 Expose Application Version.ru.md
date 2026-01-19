# SRS-004 Expose Application Version

Для контроля конфигурации запущенных приложений в микросервисной архитектуре
необходимо иметь возможность получать от приложения сведения о его версии.

Приложение сообщает о своей версии следующими способами:

## 1. Ответ на запрос GET /version

Приложения отвечают на HTTP GET запрос на всех HTTP портах (исключая health
port, включая metric-порт и все функциональные порты) с путем `/version` и отдают:

* Свое название в поле `name`
* Свою версию (AppVersion) в поле `version`
* `deployment_info` - информация из переменной окружения `DEPLOYMENT_INFO` с
  которой запущено приложение.

Пример:


```sh
curl http://localhost/version
{"version":"v1.2.3","deployment_info":"image:tag","name":"blockberry"}
```

## 2. В каждый ответ (включая ответ с ошибкой) добавляет HTTP заголовки

* `X-App-Name`
* `X-App-Version`

Например:

```sh
X-App-Name: myapp
X-App-Version: v0.1.0
```



