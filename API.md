# API Isola: свои живые активности на острове

Isola слушает локальный адрес **http://localhost:47811** (только этот компьютер). Через него любой скрипт, сборка, бот или программа могут показать на острове свою **живую активность** (прогресс, таймер, статус), обновлять её и убрать, а также показать короткое уведомление.

То же самое умеет сама программа из командной строки: `Isola.exe push …` (см. [ниже](#командная-строка)).

## Токен

Всё, кроме `GET /api/ping`, требует заголовок **`X-Isola-Token`**.

- Где взять: **Настройки → Система → «API для своих скриптов» → «Копировать»**. Токен случайный, создаётся при первом запуске.
- `Isola.exe push/update/end/toast` берут токен сами — для них ничего настраивать не нужно.
- Порт можно поменять ключом `server.port` в `%APPDATA%\Isola\settings.json` (по умолчанию 47811). Если порт уже занят (например, Isola другого пользователя Windows на этом же компьютере), берётся следующий свободный из 47812–47820. Фактический порт показан в настройках и записан в `%LOCALAPPDATA%\Isola\server.json` (`{"port": …, "pid": …}`); `Isola.exe` находит его сам.

Безопасность:

- Обращаться к API можно только с этого компьютера (`localhost`). Веб-страницы в браузере не могут ни прочитать токен, ни отправить запрос с ним: заголовок требует предварительного CORS-запроса, а Isola такие запросы отклоняет.
- Токен — это **полный доступ к Isola** (через `/api/rpc` доступно всё, что умеет программа). Обращайтесь с ним как с паролем: не публикуйте в репозиториях и чатах.
- Программы, запущенные под вашей же учётной записью, технически могут прочитать токен из `settings.json` — так же, как они могут прочитать любые ваши файлы. Это граница доверия Windows, а не Isola: не запускайте то, чему не доверяете.
- Сменить токен: закройте Isola, удалите строку `"server.token"` из `%APPDATA%\Isola\settings.json` и запустите снова — создастся новый.

## Методы

| Метод | Тело (JSON) | Что делает |
|---|---|---|
| `GET /api/ping` | — | Проверка, что Isola запущена. Токен не нужен. |
| `POST /api/activity` | активность (см. ниже) | Создать или **заменить** активность с этим `id`. |
| `POST /api/rpc` | `{"method":"activities.update","params":{"id":"…","patch":{…}}}` | Изменить только указанные поля активности. Ответ `"result": false` — такой активности нет. |
| `POST /api/activity/end` | `{"id":"…"}` | Убрать активность. |
| `POST /api/toast` | `{"title":"…","subtitle":"…","icon":"…","color":"#rrggbb","ttlMs":4000}` | Короткое уведомление на острове. |
| `POST /api/open` | `{"tab":"media"}` | Раскрыть остров (необязательно — на нужной вкладке). |

Ответы — JSON: `{"ok": true}`; ошибка токена — HTTP 401, неверное тело — 400.

### Активность

```json
{
  "id": "build",              // обязательно: ваш постоянный id, по нему обновляют и убирают
  "title": "Сборка",
  "subtitle": "проект isola-site",
  "value": "40 %",            // короткое значение справа в свёрнутом острове
  "progress": 0.4,            // 0…1 — кольцо прогресса; -1 — бесконечный индикатор; не указывать — без кольца
  "icon": "hammer",           // имя иконки Lucide (lucide.dev), например timer, download, music, check
  "color": "#0a84ff",
  "priority": 50,             // 0…100: самая важная активность — в острове, следующая — в боковом пузыре
  "alert": false,             // true — раскрыть остров на время ttlMs, как всплывающее уведомление
  "ttlMs": 0,                 // убрать автоматически через столько миллисекунд (0 / не указывать — пока не уберёте)
  "silent": false,            // без анимации «капли» при появлении
  "urgent": false             // показывать даже в «Не беспокоить» и в полноэкранных приложениях
}
```

Кнопки (`actions`) в активностях от скриптов пока не возвращают нажатия обратно — используйте их только во встроенных модулях.

## Примеры

### PowerShell

```powershell
$token = 'ВСТАВЬТЕ_ТОКЕН'
$api = 'http://localhost:47811'
function Isola($path, $body) {
    $json = $body | ConvertTo-Json -Compress
    Invoke-RestMethod -Method Post -Uri "$api$path" -Headers @{ 'X-Isola-Token' = $token } `
        -ContentType 'application/json; charset=utf-8' -Body ([Text.Encoding]::UTF8.GetBytes($json))
}

Isola '/api/activity' @{ id = 'backup'; title = 'Бэкап'; icon = 'hard-drive'; progress = 0 }
foreach ($p in 1..10) {
    Start-Sleep -Seconds 1
    Isola '/api/rpc' @{ method = 'activities.update'; params = @{ id = 'backup'; patch = @{ progress = $p / 10; value = "$($p * 10) %" } } }
}
Isola '/api/activity/end' @{ id = 'backup' }
Isola '/api/toast' @{ title = 'Бэкап готов'; icon = 'check'; color = '#32d74b' }
```

### Python (без сторонних библиотек)

```python
import json, time, urllib.request

TOKEN = "ВСТАВЬТЕ_ТОКЕН"
API = "http://localhost:47811"

def isola(path, body):
    req = urllib.request.Request(API + path, data=json.dumps(body).encode("utf-8"), method="POST",
                                 headers={"X-Isola-Token": TOKEN, "Content-Type": "application/json; charset=utf-8"})
    with urllib.request.urlopen(req, timeout=5) as r:
        return json.loads(r.read() or b"null")

isola("/api/activity", {"id": "train", "title": "Обучение модели", "icon": "brain", "progress": 0})
for epoch in range(1, 11):
    time.sleep(1)
    isola("/api/rpc", {"method": "activities.update",
                       "params": {"id": "train", "patch": {"progress": epoch / 10, "value": f"эпоха {epoch}"}}})
isola("/api/activity/end", {"id": "train"})
isola("/api/toast", {"title": "Модель обучена", "icon": "check"})
```

### Node.js (18+)

```js
const TOKEN = 'ВСТАВЬТЕ_ТОКЕН';
const API = 'http://localhost:47811';

async function isola(path, body) {
  const r = await fetch(API + path, {
    method: 'POST',
    headers: { 'X-Isola-Token': TOKEN, 'Content-Type': 'application/json; charset=utf-8' },
    body: JSON.stringify(body),
  });
  if (!r.ok) throw new Error(`Isola: HTTP ${r.status}`);
  return r.json();
}

await isola('/api/activity', { id: 'deploy', title: 'Деплой', icon: 'rocket', progress: -1 });
// … работа …
await isola('/api/activity/end', { id: 'deploy' });
await isola('/api/toast', { title: 'Задеплоено', subtitle: 'prod', icon: 'check', color: '#32d74b' });
```

## Командная строка

`Isola.exe` лежит в `%LOCALAPPDATA%\Programs\Isola`. Команды обращаются к уже запущенной Isola и сразу завершаются (код 0 — успех, 1 — Isola не запущена, 2 — нет такой активности).

```bat
Isola.exe push --id build --title "Сборка" --value 0% --progress 0 --icon hammer
Isola.exe update --id build --progress 0.7 --value 70%
Isola.exe end --id build
Isola.exe toast "Готово" --subtitle "Тесты прошли" --icon check
Isola.exe open media
```

`push` понимает `--id --title --subtitle --value --progress --icon --color --kind --ttl --priority --alert`;
`update` — `--id` и любые из `--title --subtitle --value --progress --icon --color` (меняются только указанные).

## Если не работает

- `GET http://localhost:47811/api/ping` в браузере должен ответить `{"ok":true,…}`. Нет ответа — Isola не запущена или порт занят другой программой (смените `server.port`).
- HTTP 401 — неверный токен (скопируйте заново в настройках).
- Кириллица превратилась в «????» — отправляйте тело в UTF-8 (в примере PowerShell это уже сделано).
