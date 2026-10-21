# Мультисервер в OwpenGram (Android)

Документ описывает, **как устроен** мультисерверный режим в этом репозитории и
**как его включить/доработать** (пошаговое руководство).

---

## 0. Коротко

Это форк Telegram Android (TMessagesProj), поэтому сети здесь нет в привычном виде
(RxJava/OkHttp/Retrofit/Room нет вообще). Транспорт — **MTProto поверх обычных TCP-сокетов,
реализованный нативно на C++** в `TMessagesProj/jni/tgnet/`.

Отсюда главное определение:

> **«Сервер» = MTProto-эндпоинт (host:port + RSA-ключ сервера).**

А «мультисервер» = возможность держать в одном приложении несколько аккаунтов
(в телеграм-слотах 0..9), каждый из которых подключён к **своему** эндпоинту.

| Слой | Файл |
|---|---|
| Модель сервера | `org/telegram/owpengram/OwpengramServer.java` |
| Реестр серверов, привязка к аккаунтам, feature-gating | `org/telegram/owpengram/OwpengramServers.java` |
| Email-вход как идентичность (фича бэкенда) | `org/telegram/owpengram/EmailSignupPhone.java` |
| Экран выбора сервера | `org/telegram/ui/ServerSelectFragment.java` |
| Добавление/редактирование сервера | `org/telegram/ui/AddServerFragment.java` |
| Инфо о сервере / удаление | `org/telegram/ui/ServerInfoFragment.java` |
| Нативное применение конфига | `TMessagesProj/jni/tgnet/ConnectionsManager.cpp` → `applyServerConfig` |
| JNI-мост | `TMessagesProj/jni/TgNetWrapper.cpp`, `org/telegram/tgnet/ConnectionsManager.java` |

> **Важно про текущее состояние репозитория.**
> В рабочем дереве (незакоммиченные изменения) мультисерверный выбор **выключен из UX**:
> все точки входа логина переведены на `SumeGramLogin.startLogin(...)`, который всегда
> берёт `OwpengramServers.owpengramServer()`. При этом **вся инфраструктура мультисервера
> осталась на месте и рабочей** — `ServerSelectFragment`, `AddServerFragment`,
> `ServerInfoFragment`, `OwpengramServers.listServers()` и т.д. сейчас просто не вызываются
> (dead code, но живой). Коммит `19512d39a` (HEAD) ещё показывает выбор сервера во всех
> входных точках. Раздел «Как включить обратно» — в конце файла.

---

## 1. Модель сервера

`TMessagesProj/src/main/java/org/telegram/owpengram/OwpengramServer.java`

```java
public class OwpengramServer {
    public String id;              // "owpengram" | "telegram" | "sumegram" | UUID
    public String name;            // отображаемое имя
    public String host;            // IP или домен
    public int    port;            // MTProto-порт
    public String description = "";
    public String rsaPublicKey = "";   // PEM (RSA PUBLIC KEY)
    public long   rsaKeyFingerprint = 0; // 0 = вычислить нативно из PEM
    public String logoPath = "";
    public boolean isOfficial;     // встроенный (true) vs пользовательский (false)
    public boolean isTelegram;     // официальная сеть Telegram (multi-DC, стоковые ключи)
    public boolean multiDc;        // сервер сам multiplex'ит несколько DC
    public int    mainDcId;        // домашний DC (1..5)

    public int effectiveMainDcId() {
        if (mainDcId > 0) return mainDcId;
        return multiDc ? 2 : 1;
    }
}
```

Это обычный POJO с JSON-сериализацией (`toJson()` / `fromJson()`), **не** Parcelable,
**не** Room-сущность. Флаги `isOfficial` и `isTelegram` при чтении из JSON не
восстанавливаются: `isOfficial` принудительно `false` для кастомных серверов, `isTelegram`
у кастомных всегда `false` (кастом = single-DC, не Telegram).

Авторизации по API-ключу нет вообще: аутентификация — **MTProto RSA-handshake + auth keys**.

---

## 2. Реестр серверов и привязка к аккаунтам

`TMessagesProj/src/main/java/org/telegram/owpengram/OwpengramServers.java`

### 2.1 Хранилище

`SharedPreferences` с именем файла `"owpengram_servers"`:

| Ключ | Назначение |
|---|---|
| `custom_servers` | JSON-массив кастомных серверов |
| `account_server_<N>` | id сервера, привязанного к слоту аккаунта N |

```java
private static final String PREFS_NAME = "owpengram_servers";
private static final String KEY_CUSTOM_SERVERS = "custom_servers";
private static final String KEY_ACCOUNT_SERVER = "account_server_";
```

### 2.2 Встроенные серверы

| id | Имя | host:port | isTelegram | multiDc | RSA |
|---|---|---|---|---|---|
| `telegram` | Telegram | `149.154.167.51:443` | true | true | стоковый Telegram, fp `0xd09d1d85de64fd85L` |
| `owpengram` | SumeGram | `2.27.98.189:2398` | false | false | `OWPENGRAM_RSA_KEY` (prod) |
| `sumegram` | SumeGram Test | `2.27.98.189:2498` | false | false | `SUMEGRAM_RSA_KEY` |

Фабрики: `telegramServer()`, `owpengramServer()`, `sumegramServer()`.
Список: `listServers()` = 3 встроенных + `loadCustomServers()`.
Поиск: `getServerById(id)`.

### 2.3 CRUD кастомных серверов

```java
loadCustomServers()          // распарсить JSON
saveCustomServers(list)      // private
addCustomServer(server)      // выдаёт UUID, если id пуст; ставит isOfficial = false
updateCustomServer(server)   // замена по id
removeCustomServer(id)
```

### 2.4 Привязка «аккаунт → сервер»

```java
getServerIdForAccount(int accountNum)
setServerForAccount(String serverId, int accountNum)
getServerForAccount(int accountNum)
hasServerForAccount(int accountNum)
```

Аккаунт-слот — это стандартный телеграмовский `UserConfig` (`MAX_ACCOUNT_COUNT = 10`,
`MAX_ACCOUNT_DEFAULT_COUNT = 3`). `AccountInstance` — «инстанс» слота; сервер привязан к
нему 1:1 через `OwpengramServers.getServerForAccount()`.

### 2.5 Лимиты аккаунтов

Лимит в 3 (free) / 6 (premium) аккаунта применяется **только к Telegram-серверам**;
кастомные ограничены лишь общим числом слотов (10). Это поведение десктоп-клиента.

```java
firstFreeAccountSlot()
countTelegramAccounts()
telegramAccountLimit()   // 3 или 6
canAddTelegramAccount()
anyAccountIsTelegram()
```

---

## 3. Как сервер применяется к аккаунту (ядро)

### 3.1 Java-слой

`OwpengramServers.applyServerToAccount(server, accountNum, resetKeys)`:

1. Выбирает RSA-ключ и fingerprint:
   - `server.isTelegram` → `TELEGRAM_RSA_KEY` + `TELEGRAM_RSA_FINGERPRINT`;
   - кастом со своим ключом → `server.rsaPublicKey`, fingerprint `0` (нативка посчитает);
   - кастом без ключа → общий `OWPENGRAM_RSA_KEY`.
2. Определяет домашний DC: `server.mainDcId > 0 ? mainDcId : (isTelegram ? 2 : 1)`.
3. Вызывает JNI:

```java
ConnectionsManager.native_applyServerConfig(
    accountNum, host, port, isTelegram, mainDc, key, fingerprint, resetKeys);
```

`resetKeys`:
- `true` — пользователь выбрал **новый** сервер → старые auth keys не годятся, нужен
  новый handshake;
- `false` — восстановление при старте приложения → ключи и сессии сохраняем.

`applyStoredServerToAccount(accountNum)` вызывается из
`ApplicationLoader` (строка ~257) в цикле инициализации каждого аккаунта —
это и есть «сервер подхватывается при запуске».

### 3.2 Нативный слой (C++)

`TMessagesProj/jni/tgnet/ConnectionsManager.cpp` → `ConnectionsManager::applyServerConfig(...)`.
Всё выполняется в сетевом потоке через `scheduleTask(...)`.

**Шаг 1 — per-instance RSA-ключ:**

```cpp
customPublicKey = pemKey;
customPublicKeyFingerprint = (fingerprint != 0)
        ? fingerprint
        : Handshake::calculatePublicKeyFingerprint(pemKey);
Handshake::cleanupServerKeys();   // сброс thread_local кэша ключей
```

Ключи хранятся **per ConnectionsManager instance**, т.е. per account-слот
(`ConnectionsManager.h`: `customPublicKey`, `customPublicKeyFingerprint`).
Это критично: `Handshake::serverPublicKeys` — `thread_local`-массив, общий для всех
аккаунтов. При хендшейке ключ берётся **из инстанса аккаунта**, а не из глобального кэша
(`Handshake.cpp`, ~411):

```cpp
serverPublicKeys.clear();
serverPublicKeysFingerprints.clear();
auto &cm = ConnectionsManager::getInstance(currentDatacenter->instanceNum);
if (!cm.customPublicKey.empty() && cm.customPublicKeyFingerprint != 0) {
    serverPublicKeys.push_back(cm.customPublicKey);
    serverPublicKeysFingerprints.push_back(cm.customPublicKeyFingerprint);
} else if (cm.testBackend) { /* тестовый ключ */ }
else { /* стоковый Telegram-ключ */ }
```

**Шаг 2 — таблица адресов DC 1..5:**

*Ветка `isTelegram == true`* (multi-DC, поведение как у стокового Telegram):
- создаются/восстанавливаются все 5 DC с их домашними адресами
  (`149.154.175.50:443` … `149.154.171.5:443`);
- используется **ADD** (не перезапись), чтобы адреса, полученные от `help.getConfig`,
  сохранялись — иначе не грузились бы медиа/стикеры/реакции с DC3/4/5;
- заполняются **и обычные, и download/media-адреса** — чтобы медиа-соединение имело
  эндпоинт ещё до ответа `getConfig`;
- ⚠️ **Исключение**: если аккаунт раньше сидел на single-server (locked) — все dc_id 1..5
  были принудительно перемаплены на чужой хост через `replaceAddresses`. Тогда ADD
  оставил бы «отравленные» записи, и DC1/3/4/5 продолжали бы резолвиться на старый
  сервер (реальная примена бага «не могу залогиниться в Telegram после кастомного
  сервера»). Поэтому при `wasLocked` используется **REPLACE**-семантика;
- при `resetKeys` — сброс auth keys/салтов, домашний DC = `mainDcId` (обычно 2);
- вызывается `updateDcSettings(0, false, false)` для обновления media/download-эндпоинтов.

*Ветка `isTelegram == false`* (single-server бэкенд):
- `serverOptionsLocked = true`;
- **все** dc_id 1..5 мапятся на единственный `host:port` — и обычный адрес, и
  download/media-адрес. Это нужно, чтобы ссылки на файлы вида `dc 3/4/5` продолжали
  соединяться;
- сессии пересоздаются, при `resetKeys` чистится auth key;
- `currentDatacenterId = mainDcId`.

**Шаг 3 — блокировка `help.getConfig`:**

```cpp
// ConnectionsManager.cpp:3539
if (!map.empty() && !serverOptionsLocked) { /* применить dc_options с сервера */ }
```

Для single-server бэкенда `dc_options` от сервера **игнорируются** — иначе сервер
мог бы перенаправить клиента куда угодно. Для Telegram — наоборот, применяются.
(Аналог `DcOptions::optionsLocked` в десктоп-клиенте.)

**Шаг 4 — переподключение:**

```cpp
Datacenter *cur = getDatacenterWithId(currentDatacenterId);
if (cur && !cur->hasAuthKey(...) && !cur->isHandshakingAny()) {
    cur->clearServerSalts(false); cur->clearServerSalts(true);
    cur->beginHandshake(HandshakeTypeAll, true);
}
```

Плюс `movingToDatacenterId = DEFAULT_DATACENTER_ID; saveConfig();`.

---

## 4. UX

### 4.1 `ServerSelectFragment` — выбор сервера

`TMessagesProj/src/main/java/org/telegram/ui/ServerSelectFragment.java` (весь UI программный, XML-лэйаутов нет).

* Список карточек: логотип-кружок, имя, `host:port`, **живой статус** (Online · N ms / Offline),
  кнопка **Join**.
* Список разбит на группы «Official servers» / «Custom servers» + строка «+ Add a server»
  (`buildRows()`, типы строк `TYPE_HEADER/TYPE_SERVER/TYPE_ADD/TYPE_SPACER`).
* Пинг — обычный TCP-connect с таймаутом 3 с в пуле потоков
  (`pingAll()` / `pingServer()` / `pingTcp()`), кэш в `pingCache`, `PING_CHECKING = -2`.
* Тап по строке → `ServerInfoFragment`; тап «Add» → `AddServerFragment`.

**Join-флоу** (парность с десктопом, `joinServer()` → `proceedJoin()`):

```
joinServer(server)
 ├─ проверка лимита Telegram-аккаунтов (LimitReachedBottomSheet)
 ├─ pingTcp → если недоступен: диалог "Server unreachable" / "Connect anyway"
 └─ proceedJoin(server)
      ├─ OwpengramServers.setServerForAccount(server.id, targetAccount)
      ├─ OwpengramServers.applyServerToAccount(server, targetAccount)  // resetKeys = true
      ├─ ConnectionsManager.getInstance(target).resumeNetworkMaybe()
      ├─ блокирующий диалог-спиннер "Connecting"
      └─ waitForConnection(): poll native_getConnectionState(account) каждые 200 мс,
         таймаут 20 c (CONNECT_TIMEOUT_MS)
           ├─ ConnectionStateConnected → checkEmailSignupAndGoToLogin()
           └─ таймаут → "Connection failed" (Continue / Back)
```

`checkEmailSignupAndGoToLogin()` делает «сырой» `help.getAppConfig` с флагами
`RequestFlagWithoutLogin | RequestFlagEnableUnauthorized | RequestFlagFailOnServerErrors`
и читает `email_signup_enabled` — от этого зависит, показывать ли экран телефона
или email. Флаг затем прокидывается в `LoginActivity.setEmailSignupEnabled(...)`.

### 4.2 `AddServerFragment` — добавление/правка кастомного сервера

Поля (программные вьюхи):
- **Name** (обязательно), **Description (optional)**;
- **Host / IP address** (обязательно), **Port** (по умолчанию 2398, валидация 1..65535);
- **Multi-DC mode** — тумблер, только для Telegram-совместимых серверов;
- **Main data center (1-5)** — скрыт, если включён Multi-DC;
- **RSA Public Key (PEM)**.

`save()` валидирует, собирает `OwpengramServer` и вызывает
`addCustomServer()` / `updateCustomServer()`, затем `onSavedListener.onSaved(server)`.

### 4.3 `ServerInfoFragment` — карточка сервера

Hero-карточка (лого, имя, `host:port`, бейджи «Official» / «Telegram» / «Multi-DC» | «Single DC»),
карточка «Connection status» с живым пингом и кнопкой «Check again», карточка «About»
и карточка **«Accounts»** — сколько активированных аккаунтов привязано к этому серверу:

```java
for (int a = 0; a < UserConfig.MAX_ACCOUNT_COUNT; a++) {
    if (UserConfig.getInstance(a).isClientActivated()) {
        OwpengramServer s = OwpengramServers.getServerForAccount(a);
        if (s != null && server.id.equals(s.id)) count++;
    }
}
```

Для кастомных серверов — кнопки **Edit** / **Delete** (удаление через `removeCustomServer`).

### 4.4 Где доступен выбор сервера

В HEAD (`19512d39a`) `new ServerSelectFragment(...)` вызывается из:

| Место | Строка |
|---|---|
| `IntroActivity` — «Start Messaging» | ~410 |
| `LaunchActivity.getClientNotActivatedFragment()` | ~1108 |
| `LogoutActivity` — «Add account» | ~127 |
| `MainTabsActivity` — меню аккаунтов «Add Account» | ~517 |
| `ProfileActivity` — строка «Add account» в настройках | ~14528 |
| `UserInfoActivity` — кнопка `BUTTON_ADD_ACCOUNT` | ~497 |

В рабочем дереве все эти места заменены на `SumeGramLogin.startLogin(...)`
(см. §8).

---

## 5. Per-server состояние и feature-gating

Мультисервер — не только «разные адреса»: часть состояния и фич должна
различаться по серверам, иначе данные одного бэкенда протекают в UI другого.

### 5.1 Scope локального состояния

```java
// OwpengramServers.java:237
public static String serverScopeKeyForAccount(int accountNum) {
    OwpengramServer s = getServerForAccount(accountNum);
    if (s == null || s.isTelegram || TextUtils.isEmpty(s.host)) return "";
    return s.host.toLowerCase(Locale.US);
}
```

Ключ — **host**, а не `id`: один и тот же физический бэкенд, добавленный дважды под
разными локальными id, должен делить один scope. Для Telegram и для «неопознанных»
возвращается `""` — старые (несуффиксированные) ключи хранилища сохраняются,
миграция не нужна.

Используется в:
- `Emoji.java` — «недавние» и «часто используемые» кастомные эмодзи хранятся per-server
  (`emojis2:<host>`), плюс in-memory карты `recentByScope` / `historyByScope`.
  Причина: id кастомного эмодзи — это document id, и на другом сервере он не резолвится
  навсегда (у `AnimatedEmojiDrawable.EmojiDocumentFetcher` нет отрицательного кэша).
- `LocaleController.java` — при смене активного аккаунта, если scope сервера изменился,
  **удалённый языковой пакет перекачивается**, иначе строки бренда одного сервера
  «протекают» в другой аккаунт (`lastActiveServerScope`).

### 5.2 Фичи, зависящие от сервера

| Метод | Логика | Где используется |
|---|---|---|
| `serverSupportsPremium(account)` | Telegram → `true`; иначе по appConfig `premium_purchase_blocked` / `stars_purchase_blocked` (`MessagesController.premiumLocked` / `starsLocked`) | `SettingsActivity`, `ProfileActivity` — прячет блок Premium/Stars/TON/Business |
| `serverIsOfficialTelegram(account)` | `server.isTelegram` | прячет Telegram-only пункты помощи |
| `serverHasEmailSignup(account)` | appConfig `email_signup_enabled` (`MessagesController.emailSignupEnabled`) | `ActionIntroActivity` — блокирует «сменить номер» с `R.string.PhoneNumberChangeNotSupported` |
| `sameAccountServer(a, b)` | один и тот же id **или** один и тот же эндпоинт (host+port+isTelegram) | `LoginActivity` — проверка дубликата логина (один и тот же номер на **разных** серверах = разные аккаунты) |

### 5.3 Email как идентичность

`org/telegram/owpengram/EmailSignupPhone.java` — кодирование email в синтетический
номер вида `888…` (зеркалит `EncodeEmailPhone` в Go-сервере и `Core::EncodeEmailSignupPhone`
в десктоп-клиенте):

```java
normalizeEmail(String email)
encode(String email)     // email -> "888..." phone
decode(String phone)     // phone -> email
isEmailSignupPhone(String phone)
```

Используется в `LoginActivity.EmailSignupView`.

### 5.4 Маршрутизация ссылок `owpg://`

Схема зарегистрирована в `AndroidManifest.xml` (~строка 287): `<data android:scheme="owpg" />`.

```java
findOwpengramAccountForHost(String host)  // -1, если не нашлось
```

Ищет активированный аккаунт, который «владеет» хостом: совпадение с `me_url_prefix`
(`MessagesController.linkPrefix`, приходит из appConfig) **или** с `server.host` аккаунта.
Официальный Telegram исключается (там `t.me`). Сначала проверяется выбранный аккаунт.

Потребители: `Browser.java` (переписывает хост в `t.me`, чтобы ссылка осталась в приложении),
`LaunchActivity.java` — при отсутствии владельца показывает диалог
`showOwpengramOtherServerDialog()`.

### 5.5 Группировка аккаунтов по серверам

`MainTabsActivity` (`serverLabel(int)` + `serverHeaderView(String)`) группирует аккаунты
в переключателе по серверам с разделителем между группами. То же в `SettingsActivity`,
`UserInfoActivity`. В шапке списка чатов (`DialogsActivity`) вместо «Telegram»
показывается имя текущего сервера (`serverNameForAccount`).

---

## 6. Поток данных (итого)

```
UI: ServerSelectFragment / AddServerFragment / ServerInfoFragment
        │
        ▼
OwpengramServers.setServerForAccount(id, accountNum)      // SharedPreferences
OwpengramServers.applyServerToAccount(server, accountNum, resetKeys)
        │
        ▼  JNI
ConnectionsManager::applyServerConfig(host, port, isTelegram, mainDc, pem, fp, resetKeys)
        │
        ├─ customPublicKey / fingerprint  (per-instance)
        ├─ Handshake::cleanupServerKeys()
        ├─ таблица DC 1..5  (ADD для Telegram, REPLACE + lock для single-server)
        ├─ serverOptionsLocked = true  → help.getConfig игнорируется
        ├─ recreateSessions / clearAuthKey (если resetKeys)
        └─ beginHandshake(HandshakeTypeAll, true)
        │
        ▼
MTProto (TCP + RSA handshake) → MessagesController / SendMessagesHelper / FileLoader / ...
        │
        ▼
appConfig (premium_purchase_blocked, stars_purchase_blocked, email_signup_enabled,
           me_url_prefix, owpengram) → feature-gating
```

---

## 7. Как это сделать (пошагово) — если делать с нуля

Ниже — рецепт «как реализовать мультисервер», пригодный для переноса на другую
ветку/форк. Порядок соответствует слоям снизу вверх.

### Шаг 1. Модель и хранилище

1. Создать пакет, например `org.telegram.owpengram`.
2. `OwpengramServer` — POJO с полями `id, name, host, port, description, rsaPublicKey,
   rsaKeyFingerprint, logoPath, isOfficial, isTelegram, multiDc, mainDcId`
   + `toJson()/fromJson()`.
3. `OwpengramServers` — статический реестр:
   - `SharedPreferences("owpengram_servers")`;
   - ключ `custom_servers` (JSON-массив) и `account_server_<N>` (id сервера на слот);
   - фабрики встроенных серверов, `listServers()`, `getServerById()`;
   - CRUD кастомных серверов;
   - привязка аккаунт↔сервер.

### Шаг 2. Нативная поддержка «одного эндпоинта на все DC»

1. В `ConnectionsManager.h` добавить в инстанс:
   ```cpp
   std::string customPublicKey;
   uint64_t customPublicKeyFingerprint = 0;
   bool serverOptionsLocked = false;
   ```
2. В `ConnectionsManager.cpp` реализовать `applyServerConfig(...)`:
   - сохранить ключ в инстанс, вычислить fingerprint из PEM при `fp == 0`
     (`Handshake::calculatePublicKeyFingerprint`), вызвать `Handshake::cleanupServerKeys()`;
   - для `isTelegram` — восстановить/добавить 5 реальных DC (ADD, обе разновидности
     адресов: обычный и download), при переходе с locked-режима — REPLACE;
   - для single-server — `replaceAddresses` на `host:port` для dc 1..5 и для
     download-адресов, `serverOptionsLocked = true`;
   - при `resetKeys` — `clearAuthKey`, `clearServerSalts`, `recreateSessions`;
   - `currentDatacenterId = mainDcId`, `saveConfig()`, `beginHandshake`.
3. В `Handshake.cpp` при хендшейке подставлять ключ **из инстанса аккаунта**
   (`ConnectionsManager::getInstance(currentDatacenter->instanceNum)`), а не из
   глобального `thread_local`-кэша; добавить `cleanupServerKeys()`.
4. В обработчике `help.getConfig` пропускать `dc_options`, если `serverOptionsLocked`.
5. JNI: объявить в `org/telegram/tgnet/ConnectionsManager.java`
   `native_applyServerConfig(int, String, int, boolean, int, String, long, boolean)`,
   реализовать в `TgNetWrapper.cpp` и зарегистрировать в таблице JNI-функций.

### Шаг 3. Привязка к аккаунтам и старт приложения

1. В `ApplicationLoader` после инициализации `ConnectionsManager` каждого слота:
   ```java
   OwpengramServers.applyStoredServerToAccount(a);   // resetKeys = false
   ```
2. При логине/логауте — `setServerForAccount(server.id, accountNum)`.
3. Дубликаты логина проверять per-server (`sameAccountServer`).

### Шаг 4. UI

1. `AddServerFragment` — форма (name/description/host/port/multiDC/mainDC/RSA PEM) +
   `save()` с валидацией → `addCustomServer` / `updateCustomServer`.
2. `ServerSelectFragment` — список с группами, живым пингом (TCP connect с таймаутом),
   кнопкой Join; флоу Join: ping → apply → `resumeNetworkMaybe()` → спиннер →
   poll `native_getConnectionState` до `ConnectionStateConnected` → `LoginActivity`.
3. `ServerInfoFragment` — hero, статус соединения, счётчик аккаунтов, Edit/Delete.
4. Точки входа: `IntroActivity`, `LaunchActivity.getClientNotActivatedFragment()`,
   `LogoutActivity`, `MainTabsActivity`, `ProfileActivity`, `UserInfoActivity`.

### Шаг 5. Per-server состояние

1. Ввести `serverScopeKeyForAccount(account)` и применять его к ключам локального
   состояния, которое привязано к id сущностей сервера (эмодзи, языковые пакеты и т.п.).
2. Добавить feature-gating по appConfig: `premium_purchase_blocked`,
   `stars_purchase_blocked`, `email_signup_enabled`, `me_url_prefix`.
3. Реализовать роутинг ссылок `owpg://` на аккаунт-владелец хоста.

### Шаг 6. Проверка

- Логин на кастомный сервер → logout → логин на Telegram → logout → снова кастом:
  auth keys не должны «протекать» между бэкендами (проверяется ветка `wasLocked` +
  `REPLACE`).
- Медиа с `dc 3/4/5` на single-server должно грузиться (все DC → один host).
- `help.getConfig` с сервера не должен переставлять DC-адреса на single-server.
- Два аккаунта на одном номере, но на разных серверах — не конфликт.
- Emoji-история не должна «протекать» между серверами.

---

## 8. Как вернуть мультисерверный выбор в UX (текущее состояние)

В рабочем дереве мультисервер выключен: все точки входа логина вызывают
`SumeGramLogin.startLogin(...)` → `connectAndBuildLogin(...)`, где жёстко выбирается
`OwpengramServers.owpengramServer()`.

Чтобы вернуть выбор:

1. В `SumeGramLogin.connectAndBuildLogin` заменить жёсткий выбор сервера на
   показ `ServerSelectFragment` (или сделать `ServerSelectFragment` точкой входа,
   которая сама вызывает `applyServerToAccount`).
2. В точках входа (`IntroActivity`, `LaunchActivity`, `LogoutActivity`,
   `MainTabsActivity`, `ProfileActivity`, `UserInfoActivity`) вернуть
   `presentFragment(new ServerSelectFragment(slot))`.
3. Убедиться, что `LoginActivity` умеет принять готовый `emailSignupEnabled`
   (уже есть `setEmailSignupEnabled(boolean)`).

`SumeGramLogin` при этом полезно оставить как «быстрый вход без выбора» — например,
для deep-link/shortcut-а.

---

## 9. Известные грабли

| Симптом | Причина | Решение |
|---|---|---|
| «Не могу залогиниться в Telegram после кастомного сервера» | DC1/3/4/5 остались перемаплены на кастомный хост (`replaceAddresses`) | В ветке `isTelegram` при `wasLocked` использовать **REPLACE**, а не ADD |
| Медиа/стикеры не грузятся на single-server | адреса download- DC не заполнены | маппить **и** download-адрес на тот же `host:port` |
| Сервер «подменяет» DC-адреса | `help.getConfig` применил `dc_options` | `serverOptionsLocked` + проверка `!serverOptionsLocked` в обработчике |
| Аккаунт A использует RSA-ключ аккаунта B | `thread_local` кэш ключей общий для аккаунтов | брать ключ из `ConnectionsManager::getInstance(instanceNum)` в хендшейке |
| Эмодзи из другого сервера «залипают» битыми | document id не существует на другом бэкенде, отрицательного кэша нет | per-server scope ключа (`serverScopeKeyForAccount`) |
| Бренд/строки одного сервера в UI другого | общий кэш языкового пакета | перезапрашивать lang pack при смене server scope |
| Не хватает DC для кастомного сервера | `mainDcId` вне 1..5 | `effectiveMainDcId()` + валидация в `AddServerFragment` |
| Экран логина сразу «Телефон», а нужен Email | флаг `email_signup_enabled` не успел приехать | `checkEmailSignupAndGoToLogin()` до `presentFragment(LoginActivity)` |

---

## 10. Связанные коммиты (история фичи)

```
ba1330abe  added first working multi-server functionality
bd92f583b  bugfix for connection to custom server
4c23b2814  Remove Teamgram from the server selection list
e1df85461  Scope recent custom emoji cache per backend server
b091244be  per-server lang packs
1f85641a5  Rebrand to LocalGram and fix server config
63bdd763b  rename to SumeGram
19512d39a  сменил RSA и порт   ← HEAD
```

---

## 11. Ссылки на файлы

```
TMessagesProj/src/main/java/org/telegram/owpengram/OwpengramServer.java
TMessagesProj/src/main/java/org/telegram/owpengram/OwpengramServers.java
TMessagesProj/src/main/java/org/telegram/owpengram/EmailSignupPhone.java
TMessagesProj/src/main/java/org/telegram/ui/ServerSelectFragment.java
TMessagesProj/src/main/java/org/telegram/ui/AddServerFragment.java
TMessagesProj/src/main/java/org/telegram/ui/ServerInfoFragment.java
TMessagesProj/src/main/java/org/telegram/ui/SumeGramLogin.java
TMessagesProj/src/main/java/org/telegram/tgnet/ConnectionsManager.java        (native_applyServerConfig)
TMessagesProj/src/main/java/org/telegram/messenger/ApplicationLoader.java   (applyStoredServerToAccount)
TMessagesProj/src/main/java/org/telegram/messenger/Emoji.java                (scopedPrefKey)
TMessagesProj/src/main/java/org/telegram/messenger/LocaleController.java     (lang pack per server)
TMessagesProj/src/main/java/org/telegram/messenger/MessagesController.java  (appConfig flags)
TMessagesProj/src/main/java/org/telegram/ui/Browser.java                     (owpg routing)
TMessagesProj/src/main/java/org/telegram/ui/LaunchActivity.java              (owpg routing, root login)
TMessagesProj/src/main/java/org/telegram/ui/MainTabsActivity.java            (grouping by server)
TMessagesProj/src/main/java/org/telegram/ui/SettingsActivity.java            (premium gating, server label)
TMessagesProj/src/main/java/org/telegram/ui/ProfileActivity.java             (premium gating)
TMessagesProj/src/main/java/org/telegram/ui/UserInfoActivity.java           (server label)
TMessagesProj/src/main/java/org/telegram/ui/DialogsActivity.java             (header server name)
TMessagesProj/src/main/java/org/telegram/ui/LoginActivity.java               (email signup, duplicate login)
TMessagesProj/jni/tgnet/ConnectionsManager.cpp     (applyServerConfig, serverOptionsLocked)
TMessagesProj/jni/tgnet/ConnectionsManager.h       (per-instance key fields)
TMessagesProj/jni/tgnet/Handshake.cpp              (per-instance RSA key, fingerprint)
TMessagesProj/jni/TgNetWrapper.cpp                 (JNI bridge)
TMessagesProj/src/main/AndroidManifest.xml         (owpg:// scheme)
```
