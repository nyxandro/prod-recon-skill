<div align="center">

# Prod Recon

**Скилл для Claude Code для проверки прода только на чтение. ИИ-агент сам находит сервер, читает логи и базу, отчитывается с доказательствами и ничего не трогает.**

[![Release](https://img.shields.io/github/v/release/nyxandro/prod-recon-skill)](https://github.com/nyxandro/prod-recon-skill/releases)
[![License](https://img.shields.io/github/license/nyxandro/prod-recon-skill)](LICENSE)
[![Agent Skills](https://img.shields.io/badge/Agent%20Skills-SKILL.md-6f42c1)](https://agentskills.io/specification)
[![Claude Code plugin](https://img.shields.io/badge/Claude%20Code-plugin-d97757)](#установка)

[English](README.md) · **Русский**

</div>

Prod Recon это скилл для разведки работающего сервиса только на чтение, будь то прод, стейдж или
локальный стек, в Claude Code, Codex, OpenCode, Cursor и других агентах, которые понимают Agent
Skills. Агент сам находит, как подключиться, подтверждает, какой коммит там реально работает,
читает логи и базу за нужный промежуток времени, выдаёт вывод с доказательствами и
останавливается, ничего не исправляя.

## Зачем

"Посмотри прод, только на чтение" владельцы просят чаще всего, и каждый раз всё ломается
одинаково.

- Агент спрашивает, какой сервер, хотя ответ лежит в репозитории.
- Заново пересказывает вам устройство системы.
- Находит причину и молча чинит её в том же ходе, ещё не рассказав, что было не так.

Скилл делает такую просьбу дешёвой, а результат предсказуемым.

## Установка

| Агент | Команда |
|---|---|
| Claude Code, как плагин | `/plugin install prod-recon --marketplace nyxandro/skills` |
| Claude Code, Codex, OpenCode, Cursor и другие | `npx skills add nyxandro/prod-recon-skill` |
| Любой агент, вручную | Скачайте [архив `.skill` последнего выпуска](https://github.com/nyxandro/prod-recon-skill/releases/latest/download/prod-recon.skill) и распакуйте в папку скиллов агента |

Выберите один способ. Если поставить двумя, Claude Code загрузит скилл дважды.

<details>
<summary>Другие способы установки</summary>

<br>

- Установка плагина одной командой работает в Claude Code 2.1.275 и новее. На старой версии
  сначала добавьте каталог командой `/plugin marketplace add nyxandro/skills`, потом выполните
  `/plugin install prod-recon@nyxandro`.
- Версия в каталоге не закреплена, поэтому каждый коммит сюда приходит пользователям плагина как
  обновление. Выполните `claude plugin marketplace update nyxandro` и
  `claude plugin update prod-recon@nyxandro` или включите автообновление в
  `/plugin` → Marketplaces.
- С `npx skills` добавьте `-g`, чтобы скилл стоял во всех проектах. Чтобы поставить определённую
  версию, допишите её метку, например `nyxandro/prod-recon-skill@v1.0.0`.
- Можно и клонировать репозиторий, а в папку скиллов агента положить ссылку на папку
  `prod-recon/`. Скилл это один файл `SKILL.md`, без скриптов и сборки.

</details>

## Как пользоваться

Часто хватает одного слова. На такие запросы скилл запускается.

- *Только чтение.*
- *Посмотри прод и доложи.*
- *Глянь логи на стейдже.*
- *Почему бот перестал отвечать?*
- *Были ли ошибки на прошлой неделе?*
- *Какая версия сейчас на проде?*

## Как работает

1. До первой команды агент в одном-двух предложениях говорит план. Например, "Проверю стейдж
   строго на чтение. Сначала инструкцию по деплою, потом контейнеры и логи, ничего не
   перезапуская. Цель связать сбой с сегодняшним релизом."
2. Находит, как подключиться, без вопросов. Документация проекта по деплою, память прошлых сессий,
   алиасы в `~/.ssh/config` и файлы деплоя в репозитории. Логин и пароль из вашего запроса
   используются через `sshpass`. Хост, который только похож на проект, не считается, а если вы не
   назвали окружение, агент спросит, а не пойдёт на прод по умолчанию.
3. Устанавливает, какой коммит работает на этом сервере и насколько он отстаёт от какой ветки.
   "Наверное, последний" ответом не считается.
4. Читает за промежуток времени из вашего вопроса. Контейнеры и их перезапуски, логи приложения,
   которые сначала считаются и только потом читаются, базу только через `SELECT`, сервисы, от
   которых зависит приложение, и публичный домен снаружи.
5. Делает только те выводы, которые позволяют доказательства. "Работает" означает сигнал, который
   увидел бы пользователь, а не здоровый контейнер и не зелёный CI.
6. Отчитывается и останавливается. Исправление это отдельная просьба.

## Договор "только чтение"

| Можно | Нельзя |
|---|---|
| `ssh`, и `sshpass`, если вы дали логин и пароль | Перезапускать или пересоздавать что угодно |
| `docker ps`, `docker inspect`, `docker logs`, `docker compose ps` | `docker compose up`, `down` или `restart`, любой `prune` |
| Запросы `SELECT` | `INSERT`, `UPDATE`, `DELETE`, `ALTER` |
| Чтение файлов, `df`, `free`, `uptime` | Правка любого файла, локального или на сервере |
| GET публичного домена снаружи | Отмена и перезапуск задач CI, вывод секретов |

Найденная причина не даёт права её исправить, даже если правка в одну строку.

## Пример

Отчёт о выдуманном сбое. Форма каждый раз одна и та же, чтобы отчёты разных дней можно было
сравнивать.

```text
Verdict. The bot has not replied since 14:05. The LLM gateway rejects every call.

Environment. Production, alias acme-prod. Runs 3f9c2a1 "Switch to the new model",
deployed today, 0 commits behind main. Window 12:00 to 16:00.

Facts
  14:02  The release marker shows 3f9c2a1 went live.
  14:05  First of 412 bot log lines "401 invalid api key". Every LLM call since then fails.
  15:58  GET of the health page from outside returns 200 in 0.3 s, so the web part is fine.

Not checked. The gateway's own dashboard, there is no access to it.

Proposal. Put the new key into the production env and restart the bot, on your command.
```

## Версии

Текущая версия записана в поле `metadata.version` файла `prod-recon/SKILL.md`, так что любая
установленная копия показывает, какая она. У каждой версии есть метка в git и
[выпуск](https://github.com/nyxandro/prod-recon-skill/releases) со своим архивом.
[CHANGELOG.md](CHANGELOG.md) рассказывает, что изменилось.

<details>
<summary>Устройство репозитория</summary>

<br>

```text
prod-recon/SKILL.md       сам скилл
dist/prod-recon.skill     папка скилла архивом для скачивания
CHANGELOG.md              что изменилось в каждой версии
README.md, README.ru.md   эта страница на английском и на русском
```

</details>

## Лицензия

[MIT](LICENSE)
