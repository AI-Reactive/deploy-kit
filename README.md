# deploy-kit — единая выкладка проектов AI-Reactive

Один механизм для всех сайтов на VPS. Проект отличается только файлом `deploy.conf` (5–10 строк).

## Как это работает
- push в `main` → GitHub Actions → self-hosted runner проекта на VPS → `/opt/deploy-kit/bin/dk-sync` → `/opt/deploy-kit/bin/dk-deploy`.
- Код: вместо `actions/checkout` у раннера постоянная копия `~gh-<проект>/src/<repo>`. `dk-sync` делает `git fetch --depth=1` ровно нужного коммита, `reset --hard`, `clean -ffdx` и проверяет, что копия чистая и на нужном SHA. При любой ошибке копия удаляется и клонируется заново. Авторизация — `GITHUB_TOKEN` джобы заголовком `http.extraheader` только для команды fetch, на диск токен не пишется. Экономит ~4–5 с на выкладку.
- Раскладка: `/var/www/<домен>/releases/<дата>-<коммит>`, `current` → симлинк на рабочий релиз, `shared/` — загрузки, секреты, базы (подключаются в релиз симлинками).
- Шаги: `git archive` в новую папку → хук build (если есть) → `php -l` всех .php → симлинки shared → атомарное переключение `current` (`ln -sfn` + `mv -T`) → проверка главной (200, `curl --resolve` на 127.0.0.1) → иначе автооткат → хук after_switch → хранится 5 релизов.
- opcache сбрасывать не нужно: nginx отдаёт PHP `$realpath_root`, у каждого релиза свои пути.
- Выкладки идут строго по очереди (`concurrency`).
- Раннер работает от отдельного пользователя `gh-<проект>` (без пароля, без shell, без sudo), пишет только в `/var/www/<домен>/releases` и `current`; секреты в `shared/` ему не читаемы.
- Dev-режим (только cod1): тот же workflow с другим `deploy.conf`/лейблом на push в ветки.

## Откат
GitHub → Actions → Deploy → Run workflow → action = rollback (пусто = предыдущий релиз).
На сервере: `sudo -u gh-<проект> /opt/deploy-kit/bin/dk-rollback /path/deploy.conf [релиз]`.

## Новый проект (~10 минут)
1. В репозитории проекта: скопировать `templates/deploy.conf` и `templates/project-workflow.yml` (→ `.github/workflows/deploy.yml`, заменить PROJECT на лейбл).
2. Settings → Actions → Runners → New self-hosted runner → скопировать токен.
3. На сервере от root: `/opt/deploy-kit/bin/dk-bootstrap <домен> AI-Reactive/<repo> <токен>`.
4. Положить секреты/данные в `/var/www/<домен>/shared/`, nginx: `root /var/www/<домен>/current;`, `fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;` и ПОСЛЕ `include fastcgi_params;` — `fastcgi_param DOCUMENT_ROOT $realpath_root;` (иначе realpath-кеш PHP смешивает старый и новый релиз → «Cannot redeclare»), `nginx -t && systemctl reload nginx`.
5. push в main.

## Обновление самого kit на сервере
Скрипты лежат в `/opt/deploy-kit/bin` (владелец root, раннеры не могут их менять). После изменения в репо: скопировать `bin/` в `/opt/deploy-kit/bin` от root.
