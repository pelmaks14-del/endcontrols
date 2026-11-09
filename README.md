# End Control — Fabric-мод для Minecraft 26.2

Серверный мод, который позволяет администраторам временно закрывать и открывать измерение The End
через GUI-панель или команды. Таймер идёт в реальном времени и переживает перезапуск сервера.

> **Статус сборки.** Логика таймера и парсер времени проверены тестами (39 проверок, Java 21).
> Сам мод **не был собран и не запускался**: в среде, где он писался, нет Java 25 и закрыт доступ к Maven.
> Перед использованием выполните сборку и чек-лист из раздела «Проверка после сборки».

## Загрузчик и версии

| Компонент | Версия |
|---|---|
| Minecraft | 26.2 (без обфускации, имена Mojang) |
| Java | 25 |
| Fabric Loader | ≥ 0.19.3 |
| Fabric API | 0.152.1+26.2 (или новее для 26.2) |
| Fabric Loom | `net.fabricmc.fabric-loom` 1.17-SNAPSHOT |
| Gradle | 9.6.0 (wrapper уже в проекте) |

Выбран Fabric: для 26.2 подтверждены Loom, Fabric API и встроенный Permission API (`fabric-api` → `permission.v1`).

## Структура проекта

```
endcontrol/
├── build.gradle, settings.gradle, gradle.properties
├── gradlew, gradlew.bat, gradle/wrapper/…
└── src/main/
    ├── resources/
    │   ├── fabric.mod.json
    │   └── endcontrol.mixins.json
    └── java/dev/endcontrol/
        ├── EndControlMod.java              точка входа (сервер + клиент): пакеты, команды, события
        ├── core/                           логика без зависимостей от Minecraft
        │   ├── TimeUtil.java               разбор «1h30m», форматирование HH:MM:SS
        │   ├── LockState.java              состояние и таймер (абсолютное время окончания)
        │   ├── StateStore.java             сохранение state.json (атомарная запись)
        │   ├── EndControlConfig.java       config/endcontrol.json
        │   └── AuditLog.java               журнал действий админов
        ├── server/
        │   ├── EndLockManager.java         ядро: закрыть/открыть/продлить, тик, рассылка панели, проверка входа
        │   ├── EndControlPermissions.java  права (Fabric Permission API)
        │   └── Text.java                   &-цвета и плейсхолдеры в сообщениях
        ├── net/                            ActionPayload (C2S), StatePayload и OpenPanelPayload (S2C)
        ├── command/EndControlCommand.java  /endcontrol …
        ├── mixin/ServerPlayerMixin.java    блокировка любой смены измерения в Энд
        └── client/                         EndControlClient, EndControlScreen (панель), ClientState
```

## Сборка

```bash
# нужен JDK 25 и доступ в интернет
./gradlew build
# готовый мод: build/libs/endcontrol-1.0.0.jar   (не берите *-sources.jar)
```

## Установка на сервер

1. Установите Fabric Loader 0.19.3+ для Minecraft 26.2 на сервер (Java 25).
2. В папку `mods/` положите `fabric-api-…+26.2.jar` и `endcontrol-1.0.0.jar`.
3. Запустите сервер. Появятся `config/endcontrol.json` и папка `endcontrol/` (`state.json`, `audit.log`).
4. **Игрокам-администраторам** для GUI-панели нужен тот же `endcontrol-1.0.0.jar` (и Fabric API) на клиенте.
   Обычным игрокам мод ставить не нужно: они подключаются как обычно и просто не могут войти в закрытый Энд.
   Без мода на клиенте администратор может пользоваться командами.

## Использование

| Команда | Действие |
|---|---|
| `/endcontrol` | открыть панель (нужен мод на клиенте) |
| `/endcontrol status` | статус, остаток времени, игроков в Энде |
| `/endcontrol open` | открыть Энд |
| `/endcontrol close [время]` | закрыть Энд (без времени — `defaultDurationMinutes` из конфига) |
| `/endcontrol extend <время>` | продлить блокировку |
| `/endcontrol shorten <время>` | сократить блокировку (до нуля — Энд открывается) |
| `/endcontrol reload` | перечитать конфиг |

Формат времени: `30s`, `10m`, `2h`, `1h30m`, `1d`; число без единицы — минуты; допустимы `с`, `м`, `ч`.

**Панель:** вверху — статус (красный «ЗАКРЫТ» / зелёный «ОТКРЫТ»), крупный таймер, число игроков в Энде.
Ниже введите число, выберите единицу (сек / мин / час) и нажмите «Закрыть Энд».
«Открыть Энд» и «Досрочно открыть» работают при закрытом Энде; «Продлить» и «Сократить» меняют
оставшееся время на введённое значение. Ответ сервера показывается внизу панели.

## Как работает блокировка

* Любая смена измерения игрока (портал Энда, `/tp`, `/execute in minecraft:the_end run tp`, телепорт к сущности
  из спектатора, команды других модов) проходит через `ServerPlayer#teleport(TeleportTransition)`.
  Миксин отменяет такой переход, пока Энд закрыт, и отправляет игроку настраиваемое сообщение.
* Блокируется только **вход**. Игроки, уже находящиеся в Энде, не перемещаются и ничего не теряют.
  Выйти из Энда они могут, вернуться — только после открытия.
* Таймер хранит абсолютное время окончания, поэтому идёт в реальном времени. По умолчанию он идёт и пока
  сервер выключен (`countDownWhileServerOffline`: `false` — «заморозить» таймер на время простоя).
* Автооткрытие проверяется каждый тик; в панели таймер обновляется ежесекундной синхронизацией.

## Права доступа

Через Fabric Permission API. Без плагина прав управлять может уровень **ADMINS** (оператор).
С провайдером прав (например LuckPerms) выдавайте узлы:

* `endcontrol:manage` — команды и панель (каждый пакет панели перепроверяется на сервере);
* `endcontrol:bypass` — вход в закрытый Энд. По умолчанию никому; `adminsBypassLock: true` даёт его ADMINS.

## Конфиг (`config/endcontrol.json`)

`defaultDurationMinutes`, `maxDurationHours`, `countDownWhileServerOffline`, `adminsBypassLock`,
`broadcastChanges`, `deniedMessageCooldownSeconds`, `logBlockedAttempts`, `logToFile` и блок `messages`
(`denied`, `closedBroadcast`, `openedBroadcast`, `autoOpenedBroadcast`, `noPermission`).
В сообщениях работают коды `&c`, `&a`, `&l`… и плейсхолдеры `{time}`, `{duration}`, `{admin}`, `{player}`.

## Журнал

`endcontrol/audit.log` и консоль: кто и когда закрыл/открыл/продлил Энд, отклонённые попытки без прав,
заблокированные попытки входа, автооткрытие по таймеру.

## Проверка после сборки

Эти места опираются на API 26.2, которое я не смог проверить компиляцией. При ошибке сначала смотрите сюда
(`./gradlew genSources` покажет исходники Minecraft):

1. **`ServerPlayerMixin`**: сигнатура `ServerPlayer.teleport(TeleportTransition) → ServerPlayer` и аксессор
   `TeleportTransition.newLevel()`. Если не совпало, игра не запустится с явной ошибкой миксина
   (`defaultRequire: 1`) — тихого обхода блокировки не будет.
2. **`EndControlClient`**: открытие экрана через `Minecraft.setScreenAndShow(...)` (по данным 26.2; в более новых
   версиях может быть `gui.setScreen(...)`).
3. **`EndControlScreen`**: `addRenderableOnly(...)` с лямбдой, `GuiGraphicsExtractor.fill/text`.
4. **`ServerPlayer.connection`** и `ServerPlayNetworking.canSend(listener, type)`.

Проверьте на тестовом сервере с двумя игроками (админ и обычный), пока Энд закрыт:

- [ ] вход через портал Энда не работает, игрок получает сообщение, повторный спам ограничен паузой;
- [ ] `/tp @s 0 100 0` в измерении Энда и `/execute in minecraft:the_end run tp @s 0 100 0` не работают;
- [ ] телепорт к игроку, находящемуся в Энде (`/tp`, спектатор), не работает;
- [ ] игрок, бывший в Энде при закрытии, остаётся на месте с инвентарём;
- [ ] после окончания таймера Энд открывается сам, всем приходит сообщение;
- [ ] перезапуск сервера при закрытом Энде: статус и остаток времени сохранились;
- [ ] обычный игрок не может выполнить `/endcontrol`, а поддельный пакет панели отклоняется (запись в `audit.log`).
