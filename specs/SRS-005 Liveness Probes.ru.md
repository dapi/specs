# SRS-005 Liveness Probes

Приложение позволяет делать пробы своей работоспособности по HTTP или TCP-порту. 

При выполнении Liveness/Readiness проб приложение НЕ должно открывать новые сетевые соединения или
делать запросы по сети. We probably shouldn't be checking the availability of our dependencies in readiness checks due to the potential for cascading failures. 

Startup-проба может быть умной, сложной. Liveness проба должна быть очень простой. Относитесь к Liveness пробе как к способу узнать, нужно ли приложение перезагружать.
Readiness.

Приложение пишет в log, когда приходит запрос на пробу и результат ответа, но только в log-level=`debug`

## Виды проб

* Liveness (работоспособности) - для того, чтобы узнать что контейнер жив. Если эта проба не проходит, то pod перезапустят. Делается по пути `/live`
* Readiness (готовности) - для того, чтобы узнать когда проверяемый контейнер будет готов принимать сетевой трафик. Если эта проба не проходит, то с pod-а снимут трафик. Делается по пути `/health`
* Startup (запущено) - для того чтобы определить когда приложение запустилось и можно переходить к проверкам liveness/readiness. Пока эта проба не состоится, pod не пойдет дальше по жизненному циклу.

## HTTP-пробы

Если приложение поддерживает HTTP-пробы, то порт отвечающего сервера указывается через переменную окружения `HEALTH_PORT`

* Liveness-проба делается по пути `/live`. Данный вид пробы желателен, но не обязателен. Если приложение не поддерживает такой endpoint, то шедулером используется `/health`
* Readiness-проба делается по пути `/health`
* Startup-проба делается по пути `/startup`. Желателен, но не обязателен.

Удачным считается любой ответ со статусом 200.

HTTP-запрос должен укладываться в `30ms` и не зависеть от нагрузки на сервис.

## TCP-пробы

* Порт для liveness-пробы указывается через переменную окружения `LIVENESS_PORT`
* Порт для readiness-пробы указывается через переменную окружения `HEALTH_PORT`

Успешной считается любая проба, которая успешно открыла соединение на нужном порту.

## gRPC-пробы

Подробнее: https://github.com/grpc/grpc/blob/master/doc/health-checking.md

* Порт для liveness-пробы указывается через переменную окружения `LIVENESS_PORT`
* Порт для readiness-пробы указывается через переменную окружения `HEALTH_PORT`


# Больше информации

<img src="https://andrewlock.net/content/images/2020/k8s_probes.svg" />

* [SRS-006 Liveness Probes Command](SRS-006%20Liveness%20Probes%20Command.ru.md)

* [What is difference in Liveness Probe, Readiness Probe and Startup Probe in Kubernetes?](https://medium.com/@edu.ukulelekim/what-is-difference-in-liveness-probe-readiness-probe-and-startup-probe-in-kubernetes-e116c4563c13)
* https://faun.pub/the-difference-between-liveness-readiness-and-startup-probes-781bd3141079
* https://andrewlock.net/deploying-asp-net-core-applications-to-kubernetes-part-6-adding-health-checks-with-liveness-readiness-and-startup-probes/
  
