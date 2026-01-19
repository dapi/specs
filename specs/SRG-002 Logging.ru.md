# SRG-002 Logging (Журналирование)

Этот документ описывает принципы ведения лога приложения.

1. Приложения пишут свой лог в `STDOUT`.
2. Приложение выводит стартовое сообщение со своим именем и версией сразу первой
   строкой к функции main *ДО всех инициализаций, аллокации памяти и установки
   соединений*. Например: `Starting Blockberry v1.2.3`.
3. Приложение пишет в log момент, когда оно инициализировано и готово принимать
   запросы/выполнять работу: `Initialization completed. Ready to work.`
3. Сообщение пишется в log ОДНОЙ строкой (избегать многострочных сообщений).
4. Длина записи не должна превышать 10Kb.
5. Добавлять к сообщениям `correlation_id`
6. Делить сообщения больше 10Gb на части и искать по `correlation_id`
7. Использовать UID между сервисами
8. Ограничить поля логов только самыми необходимыми
9. Избегать многострочных сообщений
10. Не писать в сообщения пароли и другую чувствительную информацию
11. Категорировать сообщения по уровням (INFO, ERROR, DEBUG и тд)
12. Приложение пишет в лог о своем завершении и причине [см. SRS-008 Graceful Shutdown](SRS-008%20Graceful%20Shutdown.ru.md)

# Форматы записей в логе

timestamp пишется в формате ISO8601 с указанием временной зоны.

## Текстовый формат

Текстовая строка в кодировке UTF-8. Формат:

```
 <DATE> <LOG_SEVERITY_SYMBOL> <MESSAGE>
```

Пример:

```
2024-12-12T19:31:35+00:00 INFO Starting Application v1.2.3
```

## JSON

Обязательные поля:

* `timestamp`
* `message` - текстовая строка с сообщением
* `level` - уровень записи 

допускаются также любые другие поля.

Пример:

```
{"timestamp":"2024-12-12T19:31:35+00:00","message":"Starting Blockbery v1.2.3","level": "INFO", "extra":"Other info"}
```
## Date/Timestamp

Допустимые значения даты/времени: https://docs.datadoghq.com/logs/log_configuration/parsing/?tab=matchers#parsing-dates

Рекомендуемый формат: `yyyy-MM-dd'T'HH:mm:ss.SSSZ` пример: `2016-11-29T16:21:36.431+0000`

## Severity/Level

Допустимые значения для level
* https://en.wikipedia.org/wiki/Syslog#Severity_level
* https://docs.datadoghq.com/logs/log_configuration/processors/?tab=ui#log-status-remapper

# Best practices

* [Logging best practices in an app or add-on for Splunk Enterprise](https://dev.splunk.com/enterprise/docs/developapps/addsupport/logging/loggingbestpractices/)
* [Logging examples in an app or add-on for Splunk Enterprise](https://dev.splunk.com/enterprise/docs/developapps/addsupport/logging/logginexamples/)
* [Logging Best Practices: The 13 You Should Know](https://www.scalyr.com/blog/the-10-commandments-of-logging/)
