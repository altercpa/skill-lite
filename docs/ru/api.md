# API-интерфейс

API фильтров и трекера позволяют работать с проверкой визитов, трафиком и статистикой. Чтобы получить API-токен, зайдите на страницу своего [профиля](/profile) в системе. Взаимодействие с сервисом осуществляется по протоколу HTTP. Успешный ответ имеет статус 200, а ошибка возвращается с соответствующим HTTP-статусом (например, 400, 401, 402, 403, 404 или 500) и JSON-телом с полями `status` и `error`. Стандартный формат запроса - GET или "чистый" POST (`form-urlencoded`). При отправке в JSON не забывайте заголовок `Content-type: application/json`. Формат результата - JSON. Ограничений на количество запросов нет.

Далее по тексту важные параметры указаны в фигурных скобках:

- `{token}` - ваш API-ключ из [личного кабинета](/profile), например `123-abdce`.
- `{key}` - ваш ключ постбека из [любого потока](/track-flows), например `666c93ba7cebe113`.
- `{flow}` - символьный идентификатор потока из списка, например `ef1c`.
- `{click}` - символьный идентификатор клика из трекера, например `ca1337`.

**Важно!** При использовании ссылок и данных фигурные скобки указывать не нужно, только сами значения!

> Функции трекера могут работать как для вебмастера, так и для команды. Для обычной работы, используйте URL вебмастера. Для работы на уровне команды, используйте URL команды с ключом любого из тимлидов.

Большая часть API может работать как в пространстве пользователя, так и в пространстве команды или администратора. По умолчанию, работа ведётся в пространстве пользователя. Для уточнения рабочего пространства, используйте поле `scope` с его указанием:

- `user` - стандартный запрос, работа от имени пользователя, доступно всем пользователям.
- `team` или `gang` - работа от имени лидера команды, доступна только тимлидам команд.
- `admin` - работа от имени администратора трекера, доступна только администраторам.

Некоторые функции имеют отдельные адреса для командной работы. В них использовать параметр `scope` не требуется. В документации указаны образцы ссылок для разного уровня доступа, если они поддерживаются.

## Фильтрация трафика

1. [Проверка визита](api-filter.md#check)
2. [Настройка кампании](api-filter.md#filterset)
3. [Список кампаний](api-filter.md#filterlist)
4. [Получение данных кампании](api-filter.md#filterget)
5. [Удаление кампании](api-filter.md#filterdel)
6. [Скачать файл фильтрации](api-filter.md#filtermake)
7. [Заблокировать IP или заголовок](api-filter.md#filterblock)
8. [Постбек мини-трекера](api-filter.md#filterpostback)
9. [Журнал визитов](api-filter.md#filterlog)
10. [Статистика визитов](api-filter.md#filterstats)

## Чёрные списки

1. [Доступные списки](api-blacklist.md#blacklistavail)
2. [Список чёрных списков](api-blacklist.md#blacklistlist)
3. [Записи списка](api-blacklist.md#blacklistshow)
4. [Создание списка](api-blacklist.md#blacklistadd)
5. [Настройка списка](api-blacklist.md#blacklistedit)
6. [Удаление списка](api-blacklist.md#blacklistdel)
7. [Добавление записи](api-blacklist.md#blacklistblock)
8. [Удаление записи](api-blacklist.md#blacklistremove)
9. [Изменение записи](api-blacklist.md#blacklistchange)
10. [Импорт записей](api-blacklist.md#blacklistimport)
11. [Экспорт записей](api-blacklist.md#blacklistexport)

## Интеграция с трекером

1. [Отправка постбека](api-tracker.md#postback)
2. [Импорт расходов](api-tracker.md#spend)
3. [Шаблоны постбеков](api-tracker.md#templates)
4. [Источники трафика](api-tracker.md#networks)

## Статистика трекера

1. [Список лидов](api-stats.md#leads)
2. [Данные лида](api-stats.md#lead)
3. [Статистика по кликам](api-stats.md#stats)
4. [Список кликов](api-stats.md#clicks)

## Click API — обработка переходов

1. [Регистрация клика по коду потока](api-click.md#tds)
2. [Регистрация клика по домену](api-click.md#domain)

## Хранилище сайтов

1. [Список сайтов](api-sites.md#siteslist)
2. [Загрузить сайт](api-sites.md#sitesadd)
3. [Изменить сайт](api-sites.md#sitesedit)
4. [Удалить сайт](api-sites.md#sitesdel)
5. [Скачать сайт](api-sites.md#sitesdownload)

## Паркованные домены

1. [Список доменов](api-domains.md#domainslist)
2. [Добавить домен](api-domains.md#domainsadd)
3. [Проверка домена](api-domains.md#domainscheck)
4. [Удалить домен](api-domains.md#domainsdel)
5. [Привязать домен к потоку](api-domains.md#domainsglue)
6. [Список файлов подтверждения](api-domains.md#domainsfiles)
7. [Создать файл подтверждения](api-domains.md#domainsconfirm)
8. [Изменить файл подтверждения](api-domains.md#domainschange)
9. [Удалить файл подтверждения](api-domains.md#domainsremove)
10. [DNS-записи для домена](api-domains.md#domainsdns)

## Потоки и пути

1. [Список потоков](api-flows.md#flowslist)
2. [Получение потока](api-flows.md#flowsget)
3. [Файл клиента TDS](api-flows.md#flowsfile)
4. [Текущие капы по путям](api-flows.md#flowscaps)
5. [Создание потока](api-flows.md#flowscreate)
6. [Клонирование потока](api-flows.md#flowsclone)
7. [Изменение настроек потока](api-flows.md#flowschange)
8. [Удаление потока](api-flows.md#flowsremove)
9. [Добавление пути к потоку](api-flows.md#flowsadd)
10. [Копирование пути](api-flows.md#flowscopy)
11. [Изменение настроек пути](api-flows.md#flowsedit)
12. [Удаление пути](api-flows.md#flowsdel)

## Коннекторы импорта расходов

1. [Типы коннекторов](api-adv.md#advtypes)
2. [Список меток для сопоставления расходов](api-adv.md#advtags)
3. [Список интеграций](api-adv.md#advlist)
4. [Форма редактирования интеграции](api-adv.md#advform)
5. [Создать интеграцию](api-adv.md#advadd)
6. [Изменить интеграцию](api-adv.md#advedit)
7. [Запустить проверку вне расписания](api-adv.md#advupdate)
8. [Удалить интеграцию](api-adv.md#advdel)
9. [Журнал запросов интеграции](api-adv.md#advlog)

## Журналы

1. [Журнал API-запросов](api-logs.md#logsrequests)
2. [Журнал исходящих постбеков](api-logs.md#logspostbacks)
