# План подготовки песочницы для испытаний Jira → Notion

*Версия от 24.09.2026, вариант для компании с Jira Premium/Enterprise и Notion Enterprise. Пути в интерфейсах сверены с актуальной документацией Atlassian и Notion; интерфейсы меняются, поэтому если пункт меню называется чуть иначе, ищите по смыслу. В Jira Cloud в 2026 году «Projects» местами переименованы в «Spaces», а «Issues» в «Work items»; в плане даны оба названия.*

## 0. Что получится и сколько времени

Модель: вы запрашиваете временные админские роли и делаете всё сами. От администраторов нужны три действия: выдать вам роли в Atlassian, создать воркспейс в Notion (или разрешить его создание) и выдать роли в Notion. Дальше песочницу Jira вы создаёте сами как Organization admin, а воркспейс Notion настраиваете как Workspace owner с ролями IT admin, Compliance admin и People admin. Покупать ничего не нужно: sandbox входит в план Jira Premium/Enterprise, воркспейс наследует Notion Enterprise.

К концу плана у вас будут: песочница Jira с двумя проектами (`PRO`, который можно синхронизировать, и `SEC`, который нельзя), три аккаунта Atlassian (вы как администратор, сервисный аккаунт, обычный участник), воркспейс Notion с закрытым тимспейсом, интеграцией и токенами, запущенный нативный Jira Sync и результаты ручных тестов, включая три теста, доступные только на Enterprise: two-way sync, ограничение connections и audit log. Всё остальное (проверки по API, прототип Notion Workers, правило Jira Automation, отчёт) я сделаю сам.

Ориентировочно: часть Jira 45–60 минут после получения песочницы, часть Notion 30–45 минут, ручные тесты 45 минут.

Работайте в трёх разных профилях браузера (или окнах инкогнито): «админ» (ваш рабочий аккаунт), «сервисный», «участник». Иначе сессии Atlassian и Notion перепутаются, и результат тестов будет непоказательным. Если роль участника играет коллега, ему достаточно одного профиля.

## 1. Значения, которые будете вводить

Заведите файл в менеджере паролей и заполняйте по ходу. Везде ниже используются именно эти имена.

| Что | Значение | Комментарий |
|---|---|---|
| Ваш аккаунт | рабочий корпоративный аккаунт | Временно Organization admin в Atlassian; Workspace owner + IT admin + Compliance admin + People admin в Notion |
| Сервисный аккаунт | штатный Atlassian service account `notion-sync-svc` | Создаёте сами в Atlassian Administration. Его токены только со скоупами, поэтому нативный Jira Sync сначала проверяется на скоупированном токене. Запасной вариант, если Notion его не примет: managed account с почтовым ящиком `jira-notion-svc@<домен>`, а это единственный шаг, где понадобится IT |
| Участник | второй корпоративный аккаунт или коллега | Имитирует обычного сотрудника, у которого есть доступ к секретному проекту. Та же почта должна быть в Notion, чтобы проверить identity mapping |
| URL песочницы Jira | появится после создания, вида `https://<site>-sandbox-<id>.atlassian.net` | Atlassian sandbox организации, без продовых данных |
| Проект, который синхронизируем | ключ `PRO`, название `PRO sandbox`, company-managed, шаблон Scrum | |
| Проект, который нельзя синхронизировать | ключ `SEC`, название `SEC secret`, company-managed, шаблон Kanban | |
| Группа всех людей | `all-humans` | Админ + участник. Сервисный аккаунт сюда не входит |
| Группа читателей PRO | `svc-notion-readers` | Только сервисный аккаунт |
| Группа Jira-админов для синка | `svc-notion-jira-admin` | Только сервисный аккаунт |
| Permission scheme для PRO | `PRO sandbox scheme` | |
| Permission scheme для SEC | `SEC sandbox scheme` | |
| Воркспейс Notion | `Jira sync sandbox` | Внутри Enterprise-организации; создаёт владелец организации одним действием или вы сами, если политика организации разрешает создание воркспейсов |
| Тимспейс Notion | `Jira PRO sandbox`, тип Private | |
| Страница для базы через API | `Jira PRO (API)` | Внутри тимспейса, пустая |
| Internal connection в Notion | `jira-pro-sync-sandbox` | |
| Personal access token в Notion | `ntn-cli-sandbox` | Возможности Notion API + Workers |

## 2. Часть Jira

### 2.1. Что должен выдать helpdesk

Переименуйте для себя этот шаг в «создаю sandbox сам». Нужна роль Organization admin (её выдаёт другой Organization admin: Atlassian Administration, `Directory`, `Users`, ваш профиль, `•••`, `Assign organization admin role`).

1. Atlassian Administration (`https://admin.atlassian.com`), `Apps`, `Sandboxes`, `Create sandbox`. Выберите Jira, вариант «в новом sandbox-сайте», **не** отмечайте `Copy production data`. Дождитесь готовности, запишите URL вида `https://<site>-sandbox-<id>.atlassian.net`.
2. Доступ к песочнице по умолчанию есть только у Organization admin. Остальным его выдают так: `Apps`, `Atlassian apps`, у sandbox `•••`, `Manage users`, добавить людей в группы песочницы. Группы песочницы называются как продовые с суффиксом `-sandbox-<id>`, например `jira-users-<site>-sandbox-<id>`.
3. Сделайте себя Jira-админом песочницы: добавьте себя в группу `jira-admins-<site>-sandbox-<id>` (или выдайте себе роль App admin для Jira песочницы: `Directory`, `Users`, ваш профиль, `Grant access`, роль для Jira Administration соответствующего сайта).
4. Проверка: URL песочницы открывается, шестерёнка `Settings` показывает `System` и `Work items` (старое название `Issues`), в нём есть `Permission schemes`. Sandbox наследует план организации, поэтому схемы прав доступны.
5. Четыре настройки политик, которые вы теперь делаете сами:
   - API-токены. `Security`, `Authentication policies`: найдите политику, под которую попадают ваш аккаунт и участник, и убедитесь, что `User API tokens` = allow. Если там блокировка, создайте отдельную политику `Sandbox testing` с разрешением API-токенов и переместите в неё только участника и себя на время тестов, остальные параметры (SSO, 2FA) скопируйте из основной политики. Штатные service accounts управляются отдельно и под эту политику не попадают.
   - OAuth-приложение Notion. `Apps`, выберите sandbox-сайт, `Connected apps`: убедитесь, что для песочницы не включён `Block user apps`, либо заранее одобрите приложение Notion, когда оно появится в списке после первой авторизации участника.
   - Data security policies. `Security`, `Data security policies`: проверьте, что ни одна политика с app access rule не покрывает sandbox-сайт.
   - Allowlist доменов для Automation (только если он включён на Enterprise): в песочнице шестерёнка `Settings`, `System`, `Automation`, настройки ограничений компонентов, добавьте `api.notion.com` для `Send web request`.

### 2.2. Проверка того, что песочница пустая

В песочнице не должно быть продовых проектов и задач. Откройте `Projects` (`Spaces`), убедитесь, что список пуст или содержит только служебные проекты песочницы. Если helpdesk всё же скопировал данные, попросите пересоздать sandbox без копирования: тесты должны идти только на синтетике, иначе теряется смысл изоляции.

### 2.3. Аккаунты

1. Сервисный аккаунт создайте как штатный Atlassian service account: `Directory`, `Service accounts`, `Create a service account`. Имя `notion-sync-svc`, описание `Jira to Notion sync test`. Выдайте ему роль пользователя Jira в sandbox-сайте (при создании выбираются роли приложений). Пароля у него нет, в интерфейс он не входит, это нормально.
2. Участник: коллега или ваш второй корпоративный аккаунт. Выдайте ему доступ к песочнице: `Apps`, `Atlassian apps`, sandbox, `•••`, `Manage users`, группа `jira-users-<site>-sandbox-<id>`.
3. Проверка: `Directory`, `Users` показывает вас и участника с доступом к Jira песочницы; `Directory`, `Service accounts` показывает `notion-sync-svc` с ролью в песочнице.

### 2.4. Группы

1. `Directory`, затем `Groups`, кнопка `Create group`.
2. Создайте `all-humans`, добавьте в неё себя и участника. Сервисный аккаунт не добавляйте.
3. Создайте `svc-notion-readers` и добавьте только `notion-sync-svc` (для service account: `Directory`, `Service accounts`, аккаунт, `•••`, `Add to group`).
4. Создайте `svc-notion-jira-admin` и добавьте только `notion-sync-svc`.
5. Ничего не меняйте в группах, созданных автоматически. Убедитесь, что сервисного аккаунта нет в `jira-admins-...`, `administrators`, `site-admins`, `org-admins` и в группах, которые дают Browse в продовых схемах, если такие группы есть и в песочнице.

### 2.5. Два проекта

1. В Jira сверху `Projects` (или `Spaces`), затем `Create project`.
2. Шаблон `Scrum` в категории Software development, `Use template`. На вопрос о типе выберите `company-managed` (не team-managed, у них другая модель прав). Название `PRO sandbox`, ключ `PRO`. Создать.
3. Повторите с шаблоном `Kanban`: название `SEC secret`, ключ `SEC`, company-managed.
4. Проверка: оба проекта видны в списке проектов, у каждого в `Project settings` есть пункт `Permissions`.

### 2.6. Тестовые задачи

В `PRO` создайте задачи с такими заголовками, чтобы потом легко проверять, что именно дошло до Notion:

- Epic: `PRO Epic Alpha`, `PRO Epic Beta`.
- Story: `PRO Story 1`, `PRO Story 2`, `PRO Story 3`; первые две привяжите к `PRO Epic Alpha` через поле Parent.
- Task: `PRO Task 1`, `PRO Task 2`.
- Назначьте `PRO Story 1` на участника, `PRO Story 2` на администратора.
- К `PRO Story 1` добавьте комментарий `Test comment 1` и любое маленькое вложение (картинка до 1 МБ).
- Переведите `PRO Story 3` в статус Done.

В `SEC` создайте три задачи `SECRET 1`, `SECRET 2`, `SECRET 3`. Одну назначьте на участника.

### 2.7. Permission schemes

Цель: в `PRO` задачи видят люди и сервисный аккаунт, в `SEC` только люди.

1. Шестерёнка `Settings`, затем `Work items` (`Issues`), в левом меню внизу `Permission schemes`.
2. У схемы `Default software scheme` (или как называется схема, привязанная к вашим проектам) нажмите `Copy`. Переименуйте копию в `PRO sandbox scheme` (кнопка `Edit` у схемы).
3. Откройте `Permissions` у `PRO sandbox scheme`. Найдите строку `Browse Projects` (в новом интерфейсе `Browse spaces`).
4. Удалите из неё грант `Any logged in user` (или `Application access`), если он есть: кнопка `Remove` справа от гранта.
5. Нажмите `Edit` в этой строке, выберите `Group`, укажите `all-humans`, `Grant`. Повторите и добавьте группу `svc-notion-readers`. Оставьте также `Project role: Administrators`, если он там был.
6. Скопируйте `PRO sandbox scheme` ещё раз, назовите копию `SEC sandbox scheme`, откройте её `Permissions`, в строке `Browse Projects` удалите `svc-notion-readers`. Должны остаться `all-humans` и роль Administrators.
7. Привяжите схемы к проектам: откройте проект `PRO`, `Project settings`, `Permissions`, в правом верхнем углу `Actions` (или `...`), пункт `Use a different scheme`, выберите `PRO sandbox scheme`, `Associate`. Для `SEC` выберите `SEC sandbox scheme`.
8. Проверка через Permission helper: шестерёнка `Settings`, `System`, слева `Admin helper` (`Permission helper`). Пользователь: сервисный аккаунт, задача `SEC-1`, право `Browse Projects`. Ожидаемый ответ: права нет. Повторите для `PRO-1`: право есть. Сделайте скриншоты обоих результатов.

### 2.8. Право Administer Jira сервисному аккаунту

Это право нужно только для нативного Jira Sync (создание webhooks). Для Workers и Automation оно не нужно, и после тестов нативного Sync его можно снять.

1. Шестерёнка `Settings`, `System`, слева `Global permissions`.
2. В строке `Administer Jira` (или в форме `Grant permission` внизу страницы) выберите право `Administer Jira`, группу `svc-notion-jira-admin`, нажмите `Add` (`Grant`).
3. Проверка: в профиле «сервисный» откройте Jira: шестерёнка `Settings` должна показывать разделы администрирования, но проект `SEC` в поиске по задачам не должен возвращать ни одной задачи. В строке поиска задач введите JQL `project = SEC`, ожидается «No work items were found».

### 2.9. API-токены

Токен сервисного аккаунта создаётся в Atlassian Administration, остальные токены каждый владелец аккаунта создаёт сам.

Сервисный аккаунт (вы как Organization admin):

1. `Directory`, `Service accounts`, `notion-sync-svc`, `Create credentials`, `API token`, `Next`. Название `notion-sync-sandbox`, срок 90 дней. Скоупы: `read:jira-work`, `read:jira-user`, а также `read:webhook:jira`, `write:webhook:jira`, `delete:webhook:jira` (если в списке гранулярных скоупов их нет, возьмите классический `manage:jira-webhook`). Для теста two-way позже понадобится ещё `write:jira-work`, лучше добавить сразу. Скопируйте значение, потом его не покажут. Это `JIRA_SVC_TOKEN_SCOPED`.
2. Запасной вариант, только если Notion откажется работать со скоупированным токеном (см. 3.7): попросите IT создать почтовый ящик `jira-notion-svc@<домен>`, заведите на него managed account, дайте ему тот же набор групп и создайте обычный токен без скоупов на `https://id.atlassian.com/manage-profile/security/api-tokens`. Это `JIRA_SVC_TOKEN`.

В профиле «админ» (ваш аккаунт):

3. `https://id.atlassian.com/manage-profile/security/api-tokens`, `Create API token` без скоупов, название `claude-sandbox-admin`, срок 30 дней. Это `JIRA_ADMIN_TOKEN`. Он нужен мне, чтобы читать список webhooks, проверять схемы прав и создавать тестовые задачи по API.

В профиле «участник» (необязательно, но полезно):

4. `Create API token` без скоупов, название `member-sandbox`, срок 30 дней. Это `JIRA_MEMBER_TOKEN`, чтобы я мог сравнить видимость проектов у участника и у сервисного аккаунта по API.

### 2.10. Контрольная точка A

Передайте мне (раздел 5) URL песочницы и токены. Я по API подтвержу, что сервисный аккаунт видит ровно `PRO`, участник видит оба проекта, и что у сервисного аккаунта есть право администрирования. Пока я не подтвердил, к Notion переходить можно, но настройку Jira Sync (2.8 в части Notion) лучше отложить, чтобы не переделывать.

## 3. Часть Notion

### 3.1. Воркспейс внутри Enterprise-организации

Нужно от владельца организации Notion, одним заходом: создать воркспейс `Jira sync sandbox` внутри организации, назначить вас Workspace owner, выдать вам роли `IT admin`, `Compliance admin` и `People admin` на время тестов (organization settings, `People`, `Manage admin roles`) и выделить воркспейсу небольшой лимит Notion credits, если организация на multi-workspace контракте (Workers работают на credits). Если политика организации разрешает участникам создавать воркспейсы, создайте его сами: переключатель воркспейсов, `Join or create workspace`, `Create a workspace`; на верифицированном домене он попадёт в организацию.

Получив доступ, проверьте:

1. В переключателе воркспейсов есть `Jira sync sandbox`, в `Settings`, `People` вы указаны как Workspace owner.
2. `Settings`, `Billing` показывает план Enterprise (наследуется от организации).
3. `Settings`, `Import` содержит пункт `Jira Sync`, а `Settings`, `Connections` открывается и содержит вкладку `Manage`.
4. Переключатель воркспейсов, `Manage organization` открывает консоль организации: с ролью IT admin вам доступны настройки интеграций и безопасности, с ролью Compliance admin вкладка `Data & compliance` с audit log, с ролью People admin управление участниками и группами. Если централизованные настройки организации ограничивают connections или Import, снимите ограничение только для этого воркспейса или заранее одобрите Jira Sync и internal connections.

### 3.2. Что проверить у владельца организации

Теперь это ваши собственные проверки:

1. Notion Workers доступны в воркспейсе (beta, расходуют credits): в разделе разработчика есть страница Workers, а при создании personal access token предлагается возможность `Workers`. Если её нет, посмотрите в консоли организации лимит credits для воркспейса и включите on-demand spend или задайте лимит.
2. Политика создания PAT: на Enterprise по умолчанию «Workspace owners and selected groups», вам как owner этого достаточно.
3. Audit log: `Manage organization`, `Data & compliance`, `Audit log`, фильтр по воркспейсу `Jira sync sandbox`. Убедитесь, что события отображаются; выгрузку сделаете сами в конце (тест T7).

### 3.3. Участник

1. `Settings`, `People`, вкладка `Members`, `Add members`, введите корпоративную почту участника (ту же, что в Jira), роль `Member`, `Invite`. Если участники провижинятся через SCIM, добавьте участника в нужную группу провайдера идентичности или сделайте это через консоль организации как People admin.
2. В профиле «участник» примите приглашение и войдите в воркспейс.
3. Проверка: в списке Members два человека, у участника роль Member, не Guest и не restricted member (иначе он не сможет пройти авторизацию Jira и создать PAT).

### 3.4. Закрытый тимспейс и страница

1. В сайдбаре найдите заголовок `Teamspaces`, нажмите `+` справа от него.
2. Название `Jira PRO sandbox`, доступ `Private` (виден только добавленным), `Create`.
3. Откройте настройки тимспейса (`...` рядом с названием, `Teamspace settings`, вкладка `Members`), добавьте участника как Member.
4. Внутри тимспейса создайте пустую страницу `Jira PRO (API)`. Скопируйте её ссылку (`Share`, `Copy link`). Это `NOTION_API_PAGE_URL`, значение не секретное.

### 3.5. Режим разработчика, интеграция и токен для CLI

1. `Settings`, `Connections`, включите переключатель `Developer Mode`. В сайдбаре появится раздел разработчика.
2. В этом разделе (или по ссылке `https://www.notion.so/developers`) нажмите `+ New connection`. Название `jira-pro-sync-sandbox`, воркспейс `Jira sync sandbox`, тип Internal. В возможностях (`Capabilities`) отметьте `Read content`, `Update content`, `Insert content`, а в блоке про пользователей `Read user information including email addresses`. Сохраните.
3. На странице интеграции скопируйте `Internal Integration Secret`. Это `NOTION_TOKEN`.
4. Подключите интеграцию к странице: откройте `Jira PRO (API)`, `...` в правом верхнем углу, `Connections`, `Add connection`, выберите `jira-pro-sync-sandbox`, подтвердите. Это единственная страница, к которой у интеграции будет доступ, базу внутри создам я.
5. Personal access token: в разделе разработчика откройте `Personal access tokens` (`https://www.notion.so/developers/tokens`), `New token`, название `ntn-cli-sandbox`, воркспейс `Jira sync sandbox`, возможности `Notion API` и `Workers`. Скопируйте значение. Это `NOTION_PAT_WORKERS`. Если пункта `Workers` в списке нет, значит Workers в этом воркспейсе пока не включены; сообщите мне, я предложу вариант входа через `ntn login --no-browser`, где вы подтверждаете код в своём браузере, а токены мне не передаются.

### 3.6. Контрольная точка B

Передайте мне `NOTION_TOKEN`, `NOTION_PAT_WORKERS` (раздел 5) и ссылку на страницу `Jira PRO (API)` в чат. Я создам базу для варианта «Automation» и подготовлю JSON правила, а также задеплою Worker. Параллельно вы делаете 3.7 и тесты.

### 3.7. Нативный Jira Sync

Основной вариант: скоупированный токен штатного service account. Если он сработает, это лучший результат для безопасности и никакого почтового ящика не нужно.

1. В профиле «админ» в Notion: `Settings`, `Import`, найдите `Jira Sync`, нажмите `Get started`.
2. Введите: e-mail сервисного аккаунта (адрес вида `...@serviceaccount.atlassian.com` из карточки `notion-sync-svc` в `Directory`, `Service accounts`), URL песочницы Jira, токен `JIRA_SVC_TOKEN_SCOPED`. `Next`.
3. Если появится ошибка или статус `Sync failed`, сделайте скриншот и сообщите мне: я проверю по API, какой именно запрос Notion не проходит. Только после этого включайте запасной вариант из 2.9 (managed account с почтовым ящиком и токен без скоупов): `Settings`, `Import`, `...` рядом с Jira, `Remove`, и повторите шаги 1–2 с `JIRA_SVC_TOKEN`.
4. Выберите `Create a new sync`. Запишите, какие проекты показаны в списке выбора: только `PRO` или также `SEC`. Сделайте скриншот списка. Отметьте только `PRO`.
5. На шаге `Select properties to sync` отметьте всё. В качестве места размещения выберите тимспейс `Jira PRO sandbox`.
6. Дождитесь появления баз проектов и задач. Запишите время от нажатия до появления всех 7 задач `PRO`. Проверьте, что в базе задач есть комментарий и вложение у `PRO Story 1`.
7. Сообщите мне, что синк создан. Я по API сниму список webhooks, которые Notion зарегистрировал в Jira, и их JQL-фильтры. Вы можете посмотреть их сами: в Jira шестерёнка `Settings`, `System`, слева `WebHooks`. Ничего там не удаляйте и не меняйте.

### 3.8. Тест T1, главный: участник пытается добавить секретный проект

1. В профиле «участник» войдите в Notion, откройте тимспейс `Jira PRO sandbox`, откройте синхронизированную базу задач.
2. Нажмите `...` над базой, `Source`. Notion попросит подключить ваш Jira-аккаунт: пройдите авторизацию под аккаунтом участника (`+member`), у которого есть доступ к `SEC`.
3. Попробуйте добавить проект `SEC` к синхронизации. Зафиксируйте результат скриншотами: проект отсутствует в списке, ошибка при добавлении, или проект добавился.
4. Если проект добавился, подождите 10 минут и проверьте, появились ли задачи `SECRET 1..3` в базе. Ничего не удаляйте, сообщите мне, я проверю по API, что именно попало в Notion и какие webhooks появились в Jira.
5. Только после моей проверки удалите `SEC` из синка (`...`, `Source`, снять отметку) и удалите появившиеся строки.

Ожидаемый и желаемый результат: `SEC` нельзя добавить или база остаётся пустой. Если данные `SECRET` появились, это ключевой аргумент против нативного варианта, и тест сам по себе окупает всю песочницу.

### 3.9. Тест T2: отзыв токена

В самом конце всех испытаний, когда я скажу, что закончил проверки по API:

1. В профиле «сервисный» откройте `https://id.atlassian.com/manage-profile/security/api-tokens` и отзовите токен `notion-sync-sandbox` (`Revoke`).
2. Измените любую задачу в `PRO` и через 15 минут посмотрите на бейдж над базой в Notion. Ожидается `Sync stopped` и предложение `Re-authenticate`. Скриншот.

### 3.10. Тест T3: правило Jira Automation (после контрольной точки B)

Я пришлю файл JSON с правилом. Ваши действия:

1. В Jira шестерёнка `Settings`, `System`, слева `Automation flows` (`Automation rules`). Справа сверху `...`, `Import flows`, `Upload JSON`, выберите файл, укажите область действия: только проект `PRO`. Импортировать.
2. Откройте импортированное правило. В каждом шаге `Send web request` в заголовке `Authorization` вставьте значение `Bearer <NOTION_TOKEN>` и отметьте `Hidden`; скрытые значения при импорте не переносятся, это нормально. Сохраните и включите правило (`Turn on`).
3. Создайте в `PRO` новую Story `PRO Story Automation`, затем поменяйте у неё статус. Через минуту проверьте базу на странице `Jira PRO (API)`: должна появиться и обновиться одна строка. Сообщите мне, дальше я проверю аудит выполнения и посчитаю шаги.

### 3.11. Тест T4: Notion Workers

Делаю я. От вас нужно только: если при первом деплое Notion попросит включить Workers для воркспейса или принять условия beta, сделайте это в `https://www.notion.so/developers/workers`. После деплоя в тимспейсе появится база `Jira PRO (Workers)`; посмотрите, что колонки read-only, а строки нельзя добавить вручную, и пришлите скриншот.

### 3.12. Тест T5: two-way sync (только Enterprise)

1. В профиле «админ» над синхронизированной базой задач нажмите бейдж `Synced`, включите `2-way sync`, пройдите повторную авторизацию Jira под своим аккаунтом.
2. В профиле «участник» откройте ту же базу, пройдите авторизацию под аккаунтом участника, измените статус у `PRO Story 2` и добавьте комментарий `From Notion by member`.
3. В Jira откройте `PRO Story 2`: проверьте, что статус изменился, и запишите, от чьего имени записан комментарий и переход (участник или сервисный аккаунт). Скриншот истории задачи. Это ответ на вопрос, соблюдаются ли права Jira при редактировании из Notion.

### 3.13. Тест T6: ограничение connections (только Enterprise)

1. В профиле «админ»: `Settings`, `Connections`, вкладка `Manage`, параметр `Limit which connections members can install` переведите в `Approved only`, список approved оставьте пустым.
2. В профиле «участник» попробуйте: (а) создать новый Jira Sync в тимспейсе (`Settings`, `Import`, `Jira Sync`); (б) добавить проект к существующему синку через `...`, `Source`; (в) создать internal connection в разделе разработчика.
3. Зафиксируйте, что из этого заблокировалось. Так мы узнаем, распространяется ли ограничение на Jira Sync, который живёт в `Import`, а не в `Connections`. После теста верните `No restrictions`.

### 3.14. Тест T7: audit log (только Enterprise)

По окончании тестов: `Manage organization`, `Data & compliance`, `Audit log`, фильтр по воркспейсу `Jira sync sandbox` и периоду тестов, экспорт. Пришлите файл мне. Я сверю, какие из наших действий (настройка Jira Sync, добавление проекта участником, подключение интеграции, PAT, Worker) там видны, а какие нет. Отдельно посмотрите audit log песочницы Jira (`Settings`, `System`, `Audit log`): там должны быть изменения permission schemes, а создания webhooks, по опыту сообщества, там нет.

## 4. Порядок и контрольные точки

1. Раздел 2 целиком (Jira), затем контрольная точка A: я проверяю права по API.
2. Разделы 3.1–3.5 (Notion), затем контрольная точка B: я создаю базу, правило и Worker.
3. Раздел 3.7 и тест T1 вами, параллельно я снимаю webhooks и данные по API.
4. Тесты T3, T4, затем T5 и T6.
5. Тест T2 последним, потому что он ломает нативный синк.
6. Выгрузка audit log своими руками (T7).
7. Итог: я собираю отчёт с результатами и дополняю документ исследования.

## 5. Как передать мне доступы

Токены в чат не присылайте. Положите их в настройки облачного окружения этой сессии: меню окружения в заголовке сессии, затем `Edit`, раздел `API credentials`, если он есть, иначе `Environment variables`. Новая сессия подхватит значения; после сохранения напишите мне «переменные добавлены», и я продолжу в новой сессии.

| Переменная | Значение |
|---|---|
| `JIRA_SITE_URL` | URL песочницы, вида `https://<site>-sandbox-<id>.atlassian.net` |
| `JIRA_ADMIN_EMAIL` | ваша рабочая почта |
| `JIRA_ADMIN_TOKEN` | токен из 2.9, шаг 3 |
| `JIRA_SVC_EMAIL` | e-mail service account вида `...@serviceaccount.atlassian.com` (или `jira-notion-svc@<домен>` для запасного варианта) |
| `JIRA_SVC_TOKEN_SCOPED` | токен со скоупами из 2.9, шаг 1 |
| `JIRA_SVC_TOKEN` | токен без скоупов из 2.9, шаг 2, только если понадобился запасной вариант |
| `JIRA_MEMBER_EMAIL` | почта участника |
| `JIRA_MEMBER_TOKEN` | токен из 2.9, шаг 4, если делали |
| `NOTION_TOKEN` | Internal Integration Secret из 3.5, шаг 3 |
| `NOTION_PAT_WORKERS` | Personal access token из 3.5, шаг 5 |

В чат достаточно прислать: URL сайта Jira, ссылку на страницу `Jira PRO (API)`, и скриншоты из пунктов 2.7 (Permission helper), 3.7 (список проектов при настройке синка) и 3.8 (результат теста T1).

## 6. Чек-лист перед тем, как позвать меня на контрольную точку A

- [ ] Роли получены: Organization admin в Atlassian; Workspace owner + IT admin + Compliance admin + People admin в Notion.
- [ ] Sandbox Jira создан без копирования данных, вы в нём Jira-админ (в Settings, Work items есть Permission schemes).
- [ ] Четыре настройки политик проверены вами (API-токены, приложение Notion в Connected apps, data security policies, allowlist для Automation).
- [ ] Три аккаунта имеют доступ к песочнице: вы, service account `notion-sync-svc`, участник.
- [ ] Группы `all-humans` (админ, участник), `svc-notion-readers` (сервисный), `svc-notion-jira-admin` (сервисный).
- [ ] Проекты `PRO` и `SEC`, оба company-managed, с тестовыми задачами.
- [ ] `PRO sandbox scheme` и `SEC sandbox scheme` привязаны к своим проектам, грант `Any logged in user` из `Browse Projects` убран.
- [ ] Permission helper: сервисный аккаунт не видит `SEC-1`, видит `PRO-1`.
- [ ] `Administer Jira` выдано группе `svc-notion-jira-admin`.
- [ ] Три токена созданы и сохранены в менеджере паролей.
- [ ] Переменные добавлены в окружение.

## 7. Что делать после испытаний

Всё своими руками: отозвать токены (свой и участника на `id.atlassian.com`, токен service account в `Directory`, `Service accounts`), удалить service account, удалить Worker, интеграцию, PAT и синки в Notion, деактивировать sandbox (`Apps`, `Sandboxes`, `Deactivate`), удалить воркспейс `Jira sync sandbox` (`Settings`, `General`, удаление воркспейса), вернуть политики в исходное состояние (authentication policy `Sandbox testing`, Connected apps) и попросить снять с вас временные роли Organization admin, IT admin, Compliance admin и People admin.

