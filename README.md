# abuse_cleandns — Конфигуратор задач CleanDNS

Веб-форма + Cloudflare Worker для управления параметрами отправки abuse-репортов.  
Данные хранятся в Airtable; форма позволяет выбирать или добавлять записи прямо из интерфейса.

---

## Как это работает

```
Пользователь (index.html)
    │  выбирает email / прокси / URL / описание / бренд
    ▼
Cloudflare Worker  (worker.js)
    │  хранит токен Airtable как секрет, проксирует запросы
    ▼
Airtable
    │  возвращает списки, сохраняет новые записи и задачи
    ▼
Бот (отдельный скрипт, на этапе запуска)
    │  берёт задачу со статусом "pending"
    ▼
CleanDNS API
```

---

## Файлы

| Файл | Назначение |
|---|---|
| `index.html` | Форма (Tabler UI, русский интерфейс) |
| `worker.js` | Cloudflare Worker — прокси к Airtable |
| `wrangler.toml` | Конфиг деплоя Cloudflare |
| `.gitignore` | Исключения git (секреты, node_modules) |

---

## Таблицы Airtable

| Ключ | ID таблицы | Поля |
|---|---|---|
| emails | `tbllVtFoicvxeMD0b` | email, active |
| urls | `tblBliXmWrr1fxZh7` | offending_url, active |
| descs | `tblVj4cUtdrwu6TLR` | description, headers, body, active |
| brands | `tblWeS58LPUlbkml0` | brand_or_company, active |
| proxies | `tblPNR6wFYrgMbFNO` | proxy, active |
| screenshots | `tblNKa0EiH1rFFpKc` | filename, full_path, active |
| results_log | `tblXJ1wKsqpQJOKmj` | (существующая) |
| **tasks** | **создать вручную** | см. ниже |

### Таблица Tasks (создать в Airtable)

Поля:

| Поле | Тип Airtable |
|---|---|
| email_id | Single line text |
| email_val | Single line text |
| proxy_id | Single line text |
| proxy_val | Single line text |
| url_id | Single line text |
| url_val | Single line text |
| desc_id | Single line text |
| desc_val | Long text |
| brand_id | Single line text |
| brand_val | Single line text |
| screenshot_id | Single line text |
| screenshot_val | Single line text |
| status | Single select: `pending` / `running` / `done` / `error` |
| created_at | Date & time |

После создания вставьте ID таблицы в `worker.js`:
```js
tasks: "tblXXXXXXXXXXXXXX",
```

---

## Деплой Worker

```bash
# 1. Установить wrangler
npm install -g wrangler

# 2. Войти в Cloudflare
wrangler login

# 3. Добавить секреты
wrangler secret put AIRTABLE_TOKEN
# → введите: pat...  (Personal Access Token из Airtable)

wrangler secret put AIRTABLE_BASE
# → введите: apppqstqaEJT6QlKJ

# 4. Задеплоить
wrangler deploy
```

Результат: `https://cleandns-form-worker.ВАШ-СУБДОМЕН.workers.dev`

Через Cloudflare Dashboard без CLI:
1. Workers & Pages → Create → Worker
2. Вставить `worker.js`
3. Settings → Variables → Secrets: `AIRTABLE_TOKEN`, `AIRTABLE_BASE`, `ALLOWED_ORIGIN`

---

## Настройка формы

В `index.html` найти и заменить:

```js
const WORKER_URL = "https://YOUR-WORKER.YOUR-SUBDOMAIN.workers.dev";
```

---

## Размещение формы

Подойдёт любой хостинг статики:

- **Cloudflare Pages** (рекомендуется)  
  ```bash
  # из папки с index.html
  wrangler pages deploy .
  ```
- GitHub Pages (Settings → Pages → Deploy from branch)
- Netlify / Vercel — drag & drop папки

---

## Проверка через curl

```bash
W="https://cleandns-form-worker.ВАШ.workers.dev"

# Health
curl "$W/api/health"

# Все данные из Airtable
curl "$W/api/data" | python3 -m json.tool

# Добавить email
curl -X POST "$W/api/add-record" \
  -H "Content-Type: application/json" \
  -d '{"table":"emails","fields":{"email":"test@example.com","active":true}}'

# Добавить прокси
curl -X POST "$W/api/add-record" \
  -H "Content-Type: application/json" \
  -d '{"table":"proxies","fields":{"proxy":"1.2.3.4:1080:user:pass","active":true}}'

# Сохранить задачу
curl -X POST "$W/api/submit-task" \
  -H "Content-Type: application/json" \
  -d '{
    "email_id":  "recXXXXXXXXXXXXXX",
    "url_id":    "recYYYYYYYYYYYYYY",
    "desc_id":   "recZZZZZZZZZZZZZZ",
    "brand_id":  "recAAAAAAAAAAAAA",
    "proxy_id":  "recBBBBBBBBBBBBBB"
  }'
```

---

## Безопасность

- Токен Airtable хранится **только** в секретах Worker — браузер его не видит
- CORS ограничен через `ALLOWED_ORIGIN` (поставьте точный домен формы, не `*`, в продакшене)
- Worker не открывает доступ к произвольным таблицам — только к перечисленным в `TABLES`

---

## Связанные репозитории

- Основной бот-отправитель: [550953/formtgtest](https://github.com/550953/formtgtest)
- Пример работающей формы: [form.shikinn.com/seo-matrasy](https://form.shikinn.com/seo-matrasy/)
