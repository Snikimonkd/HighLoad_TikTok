# РПЗ к курсовой работе

Лукаш Сергей

## 1. Тема и целевая аудитория

<B>ТикТок</B> — сервис для создания и просмотра коротких видео, принадлежащий пекинской компании «ByteDance».

### MVP

1. Регистрация/авторизация
2. Публикация видео
3. Просмотр ленты
4. Лайки/комментарии

### Целевая аудитория

По состоянию на март 2021 года **MAU** = 36,6 млн. Распределение по возрасту: 43% пользователей от 25 до 44 лет, из них 57% женщины и 43% мужчины. ([Источник](https://t.me/odigital/1329/))

## 2. Расчет нагрузки

**DAU** можно с уверенностью считать 50%([Источник](http://appbrain.ru/osnovnyie-metriki-effektivnosti-mobilnoy-reklamyi/)) от **MAU**. Таким образом **DAU** = 36,6 * 0.5 = 18,3 млн. человек.

Среднее время пользования приложением за день(информация за 2019 год) = 38 мин. ([Источник](https://tiktok-wiki.com/statistika-tik-tok.html))

Для упрощения расчетов и с учетом устаревания статистики примем среднее время пользования приложением = 40 мин.

Для получения данных о количестве запросов, я потратил 10 минут в приложении и собрал статистику о работе приложения(.har файл).

За 10 минут было отправлено **1266** запросов(82,3 МБ): 33 за видео(67,2 МБ), 827 за статиткой(11,3 МБ), 406 за бизнес-логикой(3,8 МБ).

Таким образом среднее количество запросов в секунду равно:

```
1266 / (10 * 60) * 18 300 000 / (24 * 60 * 60) = 446 RPS
```

На одного пользователя потребляется (82,3 * 8) / (10 * 60) = **1,1** Мбит/с 

Нагрузка пользователя в секунду | Сеть 
---                             | ---
Видео                           | (67,2 * 8) / (10 * 60) = **0,9** Мбит / с 
Статика                         | (11,3 * 8) / (10 * 60) = **0,15** Мбит / с 
Бизнес-логика                   | (3,8 * 8) / (10 * 60) = **0,05** Мбит / с

Нагрузка пользователя в день    | Сеть 
---                             | ---
Видео                           | 0,9 * 40 * 60 = **2160** Мбит
Статика                         | 0,15 * 40 * 60 = **360** Мбит
Бизнес-логика                   | 0,05 * 40 * 60 = **120** Мбит

Нагрузка дневной аудитории      | Сеть 
---                             | ---
Видео                           | (2160 * 18 300 000) / (24 * 60 * 60) = **446** Гбит
Статика                         | (360 * 18 300 000) / (24 * 60 * 60) = **74** Гбит
Бизнес-логика                   | (120 * 18 300 000) / (24 * 60 * 60) = **24** Гбит

### Расчет загружаемых видео

Средний размер видео = 67,2 МБ / 33 = 2,03 МБайт. За месяц пользователи загружают 20 млн видео([Источник](https://news.cpa.ru/tiktok-showed-audience-statistics/)). Таким образом за один месяц пользователи загрузят видео на 2,03 МБ * 20 млн = 38,7 ТБайт.

## 3. Логическая схема бд

![логическая схема бд](db_scheme.PNG)

## 4. Физическая схема

В качестве СУБД будем использовать PostgreSQL из-за ее производительности, распространенности, хорошой работы под высокими нагрузками. Основная нагрузка будет приходиться не на базу данных, а на каналы связи для доставки видео, поэтому смысла шардировать базу нет. Для устойчивости базы следует сделать реплики(2 штуки), они будут использоваться только для чтения.

### Расчет размера БД:

Размер uuid = 16 Байт

Размер varchar(n) = n Байт

Пароли будем хранить в зашифрованном виде, и исопльзовать рекомендации [OWASP](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html)(sha3 + bcrypt).

Размер password = 60 Байт

Размер users
```
user_id + email + username + password + avatar_path = 16 + 50 + 50 + 60 + 100 = 276 Байт
276 Байт * 36,6 млн ~ 10 ГБ
```

Размер videos
```
video_id + user_id + video_path + description = 16 + 16 + 100 + 250 = 382 Байта
382 Байт * 20 млн * 12 ~ 85,32 ГБ
```

Размер likes([кол-во лайков, поставленных пользователями за месяц](https://news.cpa.ru/tiktok-showed-audience-statistics/))
```
video_id + user_id = 16 + 16 = 32 Байт
32 Байт * 1,62 млрд * 12 = 576 ГБайт (за год)
```

Размер subscribers(предположим что каждый пользователь подписан минимум на 10 авторов)
```
author_id + subscriber_id = 16 + 16 = 32 Байт
32 Байт * 10 * 36,6 млн ~  10,9 ГБайт (за все время)
```

Размер video_feed(ленту будем генерировать на 33 * 4 ~ 150 видео для каждого пользователя)
```
video_id + user_id = 16 + 16 = 32 Байт
32 Байт * 150 * 18,3 млн = 81 ГБайт
```

Размер comments(предположим, что человек комментирует видео раз в день)
```
comment_id + video_id + text + user_id = 16 + 16 + 250 + 16 = 298 Байт
298 Байт * 365 * 18,3 млн ~ 1853 ГБайт = 1,81 ТБайт 
```

Итоговый размер бд за год
```
10 ГБайт + 85,32 ГБайт + 576 ГБайт + 10,9 ГБайт + 81 ГБайт + 1853 ГБайт = 2616 ГБайт ~ 2,55 ТБайт
```

Так как основная нагрузка идет на видео контент, следует спроектировать CDN. Контент будем разделяться на "актуальный" и "неактуальный". Лента формируется "пачками" по 150 видео, на основе работы рекомендательной системы. На основе работы этой системы можно узнать "актуальный" контент и зугрузить его на сервера.

Все видео будут загружаться в хранилища, не входящие в CDN. После работы рекомендательной системы, контент будет догружаться на CDN.

Распределение "актуального" контента по CDN, будет производиться во время минимальной активности пользователей(3-6 утра по местному времени). Роутинг внутри CDN будем осущствлять с помощью GEO BASED DNS(это способствует уменьшению задержки из-за удаленности серверов от пользователя, а так же логически совпадает с тем, что в разных регионах людям будет рекомендоваться разный контент).

Расчёт конфигурации CDN сервера:

При пиковом онлайне в 6 000 000 пользователей, пропускная способность должна равняться:

```
1.1 Мбит/с * 6 000 000 = 6.6 Тбит/c
```

При использовании 2-х сетевых карт с пропускной способностью 25 Гбит/c, то потребуется:

```
6758.4 / (25 * 2) = 136 серверов
```

В месяц загружается примерно 38,7 ТБайт видео => в день загружается 1,29 ТБайт. Будем считать, что в "актуальное" попадает лишь 15% загруженного за день контента, а хранить его надо неделю:

```
0,15 * 1,29 ТБайт = 198 ГБайт ("актуального" контента в день)
```

Актуальный контент должен храниться неделю:

```
198 ГБайт * 7 = 1387 ГБайт = 1,35 ТБайт
```

Следовательно на CDN серверах должны стоять SSD объемом 2 ТБ.

Для хранения "неактуального" контента потребуется 38,7 ТБайт в месяц.

В год:

```
38,7 ТБайт * 12 = 464.4 ТБайт
```

## 5. Технологии

Для хранения данных будем исползовать СУБД PostgreSQL, обговаривалось выше.

Для бэкэнда будем использовать Go, из-за его скорости работы под большими нагрузками, скорости разработки на нем и т.д.

Для формирования ленты будем использовать Python, т.к. потребуется использование ML инструментов.

Для распределения "актуального" контента будем использовать Python, т.к. лента формируется на Python и распределение будет удобно делать сразу после формирования этого контента.

Для раздачи статики и балансировки будем использовать Nginx, из-за его распространенности.

## 6. Схема проекта

## 7. Список серверов

Сервер                      |CPU    |RAM    |Тип диска  |Объем диска    |Кол-во серверов
---                         |---    |---    |---        |---            |---    
Nginx                       |16     |16     |           |               |50
Backend                     |16     |32     |           |               |4
PostgreSQL                  |16     |32     |SSD        |4 ТБайт        |3(1 master + 2 slaves)
CDN(актуальное хранилище)   |16     |32     |SSD        |2 ТБайт        |150
Неактуальное хранилище      |16     |16     |HDD        |64 ТБайт       |1