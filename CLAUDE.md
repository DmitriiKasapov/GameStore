# О проекте

Это учебный проект, который собирается в процессе прохождения видеоуроков на YouTube. Проект на WordPress + WooComerce. Doker. Блоки будут Gutenberg + React. Темы и плагины вынесены в корневые папки.

## Правила общения и кода

- Общение всегда на русском языке (кириллица)
- Комментарии в коде только на английском

## Общие правила

- в файле PLAN.md я записываю, что делаем на каждом занятии. Начал с 3.

## Архитектура и развёртывание (универсальный шаблон WP + Docker)

### Стек
- WordPress + WooCommerce
- Docker (mysql + wordpress + phpmyadmin)
- Gutenberg-блоки на React (`@wordpress/scripts`)

### Структура проекта

```
project-root/
├── docker-compose.yml
├── CLAUDE.md              # контекст проекта для AI-ассистента
├── PLAN.md                 # лог занятий/прогресса
├── .srv/
│   ├── database/           # volume для MySQL (данные БД)
│   └── wordpress/          # volume для WP core (ядро, не редактируется руками)
├── themes/
│   └── <theme-name>/       # кастомная тема, монтируется в wp-content/themes
├── plugins/
│   ├── <core-plugin>/      # кастомный функциональный плагин (PHP)
│   └── <blocks-plugin>/    # Gutenberg-блоки (create-block, React)
└── mu-plugins/
    └── <name>.php          # must-use плагины (нельзя отключить из админки)
```

### Принцип монтирования (важно)
`.srv/wordpress` — это полная установка WordPress, но содержимое `wp-content/themes`, `wp-content/plugins`, `wp-content/mu-plugins` **перекрывается** bind-mount'ами из корневых папок `themes/`, `plugins/`, `mu-plugins/`. Значит:
- Любые темы/плагины кладутся в корень проекта, а не внутрь `.srv/wordpress/wp-content/...`
- Файлы внутри `.srv/wordpress/wp-content/plugins` и `themes` (кроме смонтированных) физически не используются — это мёртвые остатки исходной установки образа.

### docker-compose.yml (шаблон)

```yaml
services:
  mysql:
    image: mysql:8.0
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: wordpress
      MYSQL_DATABASE: <db_name>
      MYSQL_USER: wordpress
      MYSQL_PASSWORD: wordpress
    volumes:
      - ./.srv/database:/var/lib/mysql
    ports:
      - "3306:3306"

  wordpress:
    image: wordpress:latest
    restart: unless-stopped
    depends_on:
      - mysql
    environment:
      WORDPRESS_DB_HOST: mysql:3306
      WORDPRESS_DB_NAME: <db_name>
      WORDPRESS_DB_USER: wordpress
      WORDPRESS_DB_PASSWORD: wordpress
    volumes:
      - ./.srv/wordpress:/var/www/html
      - ./themes/:/var/www/html/wp-content/themes/
      - ./plugins/:/var/www/html/wp-content/plugins/
      - ./mu-plugins/:/var/www/html/wp-content/mu-plugins/
      - ./.srv/custom.ini:/usr/local/etc/php/conf.d/custom.ini
    ports:
      - "8000:80"

  phpmyadmin:
    image: phpmyadmin:latest
    restart: unless-stopped
    depends_on:
      - mysql
    environment:
      PMA_HOST: mysql
      MYSQL_ROOT_PASSWORD: wordpress
    ports:
      - "8181:80"
```

### Команды развёртывания

#### Запуск/остановка
```bash
docker compose up -d        # поднять все контейнеры в фоне
docker compose down         # остановить и удалить контейнеры (данные в .srv/ останутся)
docker compose logs -f wordpress   # смотреть логи WP-контейнера
```

#### Доступ
- Сайт: http://localhost:8000
- phpMyAdmin: http://localhost:8181

#### Создание нового Gutenberg-блока (плагин с блоками)
```bash
cd plugins
npx @wordpress/create-block@latest <plugin-slug>
cd <plugin-slug>
npm start      # дев-режим с автосборкой при изменениях
npm run build  # финальная сборка для продакшена
```

#### Создание простого PHP-плагина (без блоков)
Вручную: `plugins/<slug>/<slug>.php` с шапкой:
```php
<?php
/**
 * Plugin Name: <Name>
 * Description: <...>
 * Version: 1.0
 * Author: <...>
 * License: GPL2
 * Text Domain: <slug>
 */
```

#### mu-plugins (общий код, нельзя отключить)
Файл кладётся прямо в `mu-plugins/<name>.php` — подключается автоматически WordPress'ом без активации.

### Чек-лист для нового проекта по этому шаблону
1. Скопировать `docker-compose.yml`, заменить `<db_name>` и порты при конфликте.
2. Создать пустые папки `themes/`, `plugins/`, `mu-plugins/`, `.srv/database/`.
3. `docker compose up -d` — дождаться первого запуска WP (создаст `.srv/wordpress`).
4. Пройти установку WP через браузер (http://localhost:8000).
5. Положить тему/плагины в соответствующие корневые папки — подхватятся автоматически через volume.
