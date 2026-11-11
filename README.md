# Локальная инфраструктура HH Clicker

Инструкция для coding agents: [AGENTS.md](AGENTS.md).

Каталог содержит общую инфраструктуру проекта. Dockerfile каждого приложения остаётся в его собственном репозитории.
Готовые пакеты для Windows, Linux и macOS устанавливаются через лаунчер — см.
[руководство пользователя](https://github.com/hh-clicker/.github/blob/master/profile/README.md).
Эта инструкция описывает ручной запуск из checkout на Windows и команды в PowerShell 7;
для Linux см. [отдельную инструкцию](https://github.com/hh-clicker/launcher/blob/master/LINUX.md#запуск-без-лаунчера-из-исходников).
Browser Agent запускается на desktop-хосте,
backend/frontend/PostgreSQL — через этот Compose, generation-agent — через отдельный Compose.

## Зависимости Windows и установка

### WSL 2 и Docker Desktop

Нужна поддерживаемая Docker Desktop 64-битная Windows с аппаратной виртуализацией, включённой
в BIOS/UEFI. Для нового окружения используйте Windows 11. Требования Docker: WSL 2.1.5 или новее,
не менее 8 ГБ RAM; точные поддерживаемые версии Windows — в [официальной инструкции Docker](https://docs.docker.com/desktop/setup/install/windows-install/).

Если WSL ещё не установлен, откройте PowerShell **от имени администратора**:

```powershell
wsl --install
```

Перезагрузите компьютер и завершите первичную настройку установленного дистрибутива. Затем обновите WSL:

```powershell
wsl --update
wsl --version
wsl --list --verbose
```

Для уже установленного WSL повторять установку не нужно. Подробности — [установка WSL от Microsoft](https://learn.microsoft.com/en-us/windows/wsl/install).

Скачайте установщик [Docker Desktop для Windows](https://docs.docker.com/desktop/setup/install/windows-install/),
установите и запустите приложение. Используйте WSL 2 backend и **Linux containers**, не Windows containers.
Для команд ниже Docker Desktop должен оставаться запущенным. Docker Compose входит в Docker Desktop;
отдельно устанавливать Docker Engine или Compose в Ubuntu не требуется.

### PowerShell, Node.js, Chrome и Git

- **PowerShell 7** — оболочка для приведённых команд, в том числе передачи кавычек в параметры Codex.
  Установка по [инструкции Microsoft](https://learn.microsoft.com/en-us/powershell/scripting/install/installing-powershell-on-windows):

  ```powershell
  winget install --id Microsoft.PowerShell --source winget
  ```

  После установки откройте **PowerShell 7** из меню «Пуск» или командой `pwsh`. Старый Windows PowerShell 5.1 остаётся отдельным приложением.
- **Node.js 24.x, не ниже 24.12, вместе с npm** — нужен локальному Browser Agent.
  На [странице загрузки Node.js](https://nodejs.org/en/download) выберите ветку 24.x, Windows и установщик для своей архитектуры.
  Запустите установщик, оставьте компоненты npm и добавление в PATH. Node нужен именно в Windows,
  а не только внутри WSL. Совместимость с 22.x начинается с 22.18; для общей инструкции используем 24.x.
- **Google Chrome** — установите обычную стабильную версию по [инструкции Google](https://support.google.com/chrome/answer/95346?hl=ru).
  После новой установки один раз откройте Chrome вручную и завершите первоначальную настройку.
  Агент использует системный Chrome с отдельным профилем. Скачивать Chromium через `playwright install` для обычного запуска не нужно.
- **Git for Windows** — для получения и обновления репозиториев. Установщик доступен на [сайте Git](https://git-scm.com/install/windows), либо:

  ```powershell
  winget install --id Git.Git -e --source winget
  ```

После установки программ откройте новое окно PowerShell 7 и проверьте:

```powershell
$PSVersionTable.PSVersion
git --version
node --version
npm.cmd --version
docker version
docker compose version
```

`docker version` должен показать и Client, и Server. Если Server недоступен, сначала дождитесь
запуска Docker Desktop. В командах ниже используется `npm.cmd`, чтобы не менять Execution Policy ради `npm.ps1`.

**Не требуется устанавливать на Windows для контейнерного запуска:** JDK/Gradle, PostgreSQL,
nginx и Codex CLI. Их версии задают Dockerfile/Compose: Java 25, PostgreSQL 17, Node 24 и
Codex CLI 0.163.0. Postman необязателен — нужен только для отдельной ручной проверки API генератора.
Для AI потребуется аккаунт с доступом к Codex; без AI кликер работает с обычными письмами.

## Первый запуск

Расположите каталоги рядом: `backend`, `frontend`, `browser-agent`, `generation-agent`, `infra`.
Compose использует относительные пути `../backend` и `../frontend`. Для первого скачивания зависимостей,
образов и работы с HH/Codex нужен интернет. Репозитории компонентов перечислены в [организации hh-clicker](https://github.com/hh-clicker).

Из каталога `infra` создайте локальные настройки, не перезаписывая существующие:

```powershell
if (!(Test-Path .env)) { Copy-Item .env.example .env }
notepad .env
```

Замените `POSTGRES_PASSWORD=change-me` своим локальным паролем и сохраните файл. `.env` с паролем
не нужно коммитить или передавать вместе с архивом проекта. Затем:

```powershell
docker compose up -d --build
docker compose ps
```

Backend автоматически применяет миграции Liquibase; отдельно создавать таблицы не нужно.
Для AI-оценки вакансий на Windows backend обращается к Generation Agent через
`host.docker.internal`; выбранный лаунчером `GENERATION_PORT` передаётся в `infra.env` автоматически.
На Linux лаунчер добавляет Compose override: оба проекта используют сеть инфраструктуры,
а backend обращается к Generation Agent по имени сервиса. Порт Generation Agent на хосте
по-прежнему привязан только к `127.0.0.1`.
После запуска адреса по умолчанию из `.env.example`:

- frontend: [http://localhost](http://localhost);
- backend: [http://localhost:8080](http://localhost:8080);
- OpenAPI: [http://localhost:8080/v3/api-docs](http://localhost:8080/v3/api-docs);
- PostgreSQL не публикуется на Windows: он доступен Backend внутри Docker network как `postgres:5432`.

Frontend обращается к backend по относительному пути `/api`; nginx проксирует его на контейнер `backend`.
Если поменяли `FRONTEND_PORT`, используйте его в адресе; при значении `80` достаточно `http://localhost`.

### Generation Agent — для оценки, анализа и AI-писем

В отдельном окне PowerShell 7 перейдите из `infra` в соседний каталог:

```powershell
Set-Location ../generation-agent
if (!(Test-Path .env)) { Copy-Item .env.example .env }
docker compose build
docker compose run --rm generation-agent codex -c 'cli_auth_credentials_store="file"' login --device-auth
docker compose up -d
Invoke-RestMethod http://127.0.0.1:17374/health
Invoke-RestMethod http://127.0.0.1:17374/auth/status
```

Выполните вход по ссылке и одноразовому коду из терминала. Device code login должен быть разрешён
в настройках аккаунта/рабочего пространства: [официальная инструкция Codex](https://learn.chatgpt.com/docs/auth).
Авторизация хранится в Docker volume, копировать Windows-файл `auth.json` в проект не нужно.
При уже сохранённом входе повторять login на каждом запуске не требуется.

По умолчанию используются `gpt-6-luna` и `low`; изменить их можно в `generation-agent/.env`.
Эти GET-проверки не запускают генерацию. `/health` не проверяет аккаунт, а `/auth/status`
возвращает сохранённый снимок действующей авторизации и лимитов без нового обращения к Codex.
Полный контракт API, повторный вход и Postman описаны в [generation-agent/README.md](../generation-agent/README.md).

### Browser Agent и вход в HH

Откройте новое окно PowerShell 7 в каталоге `browser-agent` на Windows:

```powershell
npm.cmd ci
npm.cmd start
```

Оставьте это окно открытым. При изменении `BACKEND_PORT` или `GENERATION_PORT` задайте соответствующие
адреса **перед** `npm.cmd start` (значения ниже — стандартные):

```powershell
$env:HH_CLICKER_BACKEND_URL = 'http://localhost:8080'
$env:HH_CLICKER_GENERATION_URL = 'http://127.0.0.1:17374'
```

В интерфейсе создайте HH-аккаунт, откройте управляемый Chrome, войдите в HH вручную и выберите
резюме для импорта. AI включается отдельно в настройках резюме/запуска. Запуск инфраструктуры
сам по себе не отправляет отклики; отправку разрешает кнопка запуска очереди в интерфейсе.

## Остановка

Сначала завершите или приостановите очередь в интерфейсе. Browser Agent останавливается через `Ctrl+C`
в его терминале и закрывает управляемый Chrome. Остановка уже отправленного запроса не отменяет отклик на HH.

Из каталога `infra`:

```powershell
docker compose down
```

Если запускали генератор, отдельно из `generation-agent` выполните `docker compose down`.
Для следующего запуска используйте `docker compose up -d` в обоих нужных каталогах и `npm.cmd start` у Browser Agent.
После изменения кода контейнеров добавляйте `--build`; после изменения зависимостей Browser Agent повторяйте `npm.cmd ci`.

Данные PostgreSQL сохраняются в volume `hh-clicker_postgres-data`, авторизация Codex —
в `hh-clicker-generation_codex-home`, задачи генератора — в `hh-clicker-generation_generation-data`.
Профили HH находятся в `%LOCALAPPDATA%\HH Clicker\data\browser-agent\profiles` на Windows.
Для обычного перезапуска удалять эти данные не нужно. **Не добавляйте `-v` к `docker compose down`: это удалит volumes соответствующего проекта.**
