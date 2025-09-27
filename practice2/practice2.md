# Secure proxy

## Что нужно сделать

Написать секурную проксю, которая будет защищать доступ к нашим внутренним сервисам.

## Основные идеи

- Аутентификация пользователя будет по username + [TOTP](https://en.wikipedia.org/wiki/Time-based_one-time_password)
- Настройки (в т.ч. список юзеров, их totp-секретов) храним в конфиг файле. См. пример ниже.
- После успешного входа сохраняем в браузер пользователю Cookie с ключом сессии с опциями Secure, HttpOnly.
- Маппинг `sessionKey -> username` храним в Valkey (опенсорсный Redis). Обеспечиваем TTL с момента последнего
  использования сессии (не с момента ее создания).
- Прямо при разработке используем доменные имена (можно прописать в `C:\Windows\System32\drivers\etc\hosts`) и
  self-signed TLS-сертификаты

## Пример конфиг-файла

```yaml
sessions:
  cookieDomain: .secure-proxy.lan
  cookieName: SECURE_PROXY_SESSION
  ttlSeconds: 180
users:
  - username: user1
    totpSecret: FC5PJKPSPKIO5HE2Y5YDIEJJ3ZOJ5J3K
    availableDomains:
      - site1.secure-proxy.lan
upstreams:
  - host: site1.secure-proxy.lan # должен быть поддоменом .secure-proxy.lan, чтобы мы могли увидеть куку
    destination: http://127.0.0.1:8081 # куда проксировать успешно аутентифицированные запросы
```

## Пример сценария работы

1. Пользователь заходит на сайт https://site1.secure-proxy.lan.
2. Бэкенд видит, что в запросе нет куки SECURE_PROXY_SESSION (или она устарела и не находится в Valkey)
3. Бэкенд отвечает редиректом https://auth.secure-proxy.lan, где пользователь видит форму для ввода логина и пароля.
    - Адрес для редиректа после успешного входа прикапывается в query-параметры запроса к https://auth.secure-proxy.lan.
        - Получается что-то типа `https://auth.secure-proxy.lan/?redirectUrl=https%3A%2F%2Fsite1.secure-proxy.lan`
4. Пользователь вводит юзернейм и TOTP-код, бэк его проверяет. Если все ок - проставляет куку SECURE_PROXY_SESSION и
   редиректит на адрес, указанный в
   query-параметре `redirectUrl`
5. На этот раз при запросе к https://site1.secure-proxy.lan бэк видит куку SECURE_PROXY_SESSION со свежим значением и
   проксирует запрос к соответствующему апстриму (например http://127.0.0.1:8081)
    - Еще тут важно не забыть обновить TTL записи в Valkey, чтобы он продолжил отсчитываться с момента последнего
      использования (а не создания) сессии.

## Рекомендуемые инструменты и библиотеки

- Go: https://go.dev/
    - Хорошее место чтобы начать изучение Go: https://go.dev/tour/welcome/1
- Valkey: https://valkey.io/
- Библиотеки Go
    - Логи: https://github.com/uber-go/zap
    - Конфиги: https://github.com/ilyakaznacheev/cleanenv
    - Веб-фреймворк: https://gin-gonic.com/
    - Valkey client: https://valkey.io/clients/#go
    - TOTP: https://pkg.go.dev/github.com/pquerna/otp
    - Фактическое проксирование запросов к upstream: https://pkg.go.dev/net/http/httputil#NewSingleHostReverseProxy

## Предложения по последовательности действий

1. Создать проект, настроить git
2. Создать пару доменных имен (например auth.secure-proxy.lan и site1.secure-proxy.lan). Убедиться, что они резолвятся (
   браузером или командой nslookup).
    - Использовать ли /etc/hosts или взять какой-нибудь свой реальный домен - на ваш выбор.
3. Сгенерировать самоподписанные tls-сертификаты (рекомендую использовать [XCA](https://www.hohnstaedt.de/xca/))
4. Настроить GIN, настроить использование tls-сертификатов в нем. Убедиться, что ваши домены успешно открываются по
   https
5. Попробовать взаимодействие с TOTP-библиотекой в рамках тестов: сгенерировать ключ, сгенерировать TOTP-код на его
   основе.
6. Попробовать проставить куку в браузере (пока просто при переходе по соответствующему эндпойнту). И прочитать ее при
   переходе по другому эндпойнту.
7. Развернуть valkey в докер-контейнере, попробовать взаимодействие с ним (сначала в cli, потом из go-приложения)
    - Нам понадобятся
      команды [SET](https://valkey.io/commands/set/), [GET](https://valkey.io/commands/get/), [EXPIRE](https://valkey.io/commands/expire/)
8. Развернуть рядом еще одно go-приложение (или любое другое) на другом порту (8081). Попробовать проксировать туда
   запросы с помощью [NewSingleHostReverseProxy](https://pkg.go.dev/net/http/httputil#NewSingleHostReverseProxy)
9. Попробовать html-рендеринг (понадобится для отрисовки формы для ввода логина и
   TOTP-кода): https://gin-gonic.com/en/docs/examples/html-rendering/
10. Начинаем непосредственно реализацию основной логики (все необходимые для реализации механики мы уже попробовали,
    осталось собрать все в кучу).