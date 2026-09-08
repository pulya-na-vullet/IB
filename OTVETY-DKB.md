# Ответы по задачам ДКБ (эпик CSDINPROJT-25796)

Неформальный текст для вставки в Jira / чат команды: [KOMMENTARIJ-DLYA-KOMANDY.md](./KOMMENTARIJ-DLYA-KOMANDY.md).

**Проект:** PJ07253-MMB23 / ADRBT-20339 — Страхование в SFA/SFACall с AlfaCapture 1.0  
**PPM:** PJ07253-MMB23  
**Vision:** https://rsm/visions/vision-details/6a509a36a4ce07381201a916  
**Продуктовый контур в Jira NCINS:** НФС.Некредитное страхование (каналы SFA и НИБ / Альфа-Бизнес)

**Контакты команды (для pentest / комментариев ДКБ):**

| Роль | ФИО |
|---|---|
| Product Owner | Ермолов Александр |
| Dev Teamlead | Яковлев Максим |
| QA Teamlead | Григорьев Дмитрий Вячеславович |
| Аналитик (SFA, из NCINS-164) | Самсонова Анастасия |
| Аналитик (НИБ, из NCINS-200/308) | Забарянский Юрий Геннадьевич |

Почта / rocket для этих ФИО в материалах нет — в комментарий Jira pentest ФИО уже можно класть, корпоративные адреса ДКБ обычно сами резолвят по ФИО.

Ниже — что можно закрыть или черновик комментария в Jira **уже сейчас**, и что команде нужно дописать. Источники: задачи NCINS-164/175/200/277/291/305/306/308, код `ump-ncins-pa`, стендовые URL из комментариев разработки.

Секреты (пароли тестовых УЗ, креды AC 1.0) в этот файл **не выносятся**. Они уже есть в комментариях NCINS-164 / NCINS-277 / NCINS-291 — для pentest их передавать в Jira ДКБ, не в git.

---

## Кратко

| Задача | Вывод по имеющимся данным | Можно ли закрыть / перевести |
|---|---|---|
| CSDINPROJT-25796 Epic | Это рамка, не СЗИ. Работы = дочерние задачи. | Нет. Нужен план/бюджет по дочерним. |
| CSDINPROJT-25797 Datapower | Новый внешний API «банк ↔ партнёр/интернет» проект **не поднимает**. Вызовы идут во внутренние системы банка через уже существующие шлюзы UFR / corp-gateway / mks-gateway. | Черновик комментария готов. Закрытие — только после подтверждения СА, что новых Datapower-сервисов нет. |
| CSDINPROJT-25798 CSP | Свой k8s-кластер **не создаём**. Раскатка в **существующие** кластеры UMP и UFR. | Черновик готов. Нужно подтверждение платформы, что CSP на этих кластерах уже включён. |
| CSDINPROJT-25800 SIEM | Новое ПО/интеграции есть → задача **актуальна**. Логирование в коде включено, в SIEM факт доставки **не подтверждён**. | Не закрывать. Нужны app_id, заявка 04.02.42, примеры событий. |
| CSDINPROJT-25801 Pentest | API и UI **создаются/меняются** → pentest **нужен**, не CANCELED. Контакты PO / Dev TL / QA TL есть. | В BACKLOG нельзя: нет bitbucket и нет 2 УЗ на роль. Черновик обязательного блока — ниже. |
| CSDINPROJT-25802 WAF | Свои URL — `*.moscow.alfaintra.net` (доверенная сеть). Интернет-фронт подписания — **существующий SignOnline**, не новый сервис проекта. Новый клиентский путь в Альфа-Бизнесе — зона риска. | Черновик готов. Нужно явное решение СА: SFA-only vs публикация в НИБ. |

---

## Что уже известно по архитектуре (общее для всех задач)

### Системы из поля Product Comment эпика

- WSCodeBusinessEvent (регистрация бизнес-событий)
- Альфа-Бизнес
- AlfaCapture 1.0 (`systemCodeAC=NIBNCINS`)
- WSCustomerSearchClients / WSCustomerAddressList / WSCustomerPersonalInfo
- SignOnline (канал `SFA`, продукт `non_credit_ins`)
- UMP (процессы `ump-*-ncins-pa`)
- SFA ЮЛ
- Электронный архив IBM Content Manager (CMS-атрибуты в DTO подписи: `SME_NotCredSt`)

### Что делает команда (не «новый банк в интернете»)

1. **SFA / ЕФ.Цифровое Страхование** — UI-модуль сотрудников  
   `ufr-eos-ul-ncins-ui`  
   Авторизация: пользовательский Keycloak (корпоративный AD).  
   Сессия: service panel + Redis (`ncins:managerInfo::{cus}`, `ncins:signInfo::{cus}`, TTL 1 час).

2. **Middle SFA** — `ufr-eos-ul-ncins-core-api`  
   Новые методы:  
   - `POST /v1/doc/generate-form`  
   - `POST /v1/doc/download`  
   - `POST /v1/doc/download-signed`  
   - `POST /v1/sign/create-operation`  
   Дальше: OBIP (шаблон AGREEMENT) → SignModule / SignOnline → AlfaCapture.

3. **UMP** — воркеры `ump-ncins-pa` в **существующем** k8s UMP  
   DNS: `*.ump.svc.cluster.local`  
   Кластер (dev): `umpdevwk8sm1.moscow.alfaintra.net`  
   Процессы: prepare / signing / payment / finalisation / delete documents.  
   Документы в AlfaCapture через типовой `ump-generate-and-save-document-pa`.  
   Договор в учёт: `POST /v1/ins-contracts` через corp-gateway.

4. **НИБ / Альфа-Бизнес** (соседний канал той же продуктовой линии)  
   `POST .../v1/ins-premium/calculate`  
   `http://corp-gateway-test.moscow.alfaintra.net/corp-ncins-acc-gateway/secure/corp-ncins-acc-corp-ncins-acc-api/...`  
   Токен: `mks-gateway/.../realms/corporate/.../token`, client-id `ump-ncins`.

### Стенды, которые уже фигурируют в задачах

| Контур | URL |
|---|---|
| UI DEV NCINS | `https://dev.ufrulkint-ui.moscow.alfaintra.net/ufr-eos-ul-ncins-ui/` |
| Service panel INT | `https://int.ufrulkint-ui.moscow.alfaintra.net/ufr-eos-ul-service-panel-gate-ui/` |
| API INT | `https://int.ufrulkint-api.moscow.alfaintra.net/ufr-eos-ul-ncins-core-api` |
| API DEV | `https://dev.ufrulkint-api.moscow.alfaintra.net/ufr-eos-ul-ncins-core-api` |
| Шлюз НИБ TEST | `http://corp-gateway-test.moscow.alfaintra.net/corp-ncins-acc-gateway/` |
| Download из AC | `.../ufr-eos-ul-document-processing-gateway-api/.../api/download-document` |

Все перечисленные хосты — зона `moscow.alfaintra.net` (интранет банка), не публичный интернет.

---

## CSDINPROJT-25796 — Epic ДКБ

Это не мера СЗИ, а контейнер. По инструкции ДКБ: включить дочерние меры в план и бюджет, проработать в Vision/ОТАР/Паспорте.

**Что можно ответить в эпике**

> Ознакомились. Реализация мер ИБ ведётся по дочерним задачам 25797–25802. Краткий статус: Datapower и WAF — предварительно не требуются для новых точек публикации проекта (SFA/UFR/UMP, внутренние шлюзы); CSP — раскатка в существующие кластеры UMP/UFR, ждём подтверждение платформы; SIEM и pentest — актуальны, pentest не CANCELED (есть новые API и UI). Детали — в комментариях дочерних задач.

**Чего нет:** утверждённый ОТАР/Паспорт с архитектурой СЗИ; актуальный РП (в Jira РП Inactive); оценка бюджета, если CSP/WAF всё же потребуются.

---

## CSDINPROJT-25797 — Gateway IBM Datapower

### Критерий из задачи

Datapower нужен как API-шлюз **между системами банка и системами партнёров / внешними сервисами / сайтами**.

### Вывод

По коду и стендам проект **не публикует новый API наружу** и **не интегрируется с внешней системой партнёра-страховщика напрямую**.

Цепочки:

- Сотрудник SFA → `ufr-eos-ul-ncins-ui` / `ufr-eos-ul-ncins-core-api` (платформа ЕФ ЮЛ, `*.alfaintra.net`)
- Middle → OBIP, SignModule/SignOnline, AlfaCapture, document-processing-gateway — **внутренние** системы банка
- UMP `ump-ncins-pa` → `ump-application-facade` / `ump-products` по cluster DNS
- UMP → учёт договоров НИБ через **уже существующий** `corp-gateway` + `mks-gateway` (не новый инстанс Datapower команды)

Клиент подписывает документы на **существующем** фронте SignOnline (канал SFA). Это не новый партнёрский API.

Отдельный Datapower «с нуля» команде проекта, судя по имеющимся артефактам, **не нужен**. Если corp-gateway юридически и есть Datapower — это **переиспользование уже подключённой платформы**, заявка 02.02.01.05 на новый сервис не следует из кода.

### Черновик комментария в Jira

```text
@Федорова Мария Алексеевна

По критерию необходимости Datapower: отдельный новый API-шлюз банк↔партнёр/интернет в рамках PJ07253-MMB23 не создаём.

Контур:
- SFA UI/API: ufr-eos-ul-ncins-ui, ufr-eos-ul-ncins-core-api (хосты *.moscow.alfaintra.net, авторизация Keycloak сотрудника).
- Документы: AlfaCapture 1.0, OBIP, SignOnline/SignModule, IBM CM (CMS в DTO подписи).
- Оркестрация: ump-ncins-pa в существующем k8s UMP.
- Учёт НИБ: вызов через уже существующий corp-gateway / mks-gateway
  (corp-ncins-acc-gateway/secure/corp-ncins-acc-corp-ncins-acc-api, в т.ч. POST /v1/ins-contracts, POST /v1/ins-premium/calculate).

Внешнего API партнёра и новой публикации в Интернет у сервисов команды нет.
Просим подтвердить, что задача не требует заявки 02.02.01.05 и может быть закрыта как «покрыто существующими шлюзами UFR/corp-gateway».
Если corp-gateway = Datapower и новые операции на шлюзе всё же нужно зарегистрировать — напишите, какие сервисы/операции включить в заявку.
```

### Чего нет (без этого ДКБ может не закрыть)

- Подтверждение СА/архитектора шлюзов: `corp-gateway` = IBM Datapower или нет.
- Список **новых** операций на шлюзе vs уже существующих.
- ОТАР (в задаче ссылка битая: «здесь ссылка»).
- Решение, нужно ли регистрировать `POST /v1/ins-contracts` и `ins-premium/calculate` как новые Datapower-сервисы.

---

## CSDINPROJT-25798 — Microservices CSP

### Критерий из задачи

Актуально при **создании инстанса Kubernetes**. Если раскатка в существующий кластер, но неизвестно, есть ли CSP — тоже актуально.

### Вывод

Свой кластер k8s проект **не создаёт**.

- `ump-ncins-pa` ходит в `ump-config-server.ump.svc.cluster.local`, `ump-application-facade.ump.svc.cluster.local` — это namespace/кластер **UMP**.
- Camunda Operate: `operate.umpdevwk8sm1.moscow.alfaintra.net`.
- SFA UI/API живут на платформе UFR ЕФ ЮЛ (`ufrulkint-*`) — тоже существующая платформа, не greenfield-кластер.
- В NCINS-200 есть заявка TESTDESK-15985 «изменение namespace» — это смена namespace в **уже существующем** кластере, не новый инстанс.

CSP как «подключить с нуля» команде, скорее всего, **не нужно**, если платформы UMP и UFR уже под CSP. Задачу нельзя закрыть без письменного подтверждения владельцев кластеров.

### Черновик комментария в Jira

```text
@Федорова Мария Алексеевна

Новый инстанс Kubernetes не создаём. Раскатываемся в существующие кластеры:
- UMP (ump-ncins-pa, DNS *.ump.svc.cluster.local, dev-кластер umpdevwk8sm1);
- UFR ЕФ ЮЛ (ufr-eos-ul-ncins-ui, ufr-eos-ul-ncins-core-api).
Есть заявка на смену namespace: TESTDESK-15985.

Просим подтвердить у владельцев платформ UMP и UFR, что CSP (аномалии доступа + сканирование Docker-образов) на этих кластерах уже включён.
Если да — просим закрыть задачу как покрытую платформой.
Если нет — нужны: инструкция подключения, ответственный сопровождения CSP, перечень образов (ump-ncins-pa, ufr-eos-ul-ncins-*).
```

### Чего нет

- Имена прод-кластеров / namespace UFR и UMP (кроме dev `umpdevwk8sm1` и факта TESTDESK-15985).
- Имена Docker-образов и registry.
- Подтверждение, что CSP на кластерах UMP/UFR включён.
- Рабочая ссылка на инструкцию CSP (в PDF «ссылка»).

---

## CSDINPROJT-25800 — SIEM

### Критерий из задачи

Новые СЗИ / серверы / VM / ПО / интеграции → события ИБ должны писаться и уходить в SIEM.

### Вывод

Задача **актуальна**. Появляется новое прикладное ПО и новые интеграции:

- сервисы `ufr-eos-ul-ncins-ui`, `ufr-eos-ul-ncins-core-api`, `ump-ncins-pa`, `corp-ncins-acc-api`;
- интеграции: Keycloak, Redis, AlfaCapture, SignOnline, OBIP, corp-gateway, UMP facade/products.

Что уже видно в коде/ТЗ:

- В `ump-ncins-pa` включён `alfalab.logging` (Feign, WS, HTTP servlet).
- NCINS-164: при входе фиксировать время, ID пользователя, IP/сессию; неуспешный вход — «если предусмотрено настройками» (не закрыто).
- SignOnline сам пишет журнал подписания (есть образец отчёта по операции SFA `non_credit_ins`).
- Проверка «уже в проде» по app_id на странице Confluence ДКБ — **не выполнена**: app_id нет, сервисы ещё не выглядят как прод.

**Нельзя** утверждать, что логи уже в SIEM. Нельзя закрывать задачу.

### Черновик комментария в Jira (ещё не заявка)

```text
@Федорова Мария Алексеевна

Задача актуальна: новое ПО и новые интеграции (SFA UI/middle NCINS, ump-ncins-pa, вызовы AlfaCapture/SignOnline/OBIP/corp-ncins-acc).

Состав (для заявки 04.02.42), предварительный:
1) ufr-eos-ul-ncins-ui — UI сотрудника SFA, Keycloak.
2) ufr-eos-ul-ncins-core-api — generate-form, download, download-signed, sign/create-operation.
3) ump-ncins-pa — Camunda-воркеры (документы, подписание, оплата, финализация, удаление).
4) corp-ncins-acc-api — расчёт премии, создание договора (через corp-gateway).

Логирование прикладное: alfalab.logging в ump-ncins-pa (HTTP/Feign).
События входа сотрудника: Keycloak + ТЗ NCINS-164 (время, user id, IP) — полнота относительно распоряжения 3535 не подтверждена.
Подписание клиента: журнал SignOnline (существующая система).

Заявку 04.02.42 и примеры сработавших событий пока не оформляли: нет app_id, нет выгрузки лога с датой/временем/местом хранения.
Просим подсказать ожидаемый app_id/api для проверки https://confluence.moscow.alfaintra.net/pages/viewpage.action?pageId=2093386332
```

### Чего нет (без этого SIEM не подключат)

- `app_id` / API id сервисов.
- Подтверждение, что набор событий соответствует распоряжению 3535 от 25.09.2024.
- Примеры сработавших событий: дата, время, путь к логу (Kibana/файл).
- Номер заявки 04.02.42.
- Ответ по NCINS-164: пишутся ли неуспешные логины, блокировки, недоступность Keycloak.
- Риск для ДКБ (не для комментария в git): в NCINS-308 в лог middle попадает DTO с `passwordAC`. Это надо убрать/замаскировать **до** сдачи в SIEM.

---

## CSDINPROJT-25801 — Vulnerability assessment / Pentest

### Критерий из задачи

Если есть создание/изменение **API или UI** — pentest обязателен. Иначе — комментарий и CANCELED.

### Вывод

Pentest **нужен**. Это не «только внутренние миграции».

Меняется и UI, и API:

- UI: `ufr-eos-ul-ncins-ui` (авторизация Keycloak, главный экран, ошибки доступа, `/api/examples`).
- API SFA: generate-form, download, download-signed, create-operation.
- API НИБ: calculate premium, ins-contracts.
- Интеграция getOperations.

П.2 инструкции (контакты) закрыт по ФИО. В BACKLOG **нельзя** переводить: обязательные пункты 3–4 из инструкции не закрыты (нет bitbucket, нет двух УЗ на каждую роль).

Мобильного приложения нет — пункт про binary не нужен.

### Черновик комментария (собрать недостающее и тегнуть Михайлову)

```text
@Михайлова Мария Васильевна

Разработка затрагивает создание/изменение API и UI, pentest требуется (не CANCELED).

Обязательное:
1. Проект: PJ07253-MMB23 RSM V ADRBT-20339 «Страхование в SFA/SFACall c AlfaCapture 1.0».
   Системы: SFA ЮЛ, ЕФ.Цифровое Страхование (ufr-eos-ul-ncins-ui / ufr-eos-ul-ncins-core-api),
   UMP ump-ncins-pa, AlfaCapture 1.0, SignOnline, Альфа-Бизнес (corp-ncins-acc-api).
2. Контакты:
   Product Owner: Ермолов Александр
   Dev Teamlead: Яковлев Максим
   QA Teamlead: Григорьев Дмитрий Вячеславович
   Аналитик: Самсонова Анастасия (SFA), Забарянский Юрий Геннадьевич (НИБ)
3. Исходники (Bitbucket) — НЕТ ССЫЛОК. Нужны репозитории:
   ufr-eos-ul-ncins-ui, ufr-eos-ul-ncins-core-api, ump-ncins-pa, corp-ncins-acc-api.
   Для getOperations есть git.moscow.alfaintra.net/projects/EOS_COMMON_SERVICES/repos/ufr-eos-module-info-api.
4. Тестовый стенд:
   UI: https://dev.ufrulkint-ui.moscow.alfaintra.net/ufr-eos-ul-ncins-ui/
   API INT: https://int.ufrulkint-api.moscow.alfaintra.net/ufr-eos-ul-ncins-core-api
   API DEV: https://dev.ufrulkint-api.moscow.alfaintra.net/ufr-eos-ul-ncins-core-api
   НИБ gateway TEST: http://corp-gateway-test.moscow.alfaintra.net/corp-ncins-acc-gateway/
   Вход в модуль: sessionId из service panel
   https://int.ufrulkint-ui.moscow.alfaintra.net/ufr-eos-ul-service-panel-gate-ui/
   далее /ufr-eos-ul-ncins-ui/?cus=...&sessionId=...
5. Тестовые УЗ: есть сотрудникская УЗ ADNCINS (пароль — в комментарии NCINS-164, не дублирую).
   Второй комплект на роль и УЗ клиента для полного флоу подписания — НЕТ.
   Роли: сотрудник SFA (Keycloak), клиент-подписант SignOnline, backend UMP (client ump-ncins).
6. Документация API:
   https://confluence.moscow.alfaintra.net/spaces/ECOSYSTEM/pages/3606241300/ (схема SFA)
   POST generate-form, POST sign/create-operation, getOperations — страницы ECOSYSTEM / EOSUL.
   Swagger/OpenAPI ссылкой на стенд — НЕТ.

Желательно:
- Kibana — нет ссылки.
- Изменения: новые эндпоинты generate-form, download, download-signed, create-operation,
  ins-premium/calculate, ins-contracts; UI авторизации Keycloak + getOperations; UMP-процессы NCINS.
- Мобильное приложение: нет.

Статус BACKLOG не ставим, пока нет п.3 (bitbucket) и двух УЗ на роль (п.5).
```

### Чего нет

- Ссылки Bitbucket на репозитории команды.
- Swagger на стенде.
- Корпоративная почта / rocket к ФИО выше (для ДКБ обычно достаточно ФИО).
- По 2 тестовые УЗ на каждую роль (сейчас одна сотрудникская ADNCINS).
- Клиентская УЗ для полного флоу SignOnline (в отчёте подписания есть боевой/интеграционный прогон, но не оформлен как тестовый комплект для ДКБ).
- Kibana.
- Явный перечень feature-toggle / список экранов UI.

---

## CSDINPROJT-25802 — WAF Imperva

### Критерий из задачи

Любой сервис в **недоверенные** сети (Интернет, сети партнёров, VPN) должен быть за WAF.  
Если интеграция система–система через Datapower — WAF **не** нужен.  
Новые функции/API/страницы уже опубликованного сервиса требуют **донастройки** WAF.

### Вывод

Для **новых** сервисов команды (SFA UI, ncins-core-api, ump-ncins-pa) публикация идёт на `*.moscow.alfaintra.net`. Это корпоративная сеть, не Интернет. Свой публичный URL проект не показывает.

Клиентский интернет-фронт подписания — **SignOnline / RB_SIGN_UI**, уже существующий сервис банка (в отчёте подписания канал SFA, SMS-ссылка). Новый WAF под «наш» сервис из-за этого не следует: донастройка, если вообще нужна, на стороне SignOnline при новом `prodType=non_credit_ins` / канале SFA.

Отдельный риск: канал **Альфа-Бизнес / НИБ**. Если новые страницы/API НИБ торчат в Интернет (не только corp-gateway во внутренней сети), WAF для Альфа-Бизнеса нужно донастроить. В доступном коде виден только `corp-gateway-test.moscow.alfaintra.net` (интранет).

Исполнитель заявки IT0423 по тексту ДКБ — **команда проекта**, не сотрудник ДКБ.

### Черновик комментария в Jira

```text
@Федорова Мария Алексеевна

Новые сервисы команды (ufr-eos-ul-ncins-ui, ufr-eos-ul-ncins-core-api, ump-ncins-pa)
не публикуются в Интернет / сети партнёров. Хосты стендов — *.moscow.alfaintra.net.
Сотрудник работает из SFA после Keycloak.

Интеграции система–система: UFR, UMP, AlfaCapture, SignOnline, corp-gateway (внутренняя сеть).
Клиент подписывает на существующем интернет-фронте SignOnline (не новый сайт проекта).

Предварительно: заводить новый объект WAF под сервисы команды не требуется.
Просим подтвердить:
1) SFA UI не считается публикацией в VPN/недоверенную сеть;
2) для канала НИБ/Альфа-Бизнес новые API идут только через уже защищённый контур Альфа-Бизнеса
   (тогда нужна донастройка существующего WAF — да/нет и какой application id);
3) SignOnline: нужна ли донастройка WAF на новый prodType non_credit_ins / канал SFA.

Заявку IT0423 не оформляли до этого подтверждения.
```

### Чего нет

- Решение СА: является ли доступ сотрудников SFA «VPN / недоверенная сеть».
- Application id текущего WAF Альфа-Бизнеса и SignOnline.
- Подтверждение, что НИБ-методы не будут опубликованы отдельно в Интернет.
- Номер IT0423 (основание закрытия по тексту задачи).

---

## Что прислать мне / дописать в Jira, чтобы добить ответы

Минимальный пакет от команды (без этого SIEM и pentest не сдвинуть, Datapower/CSP/WAF не закроют официально):

1. **ОТАР / актуальный Vision** (файл или доступ) — в эпике только rsm-ссылка, текста нет.
2. **Bitbucket** трёх репозиториев: `ufr-eos-ul-ncins-ui`, `ufr-eos-ul-ncins-core-api`, `ump-ncins-pa` (+ corp-ncins-acc, если он ваш).
3. ~~Контакты PO / Dev TL / QA TL~~ — есть: Ермолов Александр, Яковлев Максим, Григорьев Дмитрий Вячеславович. Почта/rocket по желанию.
4. **Две УЗ на роль** для pentest (сотрудник SFA, при необходимости вторая роль; клиент SignOnline). Пароли — только в комментарии Jira ДКБ.
5. **app_id** сервисов и ссылка Kibana.
6. **Письмо/чат платформы UMP и UFR:** CSP на кластере включён? Какие namespace/кластера прод?
7. **Письмо СА по шлюзам:** corp-gateway = Datapower? Нужна ли заявка 02.02.01.05? Нужен ли WAF (IT0423) на НИБ/SignOnline?
8. **Swagger** ncins-core-api на INT/DEV.

Пока нет Bitbucket и двух УЗ на роль, в Jira можно **уже сегодня** положить черновики 25797 / 25798 / 25802 и **обновлённый** комментарий 25801 с контактами. Статус BACKLOG у pentest не ставить.
