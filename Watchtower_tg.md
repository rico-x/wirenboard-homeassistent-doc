# Watchtower отсылка уведомлений в телеграм

Получаем уведомления telegram от Watchtower при обновлении контейнера. Эта версия в основном ориентирована на docker-compose, но должна работать и с docker run.

## Installation

### 1. Telegram Bot

Для начала нам необходимо создать нового telegram-бота. Для этого откройте новый чат с [BotFather](https://telegram.me/botfather), введите команду `/newbot` и следуйте инструкции, в результате получим сообщение вида:

![success](https://i.imgur.com/ugOzB1B.png)

### 2. Получаем Chat ID

После создания бота необходимо получить идентификатор чата. Для этого откройте новый чат с ботом, запустите его и поставьте в чате метку `@get_id_bot`. Вы должны получить сообщение следующего содержания:
![chat id](https://i.ibb.co/0GY3cFp/Bild-2023-09-03-230007186.png)

### 3. Настройки Watchtower

Теперь необходимо настроить Watchtower. Для этого необходимо добавить в файл docker-compose следующие переменные окружения:

```yaml
environment:
    - WATCHTOWER_NOTIFICATIONS=shoutrrr
    - WATCHTOWER_NOTIFICATION_URL=telegram://HTTP_API_TOKEN@telegram/?channels=CHAT_ID
```

Замените `HTTP_API_TOKEN` на токен, полученный от BotFather в шаге 1. и `CHAT_ID` на идентификатор чата, полученный от `@get_id_bot` в шаге 2.

### 4. (Re)start Watchtower

Вам необходимо (пере)запустить Watchtower:

```bash
sudo docker-compose up -d --force-recreate
```

и вы должны получить сообщение от вашего бота, что Watchtower запущен. Сообщение должно выглядеть следующим образом:

![watchtower running](https://i.ibb.co/wB2T6vK/Bild-2023-09-03-230440535.png)

При обновлении контейнера вы также получите сообщение. Оно должно выглядеть следующим образом:

![updated](https://i.ibb.co/QkD8ywr/Bild-2023-09-03-231219112.png)
