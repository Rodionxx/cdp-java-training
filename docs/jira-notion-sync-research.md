# Синхронизация Jira → Notion с выборочными проектами: исследование вариантов

*Подготовлено 24.09.2026 по официальной документации Notion и Atlassian и материалам треда `#helpdesk-inbox-eng`. Все цитаты — из первоисточников, ссылки в тексте и в разделе 9. Утверждения, которые не удалось подтвердить документацией, помечены как требующие проверки.*

## 0. Резюме

- **Претензия IT «Notion работает только под Jira-админом» верна наполовину.** Нативный Jira Sync действительно требует API-токен аккаунта с правом `Administer Jira` (он нужен Notion, чтобы автоматически создать webhooks в Jira). Но Notion официально поддерживает **сервисный аккаунт** («You can create a Jira service account to set up Jira Sync — just make sure the service account has access to any projects you want to sync»), а в Jira право `Administer Jira` **не даёт** доступа к задачам — его даёт только `Browse Projects` в permission scheme. Значит, потолок того, что вообще может утечь в Notion, задаётся правами сервисного аккаунта и может быть равен ровно `PRO`.
- **Претензия «синкает всё целиком» неверна:** проекты и свойства выбираются явно («Select projects to sync → Select properties to sync»). Верно другое: фильтра по типам задач/JQL в нативном Sync нет, и данные тянет админский токен, а личная авторизация участника лишь определяет, какие проекты он может добавить.
- **Претензия про наследование прав Notion верна** и относится к любому копированию данных в Notion, включая уже разрешённый Connect (превью видны всем, у кого есть доступ к странице). Лечится закрытым тимспейсом, а не выбором инструмента.
- **На стороне Atlassian нет органа, который ограничил бы Notion одним проектом:** app access rules действуют только на установленные Connect/Forge-приложения, а Jira Sync ходит обычным пользовательским токеном. Граница — только права аккаунта либо push, который контролируете вы.
- **Реалистичных способов получить «одни проекты льются, другие нет» — несколько (варианты A–G в разделе 4).** Лучшие для этой задачи: (1) новый **Notion Workers** — хостируемый Notion код, который сам забирает задачи по JQL read-only токеном без админа и создаёт настоящую synced database; (2) **нативный Jira Sync с сервисным аккаунтом, ограниченным `PRO`**, под компенсирующими контролями; (3) **Jira Automation → Notion API** без кода и без единого токена Jira вне Atlassian. Дальше — Getint (вендор с JQL), n8n self-hosted, собственный сервис/Forge.
- **Фильтр «эпики и сторис без тасков»** штатными правами Jira невозможен (agoriachev прав), но делается одной строкой JQL/условия в вариантах Workers, Automation, Getint, n8n, Forge.
- Рекомендация: **пилот Notion Workers** параллельно с проверкой нативного Sync на песочнице по чек-листу из раздела 4.1.5; Jira Automation — запасной вариант без разработчика. Подробности — в разделах 5–8.

## 1. Постановка задачи

Из треда в `#helpdesk-inbox-eng`:

- **Цель (banar):** живая синхронизация задач Jira в Notion, чтобы вести работу поверх Jira-данных в Notion (виды, связи, rollup, Notion AI). В идеале — только проект `PRO`, и, возможно, только определённые типы задач (эпики/сторис, без тасков).
- **Уже разрешено:** «Connect» (превью Jira-ссылок в Notion, через личный токен пользователя).
- **Позиция безопасности/IT (agoriachev, p.ivanov):** нативный Jira Sync в Notion
  1. работает только из-под Jira-админа «с крайне широкими доступами», сервисный аккаунт с ограниченным скоупом не поддерживается;
  2. не даёт выбирать, что синхронизировать, «пытается засинкать всё целиком»;
  3. синхронизированные базы наследуют права Notion (Notion сам об этом предупреждает);
  4. кто-то по неосторожности может добавить в Notion приватный проект и «посветить» его всей компании.
- **Компромисс, который уже прозвучал (agoriachev):** ограничить сервисный аккаунт проектом `PRO` и read-only доступом, подтвердить, что в `PRO` нет чувствительных данных, ограничить доступ к synced database в Notion — тогда риск «не критически выше», чем у уже разрешённого Connect. Фильтрация по типам задач штатными permission scheme невозможна.

Ниже — проверка каждого утверждения по первоисточникам (документация Notion и Atlassian по состоянию на сентябрь 2026) и разбор всех реалистичных способов построить синхронизацию «одни проекты льются, другие — нет».

## 2. Проверка утверждений IT по документации

### 2.1. «Работает только из-под Jira-админа, сервисный аккаунт нельзя» — верно наполовину

Что говорит Notion (справка [Connect Jira to Notion](https://www.notion.com/help/jira)):

> The Jira Sync connection connects your Jira projects to Notion using an Admin API token (rather than a user token) … To ensure reliable syncing, we recommend creating a scopeless token by selecting Create API token, not Create API token with scopes.

> A workspace owner must set up Jira Sync first using a Jira Admin token. After that, any member can utilize the connection.

> **You can create a Jira service account to set up Jira Sync — just make sure the service account has access to any projects you want to sync into Notion beforehand.**

То есть:

- Схема гибридная: участники авторизуются в Jira через OAuth-приложение Notion (оно видно в admin.atlassian.com → Connected apps, где админ может включить «Block user apps»), а сама синхронизация идёт по API-токену выбранного аккаунта.
- Да, для первичной настройки нужен **API-токен аккаунта с правом `Administer Jira`** (глобальное право «Jira admin»). Причина техническая: Notion автоматически регистрирует в Jira **webhooks** («The webhook setup is done automatically when the sync is first established»), а создавать admin-webhooks в Jira Cloud может только пользователь с `Administer Jira` (справка Atlassian [Manage webhooks](https://support.atlassian.com/jira-cloud-administration/docs/manage-webhooks/): «Log in as a user with the Administer Jira global permission»). Это не OAuth-приложение и не Marketplace-app, а обычный basic-auth токен пользователя.
- Но **сервисный аккаунт официально поддерживается**, и Notion прямо говорит, что синхронизироваться может только то, к чему у этого аккаунта есть доступ.
- Ключевой нюанс модели прав Jira: **`Administer Jira` ≠ доступ ко всем задачам.** Право видеть задачи проекта (`Browse Projects`) выдаётся только через permission scheme проекта, глобальное админское право его не даёт (Atlassian, [JIRA permissions general overview](https://support.atlassian.com/jira/kb/jira-permissions-general-overview/), обсуждение [«You don't have the Browse Projects permission»](https://community.atlassian.com/forums/Jira-Cloud-Admins-discussions/You-don-t-have-the-Browse-Projects-permission-which-allows-you/td-p/611981)). Значит, «Jira-админский» сервисный аккаунт может видеть **только `PRO`**, если так настроены permission schemes.
- Остаточный риск, который IT формулирует верно: токен с `Administer Jira` — это право **менять конфигурацию** Jira (схемы прав, workflow, webhooks). Владелец токена (Notion) технически мог бы расширить себе доступ. Это управляемый риск (см. раздел 4.1: аудит изменений схем, ротация токена, невозможность интерактивного логина для сервисных аккаунтов), но он есть, и его нужно честно зафиксировать в risk acceptance.

Как работают два токена в Jira Sync (гайд Notion [Unleashing collaboration with Notion's Jira connection](https://www.notion.com/help/guides/unleashing-collaboration-with-notions-jira-connection)):

> While admin authentication handles the heavy lifting of syncing everything, your personal authentication lets us know who you are and what private projects you can bring over to Notion.

> Make sure your Jira admin account has access to these projects too. Remember that only folks with access to a project can add it to Notion, even if an admin has access.

Вывод: данные тянутся **админским (сервисным) токеном**, а личная аутентификация участника нужна только для того, чтобы разрешить ему добавить проект. Проект попадёт в Notion, только если доступ к нему есть **и у сервисного аккаунта, и у добавляющего участника**. Права сервисного аккаунта — это жёсткий потолок для всего, что вообще может утечь в Notion.

### 2.2. «Не даёт выбирать, синкает всё целиком» — неверно (на уровне проектов)

Из той же справки Notion:

> From here, select the data you want to sync into Notion. First **Select projects to sync**, then **Select properties to sync**.

Выбираются конкретные проекты и конкретные свойства. Что действительно **нельзя**:

- задать JQL/фильтр по типам задач, статусам и т.п. — синхронизируется проект целиком (все work items выбранных проектов). Фильтр «только эпики и сторис» в нативном Jira Sync сделать нельзя; можно лишь спрятать таски в Notion фильтрами видов, но это не граница безопасности, а отображение;
- ограничить синхронизацию отдельными полями внутри проекта иначе, чем через «Select properties».

Ещё одно ограничение, упомянутое в стороннем гайде (декабрь 2025) и отсутствующее в официальной справке: синхронизируются только задачи, созданные или обновлённые за последний год («You can only sync items created or updated within the last year, currently») — стоит проверить в песочнице, если важна старая история `PRO`.

Комментарии и вложения (до 5 файлов ≤ 1 MB на задачу) тоже синхронизируются — это важно для оценки, что именно из `PRO` окажется в Notion.

### 2.3. «Наследует права Notion» — верно

> Note: Jira synced databases will inherit Notion permissions. Members can access synced databases on any pages they have permission to view. If a member adds a private Jira project to a Notion page, anyone with Notion page access will see it.

Это фундаментальное свойство любого копирования данных в Notion (и нативного, и через любой сторонний синк): права Jira **не переносятся**, аудитория определяется правами страницы/тимспейса Notion. Единственное исключение — two-way sync на Enterprise-плане Notion, где **редактирование** проверяется по правам Jira конкретного пользователя (для чтения это ничего не меняет).

Точно так же уже работает разрешённый Connect (справка Notion [Link previews](https://www.notion.com/help/link-previews)):

> Note: Once you've authenticated a connected app, anyone who can view your Notion page will be able to see corresponding content that you've pasted as a link preview.

То есть argument agoriachev верен: разница между Connect и Sync — в **масштабе** (отдельные задачи vs. проект целиком), а не в модели доступа.

### 2.4. «Кто-то добавит приватный проект» — верно, но ограничено двумя условиями

Чтобы приватный проект `X` оказался в Notion, одновременно должны выполняться:

1. у **сервисного аккаунта** есть `Browse Projects` в `X` (см. 2.1 — данные тянет он);
2. **участник**, добавляющий проект, сам имеет доступ к `X` в Jira и осознанно проходит аутентификацию.

Если сервисный аккаунт ограничен `PRO`, условие 1 закрывает сценарий «по неосторожности». Остаётся сценарий «умышленно + при широком доступе сервисного аккаунта», который закрывается правильной настройкой прав (раздел 4.1) и, для верности, проверкой в песочнице (раздел 4.1.5).

### 2.5. Почему у banar было «Content not found» в превью

Это ошибка **Connect (link preview)**, а не Jira Sync. По справке Notion, две типовые причины: превью запрошено не тем аккаунтом, у которого есть доступ к задаче (нужно переподключить Jira в Settings → My connections), либо организация ограничила доступ сторонних подключений/IP. К синхронизации это отношения не имеет.

## 3. Факты о модели безопасности Jira Cloud, от которых зависит решение

| Факт | Что это значит для задачи | Источник |
|---|---|---|
| `Administer Jira` — глобальное право; `Browse Projects` — проектное, выдаётся только permission scheme | Админ-аккаунт можно ограничить одним проектом по данным, сохранив право создавать webhooks | [Jira permissions overview](https://support.atlassian.com/jira/kb/jira-permissions-general-overview/) |
| В типовой permission scheme `Browse Projects` выдан «Any logged-in user» (application access) | Любой аккаунт с лицензией Jira, включая сервисный, видит все проекты на такой схеме. Чтобы потолок был строго `PRO`, схемы остальных проектов должны выдавать Browse группам/ролям, а не «любому залогиненному» | там же; практика администрирования Jira |
| Ограничить права по **типам задач** штатно нельзя; есть только Issue Security Scheme (уровни безопасности на отдельных задачах) | Фильтр «эпики и сторис, без тасков» на уровне прав недостижим; возможен лишь костыль — автоматизация, ставящая security level всем таскам | agoriachev прав; см. [Issue security](https://support.atlassian.com/jira-cloud-administration/docs/configure-issue-security-schemes/) |
| Admin-webhooks (UI или `POST /rest/webhooks/1.0/webhook`) создаёт только `Administer Jira`; они хранятся в Jira, и админ видит/редактирует их JQL и события в Settings → System → WebHooks | После настройки Notion Sync Jira-админ может проверить и при необходимости ужесточить JQL-фильтр webhook'ов, которые создал Notion | [Webhooks (developer docs)](https://developer.atlassian.com/cloud/jira/platform/webhooks/), [Manage webhooks](https://support.atlassian.com/jira-cloud-administration/docs/manage-webhooks/) |
| Создание webhook'ов **не попадает** в audit log Jira Cloud (открытая жалоба), изменения permission schemes — попадают | Контроль за токеном строится на аудите схем прав + периодической ревизии списка webhooks | [Community: missing audit log entry for webhook](https://community.atlassian.com/forums/Jira-questions/Jira-Cloud-admin-Missing-audit-log-entry-when-creating-a-new/qaq-p/3109925), [JRASERVER-75942](https://jira.atlassian.com/browse/JRASERVER-75942) |
| API-токены Atlassian живут **1–365 дней**, по умолчанию год; вечных больше нет | Токен Jira Sync придётся обновлять минимум раз в год, иначе «Sync stopped»; нужен владелец процесса | [Manage API tokens](https://support.atlassian.com/atlassian-account/docs/manage-api-tokens-for-your-atlassian-account/) |
| У Atlassian есть штатные **service accounts** (Directory → Service accounts): 5 бесплатно на организацию, не занимают лицензию, не могут логиниться в UI, подчиняются тем же правам, что и люди; их API-токены **только со скоупами** | Формальный сервисный аккаунт — хорошая практика, но Notion рекомендует **scopeless** токен, поэтому реалистичнее «обычный» managed-аккаунт (почтовый ящик `notion-sync@…`) с выданным `Administer Jira`; скоупированный токен можно попробовать в песочнице | [Understand service accounts](https://support.atlassian.com/user-management/docs/understand-service-accounts/), [Manage API tokens for service accounts](https://support.atlassian.com/user-management/docs/manage-api-tokens-for-service-accounts/) |
| Atlassian Guard: политика «user API token» (allow/block) и контроль срока жизни токенов | Если в организации API-токены заблокированы политикой, сервисный аккаунт нужно вынести в отдельную authentication policy с разрешением | [Block user API token access](https://support.atlassian.com/security-and-access-policies/docs/set-api-token-access/) |
| «App access rules» в Data Security Policies ограничивают доступ **приложений** (Marketplace/Connect/Forge, OAuth-apps) к выбранным проектам (до 15 проектов на инстанс в политике) | К Jira Sync **не применимы** — он ходит обычным пользовательским токеном, а не как app. Зато применимы к сторонним синк-приложениям из Marketplace (например, Getint) | [App access rule (Atlassian community)](https://community.atlassian.com/forums/Trust-Security-articles/Control-app-access-to-content-in-Jira-and-Confluence-with-a-new/ba-p/2712247), [DSP developer guide](https://developer.atlassian.com/cloud/jira/platform/data-security-policy-developer-guide/) |
| Synced databases в Notion доступны на планах **Business и Enterprise**; two-way sync, ограничение списка устанавливаемых connections и audit log — только **Enterprise** | Часть организационных мер зависит от плана Notion | [Synced databases](https://www.notion.com/help/synced-databases), [Enterprise connection settings](https://www.notion.com/help/enterprise-connection-settings) |

## 4. Варианты решения

### 4.1. Вариант A — нативный Notion Jira Sync + сервисный аккаунт с потолком `PRO`

Самый простой по внедрению и единственный, где в Notion получается «настоящая» synced database (бейдж Synced, автоматические projects/work items базы, identity mapping людей, комментарии и вложения, real-time через webhooks, two-way на Enterprise).

#### 4.1.1. Как устроен поток данных

1. Workspace owner Notion один раз вводит e-mail сервисного аккаунта, URL сайта Jira и его API-токен (Settings → Import → Jira Sync).
2. Notion этим токеном читает проекты/задачи и регистрирует admin-webhooks в Jira.
3. Участники создают синки в тимспейсах, к которым имеют доступ; чтобы добавить проект, проходят личную аутентификацию в Jira. Добавить можно только проект, доступный и участнику, и сервисному аккаунту.
4. Обновления приходят через webhooks «в течение минут»; раз в сутки при открытии базы — resync на случай пропущенных событий.
5. Токен истекает максимум через год → workspace owner нажимает Re-authenticate.

#### 4.1.2. Настройка на стороне Jira (потолок = `PRO`)

1. **Аккаунт.** Создать выделенный managed-аккаунт (`notion-sync@company`) или штатный Atlassian service account. Ему не нужны роли org-admin / site-admin — только продуктовый доступ к Jira и глобальное право `Administer Jira` через **отдельную группу** (например, `svc-notion-jira-admin`), а не через дефолтную группу `jira-admins-*`, которая часто фигурирует в permission schemes.
2. **Данные.** Дать `Browse Projects` в `PRO` через роль проекта или группу (`svc-notion-readers`) — по образцу KB Atlassian «Create a read-only user in Jira Cloud»: «Ensure the Browse Projects and Administer Projects permissions are not set to 'Any logged in user' or to users with 'Application access' to Jira». Для two-way (если когда-нибудь понадобится) — дополнительно `Edit Issues`, `Add Comments`, `Create Attachments` в `PRO`.
3. **Потолок.** Проверить, что ни одна другая permission scheme не выдаёт `Browse Projects` этому аккаунту: главный источник утечки — грант «Any logged-in user» в дефолтной схеме. Два уровня строгости:
   - *Уровень 1 (без переделки схем):* потолок сервисного аккаунта = «всё, что видит любой сотрудник с лицензией Jira». Приватные проекты (бюджеты, HR и т.п.) на схемах с ролями/группами остаются невидимыми. В Notion при этом выбирается только `PRO`.
   - Оговорка: team-managed проекты управляют доступом своими ролями и всегда остаются видимыми в списке проектов по названию; для них проверка делается отдельно.
   - *Уровень 2 (строгий, потолок = только `PRO`):* в схемах остальных проектов заменить грант «Any logged-in user» на группу всех сотрудников (`all-employees`), в которую сервисный аккаунт не входит. Это та самая «усложнённая ACL», но она одноразовая, и это в целом хорошая гигиена для любых интеграционных аккаунтов.
4. **Проверка.** Permission helper (Settings → System → Permission helper) и запрос `GET /rest/api/3/search/jql?jql=project!=PRO` от имени сервисного аккаунта должны показать 0 задач вне `PRO`.
5. **Токен.** Создать API-токен (Notion рекомендует scopeless), срок — максимум 365 дней; завести напоминание о ротации. Если в организации действует политика Atlassian Guard, блокирующая API-токены, — вынести аккаунт в отдельную authentication policy.
6. **Issue security (опционально).** Если внутри `PRO` есть отдельные чувствительные задачи, повесить на них security level, недоступный сервисному аккаунту (Notion не синхронизирует поле Security level, но задачу с недоступным уровнем и не увидит).
7. **После первой синхронизации.** Открыть Settings → System → WebHooks, убедиться, что webhooks, созданные Notion, ограничены JQL по `PRO`, и зафиксировать их список для периодической ревизии (в audit log их создание не попадает).

#### 4.1.3. Настройка на стороне Notion

- Класть synced database только в **закрытый тимспейс** с явным списком участников (Business+: private teamspaces); не давать гостям и не публиковать.
- Enterprise: включить `Limit which connections members can install` → «Approved only», управлять списком approved connections; включить audit log. (Нужно проверить в песочнице, распространяется ли это ограничение на Jira Sync, который живёт в Settings → Import, а не в Connections — прямого утверждения в документации нет.)
- Учесть, что с мая 2026 «Any member can build connections (not just Workspace Owners)» ([release 2026-05-13](https://www.notion.com/releases/2026-05-13)) — это про API-connections; первичная настройка Jira Sync по-прежнему только у workspace owner, но новые синки в доступных тимспейсах может создавать любой участник.
- Договориться, что новые синки/проекты добавляет только владелец процесса; при необходимости — попросить Notion support отключить самостоятельное добавление проектов участниками (если такая настройка есть; в документации не описана).
- Помнить про «Note: If you delete a property in Jira, the associated database item in your synced database will be deleted too» и про ограничение «Jira properties with more than 1,000 values won't be imported».

#### 4.1.4. Что не решается этим вариантом

- Фильтр по **типам задач** внутри `PRO` — нет (см. 2.2). Обходные пути: (а) вынести таски в отдельный проект; (б) костыль с Issue Security по типу через автоматизацию; (в) любой из вариантов B–F, где есть условия/JQL.
- Токен с `Administer Jira` хранится у Notion. Компенсирующие меры: аудит изменений permission schemes, ревизия webhooks, ротация токена, невозможность интерактивного входа для сервисного аккаунта, минимальные права по данным.

#### 4.1.5. Что обязательно проверить в песочнице перед решением

1. Сервисный аккаунт с `Administer Jira` и Browse только в `PRO` проходит настройку Jira Sync, список проектов и импорт работают.
2. Участник, имеющий доступ к проекту `X` (сервисный аккаунт — нет), пытается добавить `X`: ожидаем отказ/пустую базу. Проверить, не появился ли в Jira webhook с JQL, включающим `X`, и не пришло ли в Notion ни одной задачи `X`. (Открытый вопрос: admin-webhooks Jira формально не фильтруются правами создавшего их пользователя; поэтому сам факт регистрации webhook на `X` уже был бы риском — это нужно увидеть глазами.)
3. ~~Попробовать **скоупированный** токен~~ — уже проверено Jira-админом компании в проде (25.09.2026): со скоупированным токеном Notion отвечает «нужен админ» и не синхронизирует; у штатных Atlassian service accounts токены бывают только со скоупами. Вывод: для нативного Jira Sync нужен обычный managed-аккаунт с почтовым ящиком в authentication policy без SSO и scopeless-токен. Это делает остаточный риск варианта A (полноценный admin-токен у Notion) неустранимым.
4. Истечение токена: как выглядит «Sync stopped» и кто получает уведомление.
5. Enterprise-настройки Notion: ограничение connections, audit log событий Jira Sync.

### 4.2. Вариант B — Jira Automation → Notion API напрямую (без посредников и без Jira-токена у третьей стороны)

Идея: правило автоматизации **в проекте `PRO`** при создании/изменении/переходе/удалении задачи само вызывает Notion API (`Send web request`) и создаёт/обновляет страницу в обычной Notion-базе. Ни один токен Jira никуда не уходит; наружу передаётся только то, что явно описано в правиле; единственный секрет — internal-integration токен Notion с доступом к **одной** базе, хранящийся в правиле как скрытое значение.

#### 4.2.1. Что подтверждает документация Atlassian

- Действие `Send web request` поддерживает произвольный JSON (Custom format), кастомные заголовки и **скрытые значения**: «If a value is marked as hidden and the flow is saved, the value will be replaced by asterisks … This can't be reversed» ([Jira automation actions](https://support.atlassian.com/cloud-automation/docs/jira-automation-actions/)).
- Ответ можно использовать дальше: «You can set this action to return response data that can then be used in a subsequent action» — через smart values `{{webhookResponse.status}}` / `{{webhookResponse.body…}}` (в KB встречается и синоним `{{webResponse…}}`) при включённом «Wait for response → Delay execution of subsequent rule actions until we've received a response» ([smart values](https://support.atlassian.com/cloud-automation/docs/jira-smart-values-issues/), [KB](https://support.atlassian.com/automation/kb/getting-an-empty-value-when-using-the-webhookresponse-smart-value/)); ответ должен приходить с `Content-Type: application/json`, по сообщениям сообщества — до ~10 МБ и таймаут ~30 с. Это позволяет делать upsert: сначала запрос к Notion «найти страницу с Jira Key = PRO-123», затем ветка if/else: создать или обновить.
- Есть триггер **Incoming webhook** (URL + секрет в заголовке `X-Automation-Webhook-Token`) — для обратного направления Notion → Jira, если понадобится ([docs](https://support.atlassian.com/cloud-automation/docs/configure-the-incoming-webhook-trigger-in-atlassian-automation/)).
- Управление: правила проекта могут вести project admins (если Jira-админ это разрешил); список редакторов правила ограничивается. Atlassian прямо предупреждает: «someone with permission to edit the flow can reconfigure the Send web request action to send out data that they shouldn't … ensure that only people you trust can edit automation flows». На Enterprise-плане Jira глобальный админ может дополнительно **ограничить домены**, куда разрешено слать web request (allowlist `api.notion.com`) ([Add restrictions to automation components](https://support.atlassian.com/cloud-automation/docs/add-restrictions-to-automation-components/)).
- **Служебные лимиты:** до 65 шагов в правиле (500 в advanced flow), 60 минут процессорного времени на 12 часов, «Work items searched: 999» ([service limits](https://support.atlassian.com/cloud-automation/docs/automation-service-limits/)).
- **Лимиты использования (важно, изменились в 2026).** Автоматизация теперь тарифицируется «шагами», пул на организацию: Jira Free — 150 шагов на подписку, Standard — 400 на пользователя, Premium — 750, Enterprise — 1000 в месяц; каждый триггер, условие, действие, ветка и цикл = 1 шаг; с 3 декабря 2026 включается биллинг перерасхода ($0.50 за 1 000 шагов) либо остановка правил до сброса ([How is your automation usage calculated](https://support.atlassian.com/cloud-automation/docs/how-is-my-usage-calculated/)). Один upsert ≈ 5–6 шагов (триггер, условие, запрос-поиск, ветка, запрос создания/обновления). При 2 000 событий в месяц по `PRO` это ≈ 12 000 шагов — для организации на Standard с 100+ пользователями (40 000 шагов) укладывается, но нужно посчитать по реальному потоку событий.

#### 4.2.2. Схема правил (эскиз)

```
Trigger: Issue created | Field value changed (Summary, Status, Assignee, Priority, Labels, Fix versions, Parent, …) | Issue transitioned
Condition: issue type in (Epic, Story)          <- вот здесь решается вопрос "без тасков"
Action 1: Send web request  POST https://api.notion.com/v1/data_sources/{DS_ID}/query
          Headers: Authorization: Bearer <hidden>, Notion-Version: 2026-03-11
          Body: {"filter":{"property":"Jira Key","rich_text":{"equals":"{{issue.key}}"}}}
          [x] Delay execution until response
If {{webhookResponse.body.results.size}} = 0
    Action 2a: POST https://api.notion.com/v1/pages   (parent: data_source_id, properties: …)
Else
    Action 2b: PATCH https://api.notion.com/v1/pages/{{webhookResponse.body.results.first.id}}  (properties: …)

Отдельное правило: Trigger Issue deleted -> PATCH /v1/pages/{id} {"in_trash": true}
Отдельное правило (опционально): Comment added -> PATCH /v1/blocks/{page_id}/children (добавить блок с комментарием)
```

Первичная загрузка истории (backfill) делается один раз скриптом через Notion API или CSV-импортом, а не автоматизацией.

#### 4.2.3. Плюсы и минусы

Плюсы: ноль инфраструктуры и ноль внешних вендоров; никаких Jira-учёток и админ-токенов вне Atlassian; правило само по себе — allowlist проектов (правило живёт в `PRO`; для нескольких проектов — multi-project rule с явным списком); условия по типу задачи, статусу, меткам; полная прозрачность того, какие поля уходят; audit log выполнений.

Минусы: это не «synced database» Notion, а обычная база (нет бейджа Synced, нет автоматических projects/work items баз, identity mapping людей нужно делать руками через `people`-свойство и список пользователей Notion); одностороннее по умолчанию; поддержка правил — на команде `PRO`/Jira-админах; расход шагов автоматизации; при переименовании свойств в Notion правило ломается; Notion API ~3 запроса/с, для одного проекта достаточно.

### 4.3. Вариант C — приватное Forge-приложение (свой код, но внутри Atlassian)

[Forge](https://developer.atlassian.com/platform/forge/) — serverless-платформа Atlassian. Приватное приложение подписывается на события задач и само вызывает Notion API; никакого внешнего хостинга и никаких Jira-учёток у третьих лиц.

- **Фильтр по проекту на уровне манифеста, до вызова кода:** триггеры `avi:jira:created:issue`, `avi:jira:updated:issue`, `avi:jira:deleted:issue` с выражением `filter: expression: "event.issue.fields.project.key == 'PRO'"` — «Use manifest-level filtering to prevent Forge from invoking your function at all for events you don't care about» ([Optimise Forge costs](https://developer.atlassian.com/platform/forge/optimise-forge-costs/), [product events](https://developer.atlassian.com/platform/forge/events-reference/product_events/)). Тип задачи фильтруется тем же выражением.
- **Выход в Notion:** внешний домен нужно объявить в `permissions.external.fetch.backend` (`api.notion.com`), иначе вызов упадёт ([egress permissions](https://developer.atlassian.com/platform/forge/runtime-egress-permissions/)). Из-за этого приложение **не** получает статус «Runs on Atlassian» и выпадает из data residency по умолчанию — придётся описать, какие данные уходят ([Runs on Atlassian](https://developer.atlassian.com/platform/forge/runs-on-atlassian/), [data residency](https://developer.atlassian.com/platform/forge/data-residency/)). В preview есть customer-managed egress: админы видят и могут отключать egress-домены приложения в Connected apps.
- **Секрет Notion:** `forge variables set --encrypt NOTION_TOKEN …`, в рантайме `process.env` ([environments and versions](https://developer.atlassian.com/platform/forge/environments-and-versions/)).
- **Установка:** приватно, без публикации — `forge install --site … --product jira` от имени администратора сайта ([distribute your apps](https://developer.atlassian.com/platform/forge/distribute-your-apps/)); скоуп `read:jira-work` для событий.
- **Расписание для сверки:** `scheduledTrigger` (fiveMinute / hour / day / week, до 5 на приложение, таймаут до 900 с) — удобно для ночного reconcile и подчистки удалённых.
- **Лимиты:** async-функции до 900 с; 7 000 вызовов/мин на установку; для Notion — обычные 180/600 запросов в минуту.
- Заметка про app access rules: они распространяются на установленные Marketplace/Forge-приложения, но «A private app you are developing on Atlassian's Forge platform» указан как исключение ([coverage summary](https://support.atlassian.com/security-and-access-policies/docs/app-access-rule-coverage-summary/)) — граница проекта здесь обеспечивается манифестом и кодом, а не политикой.

**Итог:** по сути это вариант 4.6.2 (собственный сервис), размещённый у Atlassian: те же задачи по upsert-логике в Notion, но ноль инфраструктуры и push в реальном времени. Оправдан, если в компании уже есть команда, работающая с Forge; иначе Jira Automation (4.2) даёт то же самое без кода, а Notion Workers (4.6.1) — с меньшими усилиями и настоящей synced database.

### 4.4. Вариант D — готовые синк-продукты: Getint и Unito

Из всего рынка только два продукта сделаны именно под Jira ↔ Notion и одновременно позволяют ограничивать синк проектом/фильтром и работать от **не-админского** аккаунта Jira.

#### Getint — «Notion Integration for Jira (Notion Connector) FORGE» (Atlassian Marketplace)

- **Скоуп:** одна интеграция = один проект Jira, плюс **JQL-фильтр** внутри проекта — `project = PRO AND issuetype IN (Epic, Story)` делается штатно ([docs](https://docs.getint.io/guides/integration-synchronization/jira-notion-integration): «integrations involving Jira support Jira Query Language (JQL). This allows for granular control over exactly which issues enter the synchronization scope»).
- **Аккаунт Jira:** «Jira instances must have a dedicated user and an associated API token with permissions to read, write, view, and modify the project» — выделенный сервисный аккаунт с правами только в проекте; Jira-админ нужен один раз, чтобы установить приложение из Marketplace. Как Marketplace-приложение оно подпадает под Atlassian **app access rules** — ещё один слой ограничения проектами на стороне организации.
- **Направление:** одностороннее в Notion, одностороннее в Jira или двустороннее; «near real-time» (точный интервал в документации не указан).
- **Поля:** title, description (лимит 2 000 символов), status mapping, assignee, комментарии, custom fields; вложения в Notion не синхронизируются.
- **Безопасность:** SOC 2 Type II, GDPR/CCPA, AWS с возможностью выбора региона (в т.ч. EU), логи 1–14 дней или отключены, есть on-prem вариант; в Marketplace: «Stores Personal Data: No», Cloud Fortified, Bug Bounty.
- **Цена:** тарифицируется по числу пользователей **Jira** через Atlassian: до 10 пользователей бесплатно, 100 пользователей ≈ $3 500/год, 1 000 ≈ $9 050/год ([Marketplace pricing](https://marketplace.atlassian.com/apps/1231784/notion-integration-for-jira)). Для большой Jira-инсталляции это дорого ради одного проекта.

#### Unito

- **Скоуп:** flow на пару «проект Jira ↔ база Notion» + правила фильтрации, включая «Issue type is Epic», метки, исполнителей; JQL нет ([rules](https://guide.unito.io/how-to-set-up-rules)).
- **Аккаунт Jira:** подключение по OAuth требует администратора сайта («Ensure you have administrator access to your Jira instance»); есть путь через API-токен для не-админов; Unito действует строго от имени подключённого пользователя, т.е. потолок — его права ([permissions](https://guide.unito.io/what-permissions-are-required-for-jira-users)).
- **Направление и задержка:** одно- или двусторонний по каждому полю; Jira — по webhooks, Notion — только опросом каждые 5 мин (на self-serve тарифах до 15 мин).
- **Ограничения в сторону Notion:** без комментариев, вложений и форматированного текста; блок текста ≤ 2 000 символов; «Only one Notion user per workspace can connect to Unito»; связанные задачи не поддерживаются ([limitations](https://guide.unito.io/limitations-notion-integration)).
- **Безопасность:** SOC 2 Type 2, хостинг AWS **только в США**, содержимое задач не хранится (только контрольные суммы), вложения не хранятся ([security](https://unito.io/security/)).
- **Цена:** на сайте цифр нет (in-app); по сторонним источникам ≈ $65/мес (Basic, 750 items) и ≈ $299/мес (Pro, 2 500–10 000 items); items считаются **с обеих сторон** (задача в Jira + строка в Notion = 2 items).

**Остальные «синк-платформы» отпадают:** Exalate не поддерживает Notion; Whalesync — «Jira isn't available to sync yet»; 2sync, Coupler.io, Hevo, Fivetran, Sync2Sheets — нет пары Jira→Notion; Bardeen и Nekton — ручные/почасовые запуски, не live. Небольшие Marketplace-приложения (Korvex «Notion Connector for Jira», Appvibe) слишком незрелые (3 и 19 установок, без отзывов и публичной документации) для чувствительной интеграции.

### 4.5. Вариант E — low-code/iPaaS: n8n, Make, Zapier, Zoho Flow, Power Automate, Workato

Общая схема одинакова: триггер по событию/опросу Jira → (фильтр по проекту и типу) → поиск строки в Notion по ключу → создать/обновить. Различия — в том, **нужен ли Jira-админ для live-триггера**, где хранятся данные прогонов и сколько стоит.

| Платформа | Live-триггер | Фильтр проект/тип/JQL | Нужен ли Jira-админ | Данные у вендора / регион | Цена (2026) |
|---|---|---|---|---|---|
| **n8n** (self-hosted или Cloud EU) | Jira Trigger создаёт webhook с JQL-фильтром; или опрос по расписанию (`updated >= -5m`) | JQL в триггере и в запросах | Webhook по API-токену — да (Administer Jira); через OAuth-приложение со скоупом `manage:jira-webhook` — динамические webhooks без админа (обновлять раз в 30 дней); опрос — нет | self-hosted: ничего не уходит; Cloud: EU, SOC 2 | self-hosted бесплатно (Sustainable Use License, внутреннее использование); Cloud от 20 €/мес |
| **Make** | «Watch Issues» — webhook, который нужно создать в админке Jira; иначе опрос | JQL в admin-webhook; опрос — JQL | Да для instant; нет для опроса | логи 30 дней; SOC 2 Type II, ISO 27001; EU-зона | Free 1 000 credits (15 мин); Core $12/мес (1 мин) |
| **Zapier** | только опрос (1–15 мин по тарифу) | проект во всех триггерах; JQL только в «New Issue (Via JQL)»; тип — Filter-шаг | Нет (OAuth пользователя) | история прогонов хранится; только США | Free 100 tasks; Pro от $19.99/мес; Team от $69/мес |
| **Zoho Flow** | опрос 5/15 мин | триггеры «Issue created/updated in selected type» — проект + тип штатно | Нет (API-токен) | хранит данные прогонов (детали не подтверждены) | ≈ $10–24/орг/мес (не подтверждено) |
| **Power Automate** | опрос 60 с; **нет** триггера «issue updated» | проект + JQL только для новых задач | Нет (API-токен) | тенант Microsoft | Premium $15/польз./мес; Notion-коннектор только Independent Publisher без обновления свойств — слабый вариант |
| **Workato / Tray / Pipedream** | webhooks (Workato: нужен админ для регистрации) | да | Workato: service account token; real-time — админ | enterprise-контроли, региональный хостинг у Tray | по запросу (Workato ≈ $10k+/год по сторонним оценкам) |

Практический вывод по iPaaS: для команды с требованиями безопасности реально интересен только **n8n self-hosted** (данные не покидают периметр, JQL, бесплатно) — но это уже «свой сервис» с поддержкой (см. 4.6), просто без программирования. Zapier/Make/Zoho Flow подойдут для дешёвого одностороннего фида без админа, если допустимо хранение payload'ов у вендора и опрос раз в несколько минут.

### 4.6. Вариант F — своя синхронизация через Notion API: Notion Workers (хостинг у Notion) или собственный сервис

#### 4.6.1. Notion Workers — «synced database без Jira-админа» (новое, beta)

В мае 2026 Notion выпустил **Workers** — хостируемую среду для собственного кода: «Notion Workers are small Node/TypeScript programs that extend Notion. You write code, deploy it with the Notion CLI, and Notion hosts and runs it for you. No servers to manage» ([What are Notion Workers?](https://developers.notion.com/workers/get-started/overview)). Одна из трёх возможностей — **Syncs**: «A sync pulls data from external sources … and writes it to a Notion database. You define a schema for the database and an execute function that returns the data. Notion runs it on a schedule and manages the database for you» ([Syncs](https://developers.notion.com/workers/guides/syncs)).

Почему это почти идеально ложится на задачу:

- **Jira-админ не нужен вообще.** Worker сам ходит в Jira REST API (`GET /rest/api/3/search/jql` с `nextPageToken`) с токеном **read-only сервисного аккаунта**, у которого есть только `Browse Projects` в `PRO`. Webhooks не используются — значит, `Administer Jira` не требуется. Подходит даже штатный Atlassian service account со скоупированным токеном (`read:jira-work`).
- **Фильтрация любой глубины — JQL:** `project = PRO AND issuetype IN (Epic, Story) AND updated >= "<последний запуск>"`. Вопрос banar про «эпики/сторис без тасков» решается одной строкой.
- **Результат — настоящая synced database:** «Synced columns are read-only … Rows cannot be added or deleted manually. Only the sync creates rows». Upsert по ключу задачи: «primaryKeyProperty tells Notion which property uniquely identifies each row … When your sync emits a record with the same key, Notion updates the existing row instead of creating a duplicate». Поддерживаются relations между синхронизируемыми базами (`Schema.relation`) — можно построить пары «Epics ↔ Stories» или «Projects ↔ Issues», как в нативном Jira Sync.
- **Режимы и удаления:** `replace` (полная выгрузка, «any rows not seen during that cycle are automatically deleted») и `incremental` (только изменения, удаления явно). Рекомендуемый паттерн — delta-sync каждые 5 минут + ручной/периодический backfill в replace-режиме, который подчищает удалённые в Jira задачи.
- **Секреты:** «Notion encrypts worker secrets at rest and exposes them as environment variables at runtime» (`ntn workers env set JIRA_TOKEN=…`). У Notion в этом случае хранится не админский scopeless-токен, а read-only токен, ограниченный одним проектом.
- **Расписание:** по умолчанию 30 мин, «Minimum schedule is "5m", maximum is "7d"». Это не real-time, но для трекинга проекта достаточно.
- **Лимиты:** 600 запусков синков в час на воркспейс, 1 000 000 операций записи в базы в час ([Limits](https://developers.notion.com/workers/reference/limits)) — с запасом.
- **Управление доступом:** воркер шарится из Developer portal с уровнями «Can connect» / «Full access» (последний нужен, чтобы менять секреты и деплоить); исходники — в своём Git, деплой из CI по personal access token с capability Workers ([Sharing Workers](https://developers.notion.com/workers/guides/sharing-workers)).

Что нужно учесть:

- **Beta и стоимость.** «Workers are free to try during the beta period. Starting August 11 2026, Workers will run on Notion credits» ([release 2026-05-13](https://www.notion.com/releases/2026-05-13)); на странице тарифов Workers помечены «Requires Notion credits». Реальную стоимость нужно посчитать на пилоте (число запусков × объём изменений).
- **Нужен разработчик** на 1–2 дня: ~150–300 строк TypeScript (запрос в Jira, маппинг полей в `Builder.*`, пагинация, pacer под лимиты Jira) + поддержка при изменении схемы. Notion прямо позиционирует Workers как «designed to be built with AI coding agents».
- Идентичность людей: маппинг Jira-исполнителей на пользователей Notion придётся делать самим (по e-mail через Notion API); в нативном Jira Sync это делает identity mapping.
- Комментарии и вложения Jira — только если вы сами их вытянете и положите в свойства/контент; в схеме синка удобнее ограничиться полями.
- Права в Notion — те же, что и везде: база наследует права страницы/тимспейса.

#### 4.6.2. Собственный сервис (Cloud Run / Lambda / k8s / n8n) + Notion API

Максимальный контроль: данные проходят только через ваш периметр и Notion. Ключевые факты по Notion API (developers.notion.com, сентябрь 2026):

- Актуальная версия API — **2026-03-11** (заголовок `Notion-Version` обязателен); с 2025-09-03 модель «database → data source → page»: страница создаётся с `parent.data_source_id`, запрос строк — `POST /v1/data_sources/{id}/query` с фильтром по свойству (`"Jira Key" rich_text equals "PRO-123"`), обновление — `PATCH /v1/pages/{id}`, удаление — `in_trash: true` ([upgrade guide 2025-09-03](https://developers.notion.com/docs/upgrade-guide-2025-09-03), [upgrade guide 2026-03-11](https://developers.notion.com/docs/upgrade-guide-2026-03-11)).
- Лимиты: **600 запросов/мин** на connection для Business/Enterprise и **180/мин** для остальных планов + общий лимит на воркспейс; 429/529 с `Retry-After` ([Request limits](https://developers.notion.com/reference/request-limits)). Запрос к data source отдаёт максимум **10 000** результатов — при больших базах индекс строится окнами по `created_time`.
- Свойства: select/multi_select создаются на лету по имени; status-опции с 2026-03-19 управляются через API (заранее задать список с группами To-do / In progress / Complete); `people` принимает ID пользователей Notion — маппинг с Jira по e-mail через `GET /v1/users` (нужна capability «User information with email addresses»; гости не возвращаются); relation — массив ID страниц (запись заменяет массив целиком).
- Токен — internal integration, подключённый **только к целевой базе**; на Enterprise его нужно внести в approved connections; события «Integration created / permission updated / removed from approved connections» попадают в audit log Notion.
- Источник событий Jira: admin-webhooks с JQL-фильтром (нужен Jira-админ один раз, чтобы создать) **или** опрос `search/jql` по `updated >= -5m` без админа. Бэкфилл 5 000 задач ≈ 30 мин на 180 req/min, ≈ 9 мин на 600 req/min.
- Обратное направление (Notion → Jira), если понадобится: Notion **webhooks** для integration (события `page.properties_updated`, `page.deleted`, `comment.created`; в payload только ID — нужен повторный запрос; доставка at-most-once с 8 повторами в течение ~24 ч; подпись `X-Notion-Signature`) или без кода — database automation Notion «Send webhook» (платные планы) в Incoming webhook триггер Jira Automation.
- Data residency: на Enterprise Notion можно хранить данные воркспейса в EU (Франкфурт, с сентября 2025) ([Data residency](https://www.notion.com/help/data-residency)); SOC 2 Type 2, ISO 27001/27701/27017/27018 ([Security](https://www.notion.com/security)).

Готовых зрелых open-source реализаций нет (несколько заброшенных репозиториев на 4–17 звёзд), так что это разработка «с нуля» на 1–2 недели с учётом мониторинга, reconcile и обработки удалений. По сравнению с Workers выигрывает только полным контролем над инфраструктурой и возможностью real-time через Jira-webhooks.

### 4.7. Вариант G — не копировать данные вообще: Notion AI connector for Jira, Confluence

Если реальная потребность части команды — «находить и спрашивать про задачи Jira из Notion», а не вести поверх них базы, есть путь без synced database.

**Notion AI connector for Jira** (справка [Notion AI connector for Jira](https://www.notion.com/help/jira-ai-connector), beta):

- Индексирует задачи, комментарии, кастомные поля Jira для Notion AI / Enterprise Search; дашборды, фильтры и доски не индексируются; глубина — год назад; первичная индексация до 36 часов.
- **Уважает права Jira на уровне пользователя:** «Members in a workspace will only have access to retrieve information from Jira if they have access to the Jira site. If additional permissions are set on a project or issues level, users will only be able to ask questions to those that they have access to» и «Every hour, we periodically sync permissions from Jira and update the permissions in Notion». Это единственный Notion-механизм, где аудитория данных Jira ограничена правами самой Jira, а не страницы Notion.
- Требования: Business или Enterprise план Notion; настраивает **Jira admin + Notion workspace owner**; на стороне Atlassian устанавливается Forge-приложение `Notion-AI-Connector` (Marketplace-листинг «Notion AI»), то есть к нему применимы **app access rules** Atlassian (ограничение приложения выбранными проектами) — в отличие от Jira Sync.
- Минусы для задачи banar: это не база, нет видов/rollup/relations; всё равно нужен админский токен для установки; функция в beta.

**Confluence вместо Notion** для «живых» списков задач: макросы Jira в Confluence показывают каждому читателю только то, что он видит в Jira, данные не копируются. Это не то, что просит команда, но это единственный вариант, где вопрос «наследования прав» не возникает по построению; упоминается для полноты.

## 5. Сравнение вариантов

| # | Вариант | Нужен Jira-админ / что хранится вне периметра | Граница «какие проекты» | Фильтр по типам задач | Что получаем в Notion | Задержка | Усилия | Стоимость (сверх Notion) |
|---|---|---|---|---|---|---|---|---|
| A | Нативный Jira Sync + сервисный аккаунт с потолком `PRO` | **Да**: у Notion — scopeless API-токен аккаунта с `Administer Jira` (данные — только `PRO`, если схемы прав настроены) | Права сервисного аккаунта (permission schemes) + выбор проектов в Notion | Нет | Настоящая synced DB: projects + work items, identity mapping, комментарии, вложения; two-way на Enterprise | Минуты (webhooks) | Низкие (админ Jira 0.5–1 день) | 0 |
| A′ | Legacy «Jira» connection с **личным** токеном пользователя (старая synced DB) | Нет админа; в Notion — личный токен сотрудника | Права этого сотрудника | Нет | Устаревшая synced DB одного проекта, «less reliable» (опрос) | Медленнее | Минимальные | 0 |
| B | Jira Automation → Notion API | **Нет**; вне Atlassian — только internal-token Notion на одну базу (в скрытом поле правила) | Правило живёт в `PRO` | **Да** (условие в правиле) | Обычная база Notion (без бейджа Synced) | Секунды–минуты | Средние (правила, JSON), без кода | Шаги автоматизации; с 3.12.2026 — $0.50 / 1 000 сверх пула |
| C | Приватное Forge-приложение | **Админ сайта для установки**; наружу — только вызовы в api.notion.com с токеном Notion | Фильтр в манифесте | **Да** | Обычная база Notion | Секунды | Высокие (разработка + Forge) | Forge бесплатен для приватных приложений |
| D1 | Getint (Marketplace, Forge) | Админ для установки; **сервисный аккаунт без админа** для данных; вендор SOC 2, регион выбирается | Один проект на интеграцию + **JQL** | **Да** (JQL) | Обычная база; статусы, комментарии; без вложений | «near real-time» | Низкие | По числу пользователей Jira (100 польз. ≈ $3 500/год) |
| D2 | Unito | OAuth требует админа сайта, есть путь через API-токен; вендор SOC 2, хостинг США, содержимое не хранит | Проект + правила | **Да** (правило «Issue type is …») | Обычная база; без комментариев, вложений, rich text; Notion опрашивается раз в 5–15 мин | 5–15 мин | Низкие | ≈ $65–299/мес (items считаются с двух сторон) |
| E1 | n8n self-hosted | Админ только если делать webhook по API-токену; иначе опрос без админа; данные не покидают периметр | JQL | **Да** | Обычная база | 1–5 мин (опрос) или секунды (webhook) | Средние + эксплуатация | Бесплатно (лицензия для внутреннего использования) |
| E2 | Zapier / Make / Zoho Flow | Без админа (опрос); payload'ы прогонов у вендора | Проект в триггере, JQL частично | Частично | Обычная база | 1–15 мин | Низкие | $12–70/мес |
| F1 | **Notion Workers (beta)** | **Нет**; в Notion — зашифрованный read-only токен, ограниченный `PRO` | JQL | **Да** | **Настоящая synced DB** (read-only колонки, upsert по ключу, relations Epic↔Story) | 5–30 мин (расписание) | Средние (150–300 строк TS) | Notion credits с 11.08.2026 (посчитать на пилоте) |
| F2 | Свой сервис + Notion API | Нет (опрос) / админ один раз (admin-webhook); свой хостинг | JQL / webhook JQL | **Да** | Обычная база; two-way через Notion webhooks | Секунды | Высокие + эксплуатация | Инфраструктура |
| G | Notion AI connector for Jira | Jira admin + workspace owner для установки; Forge-приложение Notion | App access rules Atlassian + права Jira **по пользователю** | Нет | Не база: поиск/вопросы Notion AI с учётом прав Jira | Индексация до 36 ч | Низкие | Business/Enterprise, beta |

## 6. Рекомендация

Коротко: спор в треде упирается в один артефакт — **scopeless-токен Jira-админа у Notion**. Есть три пути: (1) принять его с компенсирующими мерами, (2) обойтись без него, отдав синхронизацию под контроль своей стороны, (3) обойтись без него, но остаться внутри Notion. Ниже — в порядке предпочтения для этой задачи.

1. **Пилот Notion Workers (F1)** — если план Notion позволяет (Business/Enterprise) и есть один разработчик на пару дней. Это единственный вариант, который одновременно даёт настоящую synced database, JQL-фильтр «эпики и сторис `PRO`», не требует Jira-админа и хранит у Notion только read-only токен, ограниченный проектом. Риски — beta и неизвестная цена в credits; поэтому именно пилот, а не сразу прод.
2. **Нативный Jira Sync с сервисным аккаунтом, ограниченным `PRO` (A)** — если безопасность готова принять админ-токен под контролями из 4.1 (отдельная группа для `Administer Jira`, Browse только в `PRO`, замена «Any logged-in user» в схемах, аудит изменений схем, ревизия webhooks, ротация токена, закрытый тимспейс). Это единственный путь получить «из коробки» комментарии, вложения, identity mapping и two-way. Перед решением — тесты из 4.1.5, особенно попытка добавить недоступный сервисному аккаунту проект.
3. **Jira Automation → Notion (B)** — если разработчика нет, а админ-токен неприемлем. Ноль инфраструктуры, граница проекта и типов задач — в самом правиле, наружу уходят только перечисленные поля. Плата — обычная (не synced) база и расход шагов автоматизации.
4. **Getint (D1)** — если нужен вендорский продукт «поставил и забыл» с JQL и не-админским аккаунтом и устраивает цена по числу пользователей Jira. **n8n self-hosted (E1)** — если есть платформенная команда и требование «данные не покидают периметр» важнее простоты.
5. **Не делать:** Unito без крайней необходимости (Notion опрашивается, нет комментариев/вложений, хостинг только США), Power Automate (нет триггера обновления), незрелые Marketplace-приложения.

Независимо от варианта:

- Права в Notion: synced/обычная база живёт в закрытом тимспейсе с явным списком участников; гостям и публичным ссылкам — нет; на Enterprise — approved connections + audit log.
- Зафиксировать письменно (risk acceptance): что именно из `PRO` попадает в Notion (поля, комментарии, вложения), кто это видит, кто владеет токеном и его ротацией, как всё выключить (удалить базу, отозвать токен/интеграцию, удалить webhooks).
- Фильтр по типам задач на уровне прав Jira невозможен (agoriachev прав); он делается либо в JQL/условии (B, C, D, E, F), либо костылём через Issue Security + автоматизацию, либо не делается (A).

## 7. План пилота (2–3 недели)

1. **Jira (день 1).** Создать сервисный аккаунт `notion-sync`; группу `svc-notion-readers` с Browse в `PRO`; проверить Permission Helper и запрос `search/jql` с `project != PRO` → 0 результатов. Если проверяется вариант A — дополнительно группа `svc-notion-jira-admin` с `Administer Jira` и аудит схем прав. Выпустить токен со сроком 365 дней, завести напоминание о ротации.
2. **Notion (день 1).** Закрытый тимспейс «Jira PRO», участники — только команда `PRO`. Internal integration «jira-pro-sync» с доступом к одной базе (для B/F2) или Worker (F1).
3. **Реализация (дни 2–5).** F1: worker с delta-sync `5m` по JQL `project = PRO AND issuetype IN (Epic, Story) AND updated >= …` и ручным backfill в replace-режиме; поля: ключ, summary, тип, статус, приоритет, исполнитель (e-mail), метки, спринт, parent/epic (relation), даты, ссылка. B: два правила (upsert и delete) с проверкой шагов на 1 000 событий.
4. **Проверки безопасности (дни 6–8).** Что реально ушло в Notion (сравнить с JQL); попытка добавить/вытащить другой проект; поведение при истёкшем токене; лимиты; логи/аудит; процедура отключения.
5. **Решение (неделя 3).** Сравнить с A по потребностям команды (нужны ли комментарии/вложения/two-way) и принять вариант; оформить risk acceptance.

## 8. Открытые вопросы для проверки в песочнице

- ~~Принимает ли Notion Jira Sync скоупированный токен вместо scopeless~~ — нет, проверено Jira-админом компании (25.09.2026); штатный Atlassian service account для нативного Sync не подходит.
- Что делает Notion Jira Sync, если участник добавляет проект, недоступный сервисному аккаунту: отказ, пустая база или регистрация webhook (проверить Settings → System → WebHooks).
- Синхронизирует ли Jira Sync только задачи за последний год (утверждение стороннего гайда).
- Распространяется ли Enterprise-ограничение «Limit which connections members can install» на Jira Sync (он живёт в Settings → Import).
- Стоимость Notion Workers в credits для ~N запусков в день и объёма изменений `PRO`; доступность Workers на текущем плане.
- Для варианта B: реальное число шагов на событие и месячный объём событий `PRO` против пула организации.

## 9. Источники

**Notion**

- Connect Jira to Notion (Jira Sync, требования, права, webhooks, ограничения) — https://www.notion.com/help/jira
- Fix common Jira Sync issues — https://www.notion.com/help/common-jira-sync-issues
- Unleashing collaboration with Notion's Jira connection (роль admin- и личной авторизации) — https://www.notion.com/help/guides/unleashing-collaboration-with-notions-jira-connection
- Synced databases (планы Business/Enterprise, one-way) — https://www.notion.com/help/synced-databases
- Link previews (видимость превью, «Content not found») — https://www.notion.com/help/link-previews
- Add & manage connections; Enterprise connection settings — https://www.notion.com/help/add-and-manage-connections-with-the-api, https://www.notion.com/help/enterprise-connection-settings
- Notion AI connector for Jira — https://www.notion.com/help/jira-ai-connector
- Audit log; Data residency; Security — https://www.notion.com/help/audit-log, https://www.notion.com/help/data-residency, https://www.notion.com/security
- Pricing — https://www.notion.com/pricing; Release 2026-05-13 (Workers, connections) — https://www.notion.com/releases/2026-05-13
- Notion Workers: overview, Syncs, Secrets, Limits, Sharing — https://developers.notion.com/workers/get-started/overview, https://developers.notion.com/workers/guides/syncs, https://developers.notion.com/workers/guides/secrets, https://developers.notion.com/workers/reference/limits, https://developers.notion.com/workers/guides/sharing-workers
- Notion API: upgrade guides 2025-09-03 и 2026-03-11, request limits, query a data source, webhooks — https://developers.notion.com/docs/upgrade-guide-2025-09-03, https://developers.notion.com/docs/upgrade-guide-2026-03-11, https://developers.notion.com/reference/request-limits, https://developers.notion.com/reference/query-a-data-source, https://developers.notion.com/reference/webhooks
- Database automations «Send webhook» — https://www.notion.com/help/webhook-actions

**Atlassian**

- Manage webhooks (Administer Jira) — https://support.atlassian.com/jira-cloud-administration/docs/manage-webhooks/
- Webhooks (developer docs: admin vs dynamic, JQL, expiry) — https://developer.atlassian.com/cloud/jira/platform/webhooks/
- Permissions overview; restrict project access; read-only user — https://support.atlassian.com/jira/kb/jira-permissions-general-overview/, https://support.atlassian.com/atlassian-cloud/kb/restrict-project-access-to-certain-groups-or-users-in-jira-cloud/, https://support.atlassian.com/jira/kb/create-a-read-only-user-in-jira-cloud/
- Issue security schemes; set security level via automation — https://support.atlassian.com/jira-cloud-administration/docs/configure-issue-security-schemes/, https://support.atlassian.com/automation/kb/automation-or-set-issue-security-level-via-automation-for-jira/
- Service accounts; API tokens for service accounts — https://support.atlassian.com/user-management/docs/understand-service-accounts/, https://support.atlassian.com/user-management/docs/manage-api-tokens-for-service-accounts/
- Manage API tokens (срок 1–365 дней, scopes) — https://support.atlassian.com/atlassian-account/docs/manage-api-tokens-for-your-atlassian-account/
- Block user API token access (Guard) — https://support.atlassian.com/security-and-access-policies/docs/set-api-token-access/
- App access rules: block app access, coverage summary, OAUTH20-2493 — https://support.atlassian.com/security-and-access-policies/docs/block-app-access/, https://support.atlassian.com/security-and-access-policies/docs/app-access-rule-coverage-summary/, https://jira.atlassian.com/browse/OAUTH20-2493
- Connected apps (3LO, Block user apps) — https://support.atlassian.com/security-and-access-policies/docs/manage-your-users-third-party-apps/
- Jira automation actions (Send web request, hidden values); usage (steps); service limits; incoming webhook; restrictions — https://support.atlassian.com/cloud-automation/docs/jira-automation-actions/, https://support.atlassian.com/cloud-automation/docs/how-is-my-usage-calculated/, https://support.atlassian.com/cloud-automation/docs/automation-service-limits/, https://support.atlassian.com/cloud-automation/docs/configure-the-incoming-webhook-trigger-in-atlassian-automation/, https://support.atlassian.com/cloud-automation/docs/add-restrictions-to-automation-components/
- Smart values webhookResponse — https://support.atlassian.com/cloud-automation/docs/jira-smart-values-issues/, https://support.atlassian.com/automation/kb/getting-an-empty-value-when-using-the-webhookresponse-smart-value/
- Forge: product events, egress permissions, Runs on Atlassian, data residency, environments/variables, distribute apps, limits — https://developer.atlassian.com/platform/forge/events-reference/jira/, https://developer.atlassian.com/platform/forge/runtime-egress-permissions/, https://developer.atlassian.com/platform/forge/runs-on-atlassian/, https://developer.atlassian.com/platform/forge/data-residency/, https://developer.atlassian.com/platform/forge/environments-and-versions/, https://developer.atlassian.com/platform/forge/distribute-your-apps/, https://developer.atlassian.com/platform/forge/limits-invocation/
- Jira scopes (manage:jira-webhook и granular) — https://developer.atlassian.com/cloud/jira/platform/scopes-for-oauth-2-3LO-and-forge-apps/
- Audit log gap for webhooks — https://community.atlassian.com/forums/Jira-questions/Jira-Cloud-admin-Missing-audit-log-entry-when-creating-a-new/qaq-p/3109925
- Marketplace: Getint — https://marketplace.atlassian.com/apps/1231784/notion-integration-for-jira; Notion AI — https://marketplace.atlassian.com/apps/1235932/notion-ai

**Сторонние инструменты**

- Getint docs — https://docs.getint.io/guides/integration-synchronization/jira-notion-integration; security — https://www.getint.io/security
- Unito: rules, Jira permissions, Notion limitations, webhook support, security — https://guide.unito.io/how-to-set-up-rules, https://guide.unito.io/what-permissions-are-required-for-jira-users, https://guide.unito.io/limitations-notion-integration, https://guide.unito.io/unito-webhook-support, https://unito.io/security/
- n8n Jira credentials / Notion node / security — https://docs.n8n.io/integrations/builtin/credentials/jira/, https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.notion/, https://n8n.io/legal/security/
- Make Jira / Notion — https://apps.make.com/jira, https://apps.make.com/notion; Zapier Jira — https://zapier.com/apps/jira-software-cloud/integrations/notion; Zoho Flow — https://www.zohoflow.com/apps/jira-cloud/integrations/notion/; Power Automate Jira / Notion IP — https://learn.microsoft.com/en-us/connectors/jira/, https://learn.microsoft.com/en-us/connectors/notionip/
- Exalate supported integrations (нет Notion) — https://docs.exalate.com/docs/exalate-supported-integrations; Whalesync Jira (не доступен) — https://www.whalesync.com/connector/jira
- Сторонний обзор нативного Jira Sync (окно в один год, two-way с ноября 2025) — https://www.simonesmerilli.com/business/notion-jira-sync-native
