# ВНИМАНИЕ! ПРОЕКТ ПЕРЕЕХАЛ НА GITFLIC git@gitflic.ru:solo-it/telegram-nginx-docker-webhook.git



# Python Telegram бот на webhook в docker на любом VPS/VDS за 3 минуты (nginx, ssl, compose)

## Основная информация

На просторах гитхаба и особенно в .ру сегменте нет цельного базового решения для деплоя python telegram бота(ов) на виртуалку под докером и на вебхуках и ssl сертификатом от Let's Encrypt. Решил исправить ситуацию и сделать универсальный образ, готовый к развертыванию на VPS. Если запустите этот образ, то его можно легко дополнять микросервисами, в том числе django, postgres, redis и тд.

Бот в комплекте основан на библиотеке aiogram - настройка его webhook'а описана тут https://docs.aiogram.dev/en/latest/examples/webhook_example.html. Это простой эхо-бот болванка, готовый для добавления вашей логики.
У aiogram уже есть собственный веб-сервер, но нам интересен микросервисный подход и подключение других сервисов в docker compose, поэтому используем nginx.

Описания @botfather и самого бота не будет. Если вы дошли до деплоя, то уже и так в курсе.
<br/><br/>
## Подготовка

Чтобы все заработало, нам понадобится самый простой VPS, привязанный к нему домен, ssh доступ и пара минут.
<br/><br/>
**Подготовка**

0. Выбираем понравившегося провайдера, заказываем простой виртуальный сервер на ubuntu (любой, 18-20 версий), 1 gb RAM и 10 gb SSD будет достаточно (хватит еще на пару микросервисов). Запоминаем его IP.
1. Скорее всего, провайдер предоставит бесплатный домен в своей подсети, но имейте в виду - Lets's Encrypt скорее всего не выпустит SSL сертификат, тк у него есть почасовая квота, которая по распространенным доменам расходуется моментально (например, хостинг timeweb и их бесплатный ***.tw1.ru)
2. Покупаем домен и привязываем его к IP нашего VPS. Не забываем проверить, что домен попал в общедоступные DNS-сервера (это все на автомате должен сделать доменный провайдер). Обычно это видно в личном кабинете провайдера в соответствующем разделе. А - запись для ipv4, AAAA - запись для ipv6. 

    2.1 Получили домен "your_domain.com"
    
    2.2 Проверим в терминале (macos/linux) привязку домена к ip:
    ```
    nslookup your_domain.com
    ```
    В ответ должны получить:
    ```
    Non-authoritative answer:
    Name:	your_domain.com
    Address: ip вашего VPS
    ```

3. Подготавливаем VPS:

    3.1. Заходим на VPS по ssh, устаналиваем docker (compose также установится в комплекте) по гайду: https://docs.docker.com/engine/install/ubuntu/. Самый простой вариант - добавить репозиторий и скачать последнюю версию. Если ошибок нет - отлично, двигаемся дальше.

    3.2. Клонируем этот репозиторий полностью:
    ```
    git clone https://github.com/ssharkexe/telegram-nginx-docker-webhook.git
    ```

    3.3. Переходим в папку с проектом и прописываем переменные окружения в .env файл:
    ```
    cd telegram-nginx-docker-webhook
    sudo nano .env
    ```
    указываем в кавычках токен вашего бота (TELEGRAM_TOKEN) и домен (NGINX_HOST), сохраняем.


4. Дальше необходимо запустить отдельно nginx на 80 порту без SSL, чтобы выпустить для него SSL сертификат, но возникает коллизия: в составе всего проекта nginx не запустится, тк будет искать SSL сертификаты в указанных нами папках, а их там до обращения к let's encrypt нет. Есть несколько разных решений, я использую самое очевидное и с минимальной правкой кода:

    4.1 Nginx с версии 1.19 поддерживает загрузку переменных окружения напрямую в свой конфигурационный файл, поэтому отдельно конфиг nginx править мы не будем (мы уже прописали в переменную окружения NGINX_HOST наш домен). Передача переменных происходит так: nginx при запуске забирает файл ***.conf.template (по сути - тот же nginx.conf, только с переменными окружения вида ${VARIABLE}), подменяет в нем все переменные на их значения и сохраняет этот файл в свою рабочую директорию nginx/conf.d/. Об этом указано на официальной странице nginx docker образа: https://hub.docker.com/_/nginx
    
    4.2 В docker-compose.yaml в разделе nginx есть следующее:
    ```
    volumes:
      - ./nginx/first_start/:/etc/nginx/templates/:ro
     # - ./nginx/templates/:/etc/nginx/templates/:ro
    ```
    В комплекте у нас есть 2 конфига: сокращенный, только на 80 порт и без ssl, и полный для постоянной работы, на 80, 443 портах и ссылками на сертификаты. По умолчанию при первом запуске nginx подхватит конфиг из папки ./nginx/first_start/, поэтому:
    ВАЖНО: после выпуска сертификата нужно удалить / закомментировать первую строку и раскомментировать вторую, где указана папка ./nginx/templates/

    4.3 Настройки выполнены, собираем образ и запускаем nginx
    ```
    docker compose --env-file .env build
    docker compose run --rm -d -p 80:80 nginx
    ```
    Важно, что при запуске отдельных сервисов compose не считывает мэппинг портов из docker-compose.yaml, поэтому в команде выше мы указали 80:80 вручную.

    4.4 Отлично, nginx запущен, можно тут же в консоли выполнить:
    ```
    curl http://your_domain.com
    ```
    Если появилась ошибка 301 - все отлично!  Идем дальше.
<br/><br/>
## Финальная часть. Получаем SSL сертификаты от Let's Encrypt

Nginx запущен, ждет обращения по адресу /.well-known/acme-challenge/. Пробуем смоделировать выпуск сертификата (не забываем заменить your_domain.com на ваш домен):
```
docker compose run --rm  certbot certonly --webroot --webroot-path /var/www/certbot/ --dry-run -d your_domain.com
```
Вводим почту и соглашаемся на выпуск. Если в консоли появилось:
```
The dry run was successful.
```
Все отлично, можно выпускать сертификат. Выполняем аналогичную команду, только без --dry-run:
```
docker compose run --rm  certbot certonly --webroot --webroot-path /var/www/certbot/ -d your_domain.com
```

Поздравляю, сертификат выпущен (на 3 месяца)! Осталось завершить контейнер с nginx:
```
docker compose kill
docker compose down
```
Скорректировать docker-compose.yaml в соответствии с п4.2:
```
volumes:
 # - ./nginx/first_start/:/etc/nginx/templates/:ro
  - ./nginx/templates/:/etc/nginx/templates/:ro
```
И запустить весь проект, предварительно его пересоздав:
```
docker-compose --env-file .env build
docker-compose up
```
В консоли появится "Telegram servers now send updates to https://your_domain.com. Bot is online". Вы великолепны!

Обновление сертификата через 3 месяца командой (если nginx уже запущен, первую команду пропускаем):
```
docker-compose up
docker-compose run --rm certbot renew
```
<br/><br/>
## Заметки
Если вылезут баги, прошу оформить Issue.
* В директории /bot есть Dockerfile, в котором прописан образ python 3.9 alpine и параметры запуска скрипта (+ бонус, установка timezone MSK)
* Бот можно запустить и в режиме polling для отладки, в main.py установить IS_WEBHOOK = 0
