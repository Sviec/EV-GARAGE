# EV-GARAGE на одном хосте

## Решение

Приложение публикуется как обычный Node.js-процесс на **одном** сервере. На этом же сервере живут PostgreSQL и файлы фотографий.

Облако не используем: ни Cloudflare (D1, KV, OpenNext), ни Vercel Blob, ни Yandex Object Storage, ни другой S3. Отдельное хранилище и вынос базы понадобятся только если появится реальная нагрузка — это отдельное решение, не часть текущей публикации.

Порядок: сначала код и локальная проверка, затем установка на хост. Production-трафик не переключать, пока локальная проверка не пройдена.

## Что где хранится

| Данные | Где |
|---|---|
| Каталог, контакты, архив моделей, видео из админки, статусы предложений, метаданные фото | PostgreSQL (`DATABASE_URL`) |
| Байты фотографий | Каталог `PHOTOS_DIR` на диске хоста |
| Само приложение | `next build` / `next start`, Node.js 22+ |

В базе у фотографии хранятся ключ файла, slug модели, порядок, MIME-тип, размер и дата. Сами байты в PostgreSQL не пишутся.

Каталог для первой публикации берётся из `src/data/cars.ts` (`npm run db:seed`). Фотографий в репозитории нет: галереи пустые, пока их не загрузят через админку. Старый Cloudflare KV отсюда не копируется.

Ошибка базы или диска логируется и возвращается клиенту как ошибка. Пустой каталог из-за недоступной базы и фиктивный `{ ok: true }` не допускаются.

## Локальная проверка

```bash
docker compose up -d postgres
npm ci
npm run db:migrate
npm run db:seed
npm run build
npm start
```

Переменные — в `.env` (образец `.env.example`, секреты в git не коммитить):

```env
NODE_ENV=production
PORT=3000
DATABASE_URL=postgresql://ev:ev@127.0.0.1:5432/ev_garage
POSTGRES_PORT=5432
ADMIN_PASSWORD=change-me
PHOTOS_DIR=./data/photos
```

Если порт 5432 на машине уже занят, поставьте другой `POSTGRES_PORT` и тот же порт в `DATABASE_URL`.

Проверить: главную, страницу марки/модели/комплектации, сравнение, `GET /api/catalog` и `GET /api/brands`, вход в админку, правку контактов, архив, видео, статусы предложений, загрузку/показ/перестановку/удаление фото. Повторить с остановленной базой: API каталога должен ответить 500, а не пустым списком. Файл больше 4 МБ и тип не JPG/PNG/WEBP отклоняются без записи.

Команды: `npm run lint`, `npm run typecheck`, `npm run build`.

## Публикация на одном хосте

На сервере: Docker, Git, порты 80/443 наружу и 3000 только локально.

1. Склонировать репозиторий, создать `.env` (не из git):

```env
NODE_ENV=production
PORT=3000
POSTGRES_USER=ev
POSTGRES_PASSWORD=UNIQUE_DB_PASSWORD
POSTGRES_DB=ev_garage
DATABASE_URL=postgresql://ev:UNIQUE_DB_PASSWORD@postgres:5432/ev_garage
ADMIN_PASSWORD=UNIQUE_STRONG_PASSWORD
PHOTOS_DIR=/data/photos
```

`DATABASE_URL` для контейнера приложения смотрит на хост `postgres` внутри compose. Пароль в URL и в `POSTGRES_PASSWORD` один и тот же.

2. Запуск всего стека:

```bash
docker compose --profile full up -d --build
```

Postgres, миграции, сид каталога (только если база пустая) и Next.js поднимаются вместе. Каталог фото — volume, он переживает пересборку образа. Постоянной папки фото внутри образа нет.

3. HTTPS: reverse proxy на `127.0.0.1:3000`. Пример Caddy:

```
example.com {
  reverse_proxy 127.0.0.1:3000
}
```

4. Проверить на домене те же страницы и цикл фото. После этого переключать DNS. При сбое до переключения DNS деплой остановить. Исходный Cloudflare не удалять, пока новый хост не проверен.

Миграции применяются при старте контейнера. `db:seed` не затирает уже заполненный каталог и не трогает фото, контакты, архив и видео.

## Обновление с GitHub

Рабочая копия на сервере — `/opt/ev-garage`. Push в `main` запускает `.github/workflows/ci.yml`:

1. Job `check`: `npm ci`, lint, typecheck, build. Если проверка красная, сервер не трогается.
2. Job `deploy`: на машине GitHub собирает образ и публикует `ghcr.io/olegsh247-stack/ev-garage:<sha коммита>`.
3. Затем Actions заходит на сервер по SSH, делает `git fetch` и `git reset --hard` на этот коммит и запускает `scripts/deploy-host.sh <sha>`.
4. Скрипт скачивает этот образ и выполняет `docker compose --profile full up -d` без сборки на VPS. Контейнер сам применяет миграции. Скрипт ждёт ответ `/api/brands`.

Volumes Postgres и фото, а также `.env`, при обновлении не удаляются. Пока в GitHub Actions не заданы `SSH_HOST`, `SSH_USER`, `SSH_PRIVATE_KEY` и `SSH_KNOWN_HOSTS`, образ публикуется, а сервер не меняется.

Ручной повтор уже опубликованного коммита, от пользователя `deploy`:

```bash
cd /opt/ev-garage
git fetch origin
git reset --hard <sha>
sh scripts/deploy-host.sh <sha>
```

## Что сделать для публикации

Remote — GitHub, ветка выкладки — `main`. Секреты и `.env` не коммитить. Дальше сайт обновляется push-ем в `main`.

1. Закоммитить код и отправить его в `main`.
2. Взять VPS с Ubuntu 24.04, от 2 ГБ RAM. Открыть входящие порты 22, 80 и 443.
3. На сервере один раз. Каталог `/opt/ev-garage` должен быть клоном `https://github.com/olegsh247-stack/EV-GARAGE.git`, файл `.env` остаётся на месте:

```bash
sudo sh /opt/ev-garage/scripts/bootstrap-host.sh
```

Скрипт ставит Docker, пользователя `deploy` и ключ, которым GitHub Actions заходит на сервер.

4. В GitHub → Settings → Secrets and variables → Actions:

| Имя | Значение |
|---|---|
| `SSH_PRIVATE_KEY` | приватный ключ из `/root/ev-garage-ci` |
| `SSH_USER` | `deploy` |
| `SSH_HOST` | публичный IP сервера |
| `SSH_KNOWN_HOSTS` | вывод `ssh-keyscan -H IP` с вашего компьютера |

После сохранения ключа в GitHub файл `/root/ev-garage-ci` на сервере удалить.

5. На сервере отредактировать `/opt/ev-garage/.env`: свои `POSTGRES_PASSWORD`, тот же пароль в `DATABASE_URL` (хост в URL — `postgres`), `ADMIN_PASSWORD`. Пока домен не смотрит на сервер, оставить `SITE_ADDRESS=:80`.
6. Push в `main` (или повторный запуск workflow). Actions публикует образ, сервер его скачивает. Открыть сайт и проверить каталог и админку.
7. DNS: A-запись домена на этот IP. В `.env` поставить `SITE_ADDRESS=example.com` и выполнить `docker compose --profile full up -d`. Caddy сам получит сертификат. Проверить `https://домен`.
8. Дальше для нового кода достаточно push в `main`. Правка `.env` по-прежнему делается на сервере.

### Без Docker для самого приложения

Postgres можно оставить в Docker (`docker compose up -d postgres`), приложение запустить с хоста: `npm ci`, `npm run db:migrate`, `npm run db:seed`, `npm run build`, systemd-юнит `deploy/ev-garage.service`. `PHOTOS_DIR` в этом случае — обычный путь на диске, например `/var/lib/ev-garage/photos`, владелец — пользователь сервиса.

Логи процесса должны содержать ошибки PostgreSQL и диска. Строку `DATABASE_URL` и пароль админки в логи не писать.
