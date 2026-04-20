<h1 align="center">
  Скилл для Точка Банк API — Claude Code
</h1>

<p align="center">
  <strong>Claude Code скилл для работы с REST API Точка.Банк — счета на оплату, платёжки, СБП, выписки Open Banking, закрывающие документы, вебхуки</strong>
</p>

<p align="center">
  <a href="#установка">Установка</a> •
  <a href="#возможности">Возможности</a> •
  <a href="#использование">Использование</a> •
  <a href="#хук-подтверждения">Хук подтверждения</a> •
  <a href="#документация">Документация</a> •
  <a href="#лицензия">Лицензия</a>
</p>

<p align="center">
  <img src="https://img.shields.io/badge/Claude_Code-Skill-blueviolet?style=flat-square" alt="Claude Code Skill">
  <img src="https://img.shields.io/badge/Точка.Банк-API-red?style=flat-square" alt="Tochka Bank">
  <img src="https://img.shields.io/badge/Лицензия-MIT-green?style=flat-square" alt="MIT License">
  <img src="https://img.shields.io/badge/Python-stdlib_only-blue?style=flat-square" alt="stdlib-only">
</p>

---

## Обзор

Скилл расширяет [Claude Code](https://claude.ai/code) специализированными знаниями и Python-клиентом (только стандартная библиотека) для [REST API Точка.Банк](https://developers.tochka.com/docs/tochka-api/). Покрывает два потока аутентификации — личный JWT и OAuth 2.0 + Consent — и 19 CLI-команд для самых частых задач: выставление счетов, выписки, платёжные поручения, СБП QR-коды, вебхуки. Всё проверено на продакшне по состоянию на 2026-04-20.

## Возможности

- **Два потока аутентификации** — личный JWT (просто, для чтения + черновики платёжек) или OAuth 2.0 + Consent (полный Invoice API + Open Banking выписки)
- **Хранилище с шифрованием** — токены в macOS Keychain / Linux secret-tool / Windows Credential Manager, файловый fallback; access_token OAuth автоматически обновляется
- **Интерактивные визарды** — `init` и `init --oauth` с проверкой TTY, подсказками по регистрации приложения и локальным HTTPS-callback сервером (с поддержкой mkcert)
- **19 CLI-команд**, охватывающих:
  - **Счета и балансы** — `list-accounts`, `get-balance`
  - **Входящие платежи** — `list-incoming` (СБП + карты через эквайринг)
  - **Черновики платёжек** — `list-for-sign` («На подпись»)
  - **Полная выписка** — `list-statement` (асинхронный Open Banking flow)
  - **Счета на оплату** — `create-invoice`, `send-invoice` с автозагрузкой PDF
  - **Закрывающие документы** — акт / УПД / ТОРГ-12 / счёт-фактура через `create-closing-doc`
  - **Платёжные ссылки** (интернет-эквайринг) — `create-payment-link`
  - **Вебхуки** — `register-webhook` / `test-webhook` / `list-webhooks` / `delete-webhook`
  - **Реестр эквайринга** — суточная сверка через `list-registry`
  - **Диагностика OAuth consent** — `list-consents` / `get-consent`
- **Флаг `--format {json,id,url}`** у `create-*` команд — получить только ID или URL в stdout (envelope в stderr), без `jq`
- **Шпаргалка error → fix** (18 строк) — конкретные ошибки продакшна с причиной и исправлением
- **Опциональный хук безопасности** — запрос подтверждения в Claude Code с метками `[PROD]` / `[SANDBOX]` перед каждым изменяющим вызовом
- **Offline OpenAPI-спека** — `references/swagger.json` (OpenAPI 3.1.0, Tochka.API v1.90.4-stable) для поиска схем через `jq`

## Установка

### Вариант 1 — уровень проекта (только этот репозиторий)

```bash
mkdir -p .claude/skills
git clone https://github.com/rodion-m/tochka-bank-skill.git .claude/skills/tochka-bank-api
```

### Вариант 2 — уровень пользователя (все проекты)

```bash
mkdir -p ~/.claude/skills
git clone https://github.com/rodion-m/tochka-bank-skill.git ~/.claude/skills/tochka-bank-api
```

После клонирования перезапустите Claude Code, чтобы скилл был обнаружен.

### Первоначальная настройка

Скилл поставляется с интерактивными визардами. Выберите **один** вариант в зависимости от задачи:

```bash
# Просто: личный JWT из ЛК Точки. Подходит для чтения + черновиков платёжек.
# НЕ покрывает счета и выписки (возвращают 501 с личным JWT).
! python3 .claude/skills/tochka-bank-api/scripts/tochka_client.py init

# Полный: OAuth 2.0 + Consent. Нужен для счетов, закрывающих документов, выписок.
# Требует однократной регистрации приложения на https://i.tochka.com/bank/services/m/integration/new
! python3 .claude/skills/tochka-bank-api/scripts/tochka_client.py init --oauth
```

Префикс `!` **обязателен** — визард читает секреты через `getpass`, которому нужен настоящий TTY. Bash-инструмент Claude Code TTY не предоставляет; визард проверяет это при запуске и сразу выдаёт понятную подсказку.

Для уровня пользователя замените `.claude/skills/tochka-bank-api/` на `~/.claude/skills/tochka-bank-api/`.

## Использование

После установки Claude Code автоматически подключает скилл при задачах, связанных с Точка.Банком. Примеры:

```
> Выставь счёт через Точку на 50 000 рублей для ООО «Ромашка»
> Получи выписку из Точки за последнюю неделю
> Зарегистрируй вебхук на incomingPayment к https://receiver.example.com/tochka
> Какие платёжки ждут подписания в Точке?
> Проверь баланс счёта в Точке
> Создай закрывающий акт по счёту №7
```

Агент обращается к [SKILL.md](SKILL.md) за маппингом задача → команда, использует `--format` для pipeline-friendly вывода и при необходимости заглядывает в [references/endpoints.md](references/endpoints.md) или [ReDoc](https://enter.tochka.com/doc/v2/redoc).

## Хук подтверждения

Изменяющие вызовы к API (выставление счетов, отправка платёжных поручений, регистрация СБП QR-кодов, настройка вебхуков) затрагивают реальные деньги или постоянные записи в банке. В репозитории есть **опциональный, но настоятельно рекомендуемый** хук [`hooks/tochka-require-confirmation.sh`](hooks/tochka-require-confirmation.sh), который требует подтверждения перед каждым таким вызовом.

Установка:

```bash
# 1. Скопируйте хук-скрипт
mkdir -p .claude/hooks
cp .claude/skills/tochka-bank-api/hooks/tochka-require-confirmation.sh .claude/hooks/
chmod +x .claude/hooks/tochka-require-confirmation.sh
```

2. Зарегистрируйте в `.claude/settings.json` (добавьте блок `hooks` к существующей конфигурации):

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/tochka-require-confirmation.sh",
            "timeout": 10
          }
        ]
      }
    ]
  }
}
```

Хук показывает запрос подтверждения Claude Code с меткой `[PROD]` или `[SANDBOX]` и именем команды перед каждым изменяющим вызовом:

- `[PROD] create-invoice — реальные деньги / банковские записи могут быть затронуты`
- `[SANDBOX] create-invoice — только песочница, безопасно разрешить`

Read-only команды (`list-*`, `get-*`, `config`, валидация `init`) проходят без запросов.

## Документация

- [`SKILL.md`](SKILL.md) — роутер для агентов: Quickstart (с выбором по цели), матрица возможностей JWT vs OAuth, таблица задача → команда, критичные ошибки схемы, команды на каждый день, шпаргалка error → fix
- [`references/auth.md`](references/auth.md) — JWT vs OAuth в деталях, полный авторитетный список разрешений (19 значений), устройство визардов, подводные камни регистрации OAuth-приложения (в т.ч. баг `localhost` vs `127.0.0.1`)
- [`references/endpoints.md`](references/endpoints.md) — схемы тел запросов, правила валидации, справочник полей для 19+ эндпоинтов; различия полей для ИП vs ООО при выставлении счетов
- [`references/webhooks.md`](references/webhooks.md) — верификация подписи (через OIDC discovery), семантика повторных попыток, идемпотентность
- [`references/swagger.json`](references/swagger.json) — offline OpenAPI 3.1.0 спека (447 КБ) для поиска схем через `jq`. Источник: `https://enter.tochka.com/doc/openapi/swagger.json`

## Ограничения

- **Не для мультитенантного SaaS.** OAuth поддерживает мультитенантность, но визарды оптимизированы под автоматизацию «своего» аккаунта.
- **Только Точка.Банк.** У Тинькофф, Сбера, ВТБ другие схемы — не переносите примеры.
- **Отличие от стандартов АФТ/ОБР.** API Точки не соответствует утверждённым ЦБ стандартам АФТ (wiki.openbankingrussia.ru, Accounts v2.0/v3.0, ФАПИ Advanced v2.0). Ожидайте смену схем около **01.10.2026**.
- Некоторые пути и имена полей меняются без строгого semver — всегда сверяйтесь с живым [ReDoc](https://enter.tochka.com/doc/v2/redoc) перед деплоем.

## Вклад в проект

Приветствуются issues и PR. Особенно полезны:

- Новые строки в шпаргалку error → fix (точная строка ошибки + причина + исправление).
- Обновления схем при breaking changes Точки.
- Верификация СБП-операций вживую (в скилле нет live-проверенных СБП-вызовов — схемы из swagger + ReDoc).

## Лицензия

[MIT](LICENSE) — используйте, форкайте, адаптируйте свободно.
