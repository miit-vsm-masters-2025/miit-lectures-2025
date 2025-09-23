# Лекция 4

## Заканчиваем с tls-сертификатами
  - Генерируем серты с помощью xca
  - Закидываем серты внутрь контейнера с помощью scp
  - Используем в Caddy https://caddyserver.com/docs/caddyfile/directives/tls
  - Краткий рассказ про letsencrypt
    - Можно получать боевые серты бесплатно
    - Нужно подтвеждать владение доменом: либо ответить на запрос через порт 443, либо использовать DNS-подтверждение
      - Wildcard-сертификат - только через dns-подтверждение
    - Можно использовать caddy или более универсальный вариант с certbot

Пример с certbot и dns:
```
while true; do
  certbot certonly --non-interactive --agree-tos --dns-cloudflare --dns-cloudflare-credentials=/app/cloudflare.ini -d=*.mydomain.ru
  echo Sleeping...
  sleep 1d
done
```

## Удаленное подключение к Docker-демону
- Обычно взаимодействие с докер-демоном происходит через unix-сокет `/var/run/docker.sock`
  - Этот сокет потенциально можно смонтировать внутрь любого докер-контейнера, тем самым выдав доступ к любым действиям с докером (а заодно и рутовые права на хост, так что осторожнее)
- Мы можем заставить демон дополнительно слушать на tcp-сокете, и даже выставить его "в интернет". 
  - Для обеспечения безопасности можно (и нужно) использовать TLS-сертфикаты. Не только серверный, но и клиентский (т.н. mTLS).
  - Для этого нам нужно запустить демон с параметрами `/usr/bin/dockerd -H fd:// -H tcp://0.0.0.0:2376 --tlsverify --tlscacert=/etc/docker/ca.pem --tlscert=/etc/docker/server-cert.pem --tlskey=/etc/docker/server-key.pem`
    - fd:// - фича systemd, можно просто указать `-H unix:///var/run/docker.sock`
  - Для пробрасывания дополнительных параметров можно использовать systemd unit drop-in. См. пример docker-drop-in.conf ниже.
  - После создания drop-in файла нужно заставить systemd перечитать конфиги (systemctl daemon reload) и рестартовать докер (systemctl restart docker).
  - Убедиться, что все стартовало хорошо, можно командами `systemctl status docker` и `journalctl -u docker -f -n 100`

## Подготовка к реализации секурной прокси

- Рассказ о базовых идеях
  - Хотим сделать проксю на go, которая будет проверять https-only куку в запросах от пользователя и, если все ок, проксировать запросы к другим нашим сервисам 
  - Маппинг cookie -> user хотим хранить в Valkey (форк Redis)
    - Если куки в запросе не оказалось (или мы просто не смогли найти переданное значение в valkey) - проставляем ее и создаем в valkey пустой маппинг
    - Кука имеет срок жизни с момента последнего использования (не создания). Обновляем ttl при запросах.
  - У каждого юзера могут быть свои доступные сервисы, прописываем все это в конфиге
- Немного ссылок на инструменты
  - Valkey: https://valkey.io/
  - Go: https://go.dev/
  - Goland (IDE): https://www.jetbrains.com/go/
    - Можно использовать любую IDE с плагином для Golang
  - Библиотеки Go
    - Логи: https://github.com/uber-go/zap
    - Конфиги: https://github.com/ilyakaznacheev/cleanenv
    - Веб-фреймворк: https://gin-gonic.com/
    - Dependency injection: https://github.com/google/wire
      - Но месяц назад Гугл его молча сослал в архив, кто наследник пока неясно: https://www.reddit.com/r/golang/comments/1mnub7p/wire_repo_archived_without_notice/
  - Хорошее место чтобы начать изучение Go: https://go.dev/tour/welcome/1