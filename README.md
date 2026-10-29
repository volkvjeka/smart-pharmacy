## Руководство по развертыванию «Умная домашняя аптечка» (Self-Hosted)

---

> **Назначение:** Полная техническая спецификация для развертывания персонального ассистента в семейной беседе ВКонтакте для сквозного учета домашних медикаментов, голосового ввода, контроля сроков годности (FEFO) и фармацевтической сверки списков лекарств.

---

## 📑 Содержание

1. **Архитектура и сетевая безопасность**
2. **Подготовка API-ключей и шлюза**
   * 2.1. ВКонтакте Callback API
   * 2.2. GigaChat API
   * 2.3. Внешний Reverse Proxy (Traefik)
3. **Развертывание Docker-сервисов**
   * 3.1. Сеть и файловая структура
   * 3.2. Стек СУБД: PostgreSQL 16 и NocoDB
   * 3.3. Оркестратор n8n и защита NO_PROXY
   * 3.4. Локальный ИИ: Whisper STT (и опционально Ollama)
4. **Схема базы данных PostgreSQL (`pharmacy`)**
5. **Конфигурация сценариев n8n**
   * 5.1. Учетные записи (Credentials)
   * 5.2. Сценарий 1: `aptecka-vk-main` (Входящий шлюз)
   * 5.3. Сценарий 2: `aptecka-llm-dispatcher` (Очередь и бизнес-логика)
6. **Автоматическое обслуживание и резервное копирование**
7. **Устранение неполадок (Troubleshooting)**

---

## 1. Архитектура и сетевая безопасность

Сервис разворачивается на домашнем сервере / VPS в изолированных Docker-контейнерах:

```text
[ Семейная беседа VK ] 
       │ (текст / голосовые / фото)
       ▼
[ VK Callback API ] ────► HTTPS POST (порт 443)
       │
       ▼
[ Reverse Proxy (Traefik / Nginx) ] ──(наружу открыт ТОЛЬКО /webhook/vk)
       │ HTTP (порт 5678)
       ▼
[ Docker-сеть "proxy" ]
   ├── [ n8n ] (Оркестрация конвейера)
   │     ├── Служебная БД ────► postgres:5432/n8n
   │     └── База аптечки ────► postgres:5432/pharmacy
   │
   ├── [ PostgreSQL 16 ] (Изолированные БД: pharmacy, n8n, nocodb)
   ├── [ NocoDB ] (Web-интерфейс к базе; доступ строго из LAN)
   ├── [ Whisper ASR ] (Локальный перевод голосовых в текст, faster-whisper)
   ├── [ Ollama ] (Опционально: OCR упаковок; порт 11434 внутри сети)
   └── [ GigaChat API ] (Облачная языковая модель Сбера для анализа симптомов)
```

### Принципы безопасности:
1. **Минимальная поверхность атаки:** Из интернета доступен **исключительно** маршрут `POST /webhook/vk`. Все остальные веб-интерфейсы (n8n, NocoDB, СУБД) закрыты внутри локальной сети (LAN).
2. **Медицинский Guardrail (Уровень 0):** Проверка жизнеугрожающих состояний (кровотечения, загрудинная боль, удушье, судороги, сверхвысокая температура) выполняется жестким регулярным выражением **до вызова нейросетей**. При обнаружении выдается директива звонить в **103/112**, работа конвейера принудительно останавливается.
3. **Анонимизация (ПДн):** Перед отправкой текста в облачную LLM скрипт маскирует номера телефонов (`[PHONE]`), возраст (`[AGE]`) и ФИО (`[NAME]`).
4. **Контроль очереди:** Запросы к LLM сериализуются через таблицу `llm_requests` с механизмом `FOR UPDATE SKIP LOCKED`, что исключает лимиты `429 Too Many Requests`.

---

## 2. Подготовка API-ключей и шлюза

### 2.1. ВКонтакте: Настройка сообщества и Callback API
1. Создайте закрытую группу ВКонтакте.
2. В разделе **Управление** ➔ **Работа с API**:
   * Создайте ключ доступа с правами доступа к сообщениям сообщества (`VK_GROUP_TOKEN`).
   * Во вкладке **Callback API**:
     * Версия API: строго **`5.199`**.
     * Скопируйте строку подтверждения (`VK_CONFIRMATION_CODE`).
     * Введите надежный секретный ключ из букв и цифр (`VK_SECRET`).
     * Во вкладке **Типы событий** отметьте **ТОЛЬКО** пункт: *«Входящие сообщения» (`message_new`)*.
3. В разделе **Управление** ➔ **Сообщения**:
   * Включите сообщения и в **Настройках для бота** активируйте возможности бота и чекбокс *«Разрешать добавлять сообщество в беседы»*.
4. Добавьте бота в семейную беседу и **обязательно назначьте его Администратором беседы** (иначе ВК блокирует передачу сообщений боту без прямого упоминания).

### 2.2. GigaChat API
1. Авторизуйтесь на портале [developers.sber.ru](https://developers.sber.ru/) (Сбер ID).
2. Создайте проект и подключите сервис **GigaChat API**.
3. Скопируйте **Авторизационные данные** (строка Client Secret / Base64).

### 2.3. Внешний Reverse Proxy (Пример Traefik dynamic yml)
Конфигурация маршрутизации (замените `<YOUR_DOMAIN>` и `<SERVER_LOCAL_IP>`):

```yaml
http:
  routers:
    n8n-vk-webhook:
      rule: "Host(`<YOUR_DOMAIN>`) && (Path(`/webhook/vk`) || Path(`/webhook/vk/`))"
      entryPoints: ["websecure"]
      service: n8n-service
      tls:
        certResolver: le
      priority: 30

    n8n-lan-ui:
      rule: "Host(`<YOUR_DOMAIN>`)"
      entryPoints: ["websecure"]
      service: n8n-service
      middlewares: ["lan-whitelist"]
      tls:
        certResolver: le
      priority: 10

  middlewares:
    lan-whitelist:
      ipAllowList:
        sourceRange:
          - "192.168.0.0/16"
          - "10.0.0.0/8"
          - "172.16.0.0/12"

  services:
    n8n-service:
      loadBalancer:
        servers:
          - url: "http://<SERVER_LOCAL_IP>:5678"
```

---

## 3. Развертывание Docker-сервисов

### 3.1. Подготовка каталогов
```bash
docker network create proxy || true

mkdir -p /opt/stacks/db/initdb
mkdir -p /opt/stacks/n8n/local-files
mkdir -p /opt/stacks/whisper/whisper_cache
mkdir -p /opt/stacks/ollama/ollama_data
mkdir -p /opt/scripts
mkdir -p /opt/backup/daily /opt/backup/weekly

# Назначение прав для n8n (UID 1000)
chown -R 1000:1000 /opt/stacks/n8n/local-files
```

### 3.2. Стек 1: PostgreSQL 16 и NocoDB
Файл `/opt/stacks/db/.env`:
```bash
POSTGRES_PASSWORD=<STRONG_POSTGRES_PASSWORD>
N8N_DB_USER=n8n_user
N8N_DB_PASS=<STRONG_N8N_PASSWORD>
PHARMACY_DB_USER=pharmacy_app
PHARMACY_DB_PASS=<STRONG_PHARMACY_PASSWORD>
NOCODB_DB_USER=nocodb_user
NOCODB_DB_PASS=<STRONG_NOCODB_PASSWORD>
```

Скрипт инициализации `/opt/stacks/db/initdb/01-init.sql`:
```sql
\set ON_ERROR_STOP on

CREATE DATABASE n8n;
CREATE DATABASE nocodb;
CREATE DATABASE pharmacy;

CREATE USER n8n_user WITH ENCRYPTED PASSWORD '<STRONG_N8N_PASSWORD>';
CREATE USER nocodb_user WITH ENCRYPTED PASSWORD '<STRONG_NOCODB_PASSWORD>';
CREATE USER pharmacy_app WITH ENCRYPTED PASSWORD '<STRONG_PHARMACY_PASSWORD>';

ALTER DATABASE n8n OWNER TO n8n_user;
ALTER DATABASE nocodb OWNER TO nocodb_user;
ALTER DATABASE pharmacy OWNER TO pharmacy_app;

GRANT ALL PRIVILEGES ON DATABASE n8n TO n8n_user;
GRANT ALL PRIVILEGES ON DATABASE nocodb TO nocodb_user;
GRANT ALL PRIVILEGES ON DATABASE pharmacy TO pharmacy_app;
```

Файл `/opt/stacks/db/docker-compose.yml`:
```yaml
networks:
  proxy:
    external: true

services:
  postgres:
    image: postgres:16-alpine
    container_name: postgres
    restart: unless-stopped
    networks:
      - proxy
    environment:
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      TZ: Europe/Moscow
      PGTZ: Europe/Moscow
    volumes:
      - ./postgres_data:/var/lib/postgresql/data
      - ./initdb:/docker-entrypoint-initdb.d:ro
    ports:
      - "127.0.0.1:5432:5432"
    deploy:
      resources:
        limits:
          memory: 2G
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

  nocodb:
    image: nocodb/nocodb:latest
    container_name: nocodb
    restart: unless-stopped
    networks:
      - proxy
    depends_on:
      postgres:
        condition: service_healthy
    environment:
      - NC_DB=pg://postgres:5432?u=${NOCODB_DB_USER}&p=${NOCODB_DB_PASS}&d=nocodb
      - NC_ALLOW_LOCAL_EXTERNAL_DBS=true
    volumes:
      - ./nocodb_data:/usr/app/data
    ports:
      - "127.0.0.1:8080:8080"
```

Запуск СУБД:
```bash
cd /opt/stacks/db && docker compose up -d
```

### 3.3. Стек 2: Оркестратор n8n
Файл `/opt/stacks/n8n/.env`:
```bash
DOMAIN_NAME=<YOUR_DOMAIN>
N8N_DB_USER=n8n_user
N8N_DB_PASS=<STRONG_N8N_PASSWORD>
# Генерация случайного ключа: openssl rand -hex 32
N8N_ENC_KEY=<YOUR_32_BYTE_HEX_ENCRYPTION_KEY>
```

Файл `/opt/stacks/n8n/docker-compose.yml`:
```yaml
networks:
  proxy:
    external: true

services:
  n8n:
    image: docker.n8n.io/n8nio/n8n:latest
    container_name: n8n
    restart: unless-stopped
    networks:
      - proxy
    ports:
      - "5678:5678"
    environment:
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_HOST=postgres
      - DB_POSTGRESDB_PORT=5432
      - DB_POSTGRESDB_DATABASE=n8n
      - DB_POSTGRESDB_USER=${N8N_DB_USER}
      - DB_POSTGRESDB_PASSWORD=${N8N_DB_PASS}
      - N8N_HOST=${DOMAIN_NAME}
      - N8N_PROTOCOL=https
      - N8N_WEBHOOK_URL=https://${DOMAIN_NAME}
      - N8N_PROXY_HOPS=1
      - GENERIC_TIMEZONE=Europe/Moscow
      - TZ=Europe/Moscow
      - N8N_ENCRYPTION_KEY=${N8N_ENC_KEY}
      - EXECUTIONS_DATA_PRUNE=true
      - EXECUTIONS_DATA_MAX_AGE=48
      # Защита от 502 Bad Gateway при наличии внешнего прокси:
      - NO_PROXY=localhost,127.0.0.1,::1,172.16.0.0/12,172.17.0.0/16,172.18.0.0/16,10.0.0.0/8,192.168.0.0/16,whisper,ollama,postgres,nocodb,.local
    volumes:
      - ./n8n_data:/home/node/.n8n
      - ./local-files:/files
    deploy:
      resources:
        limits:
          memory: 3G
```

Запуск n8n:
```bash
cd /opt/stacks/n8n && docker compose up -d
```

### 3.4. Стеки локального ИИ: Whisper (STT) и Ollama (Опционально)

#### Whisper STT (`/opt/stacks/whisper/docker-compose.yml`):
```yaml
networks:
  proxy:
    external: true

services:
  whisper:
    image: onerahmet/openai-whisper-asr-webservice:latest
    container_name: whisper
    restart: unless-stopped
    networks:
      - proxy
    environment:
      - ASR_MODEL=small
      - ASR_ENGINE=faster_whisper
    volumes:
      - ./whisper_cache:/root/.cache
    deploy:
      resources:
        limits:
          memory: 3G
```

#### Ollama (`/opt/stacks/ollama/docker-compose.yml`) — *Экспериментально:*
> 💡 *Примечание:* Для ввода лекарств значительно удобнее использовать голосовые сообщения. Модуль Ollama опционален и требует дополнительных ресурсов CPU/GPU.
```yaml
networks:
  proxy:
    external: true

services:
  ollama:
    image: ollama/ollama:latest
    container_name: ollama
    restart: unless-stopped
    networks:
      - proxy
    environment:
      - OLLAMA_KEEP_ALIVE=5m
    volumes:
      - ./ollama_data:/root/.ollama
    deploy:
      resources:
        limits:
          memory: 6G
```
Запуск сервисов:
```bash
cd /opt/stacks/whisper && docker compose up -d
# Если требуется Ollama:
# cd /opt/stacks/ollama && docker compose up -d && docker exec -it ollama ollama pull qwen2.5-vl:3b
```

---

## 4. Схема базы данных PostgreSQL (`pharmacy`)

Подключитесь к СУБД и создайте таблицы:

```bash
docker exec -i postgres psql -U postgres -d pharmacy << 'EOF'
\set ON_ERROR_STOP on

CREATE EXTENSION IF NOT EXISTS pg_trgm;

GRANT ALL ON SCHEMA public TO pharmacy_app;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT ALL ON TABLES TO pharmacy_app;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT ALL ON SEQUENCES TO pharmacy_app;

CREATE OR REPLACE FUNCTION touch_updated_at()
RETURNS trigger AS $$
BEGIN
  NEW.updated_at = now();
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- 1. Реестр медикаментов
CREATE TABLE IF NOT EXISTS medicine (
  id           bigserial PRIMARY KEY,
  title        text NOT NULL,
  title_norm   text GENERATED ALWAYS AS (lower(btrim(title))) STORED,
  purpose      text,
  purpose_src  text NOT NULL DEFAULT 'user' CHECK (purpose_src IN ('user','ai')),
  expiry_date  date,
  expiry_key   date GENERATED ALWAYS AS (COALESCE(expiry_date, DATE '9999-12-31')) STORED,
  expiry_label text,
  qty          int NOT NULL DEFAULT 1 CHECK (qty >= 0),
  status       text NOT NULL DEFAULT 'in_stock' CHECK (status IN ('in_stock','out_of_stock')),
  added_by     text,
  photo_ref    text,
  search_tsv   tsvector GENERATED ALWAYS AS (
                 setweight(to_tsvector('russian', coalesce(title,'')), 'A') ||
                 setweight(to_tsvector('russian', coalesce(purpose,'')), 'B')
               ) STORED,
  created_at   timestamptz NOT NULL DEFAULT now(),
  updated_at   timestamptz NOT NULL DEFAULT now()
);

CREATE UNIQUE INDEX IF NOT EXISTS medicine_uniq ON medicine (title_norm, expiry_key);
CREATE INDEX IF NOT EXISTS idx_medicine_title_norm_trgm ON medicine USING gin (title_norm gin_trgm_ops);
CREATE INDEX IF NOT EXISTS idx_medicine_tsv ON medicine USING gin (search_tsv);
CREATE INDEX IF NOT EXISTS idx_medicine_status ON medicine (status);
CREATE INDEX IF NOT EXISTS idx_medicine_expiry ON medicine (expiry_date);

DROP TRIGGER IF EXISTS trg_medicine_touch ON medicine;
CREATE TRIGGER trg_medicine_touch
  BEFORE UPDATE ON medicine
  FOR EACH ROW EXECUTE FUNCTION touch_updated_at();

-- 2. Дедупликация вебхуков
CREATE TABLE IF NOT EXISTS processed_events (
  event_id    text PRIMARY KEY,
  received_at timestamptz NOT NULL DEFAULT now()
);

-- 3. Двухфазные подтверждения
CREATE TABLE IF NOT EXISTS pending_actions (
  id           bigserial PRIMARY KEY,
  peer_id      bigint NOT NULL,
  from_id      bigint NOT NULL,
  action       text NOT NULL CHECK (action IN ('confirm_expiry','bulk_writeoff','pick_item','ask_expiry','add_medicine')),
  payload      jsonb NOT NULL,
  expires_at   timestamptz NOT NULL DEFAULT now() + interval '15 minutes',
  completed_at timestamptz,
  created_at   timestamptz NOT NULL DEFAULT now()
);

CREATE INDEX IF NOT EXISTS idx_pending_lookup ON pending_actions (peer_id, from_id, completed_at, expires_at);

-- 4. Очередь задач LLM
CREATE TABLE IF NOT EXISTS llm_requests (
  id                  bigserial PRIMARY KEY,
  peer_id             bigint NOT NULL,
  from_id             bigint NOT NULL,
  message_id          bigint,
  event_id            text,
  kind                text NOT NULL DEFAULT 'pipeline',
  status              text NOT NULL DEFAULT 'queued' CHECK (status IN ('queued','processing','done','failed')),
  input               jsonb NOT NULL,
  output              jsonb,
  error               text,
  attempts            int NOT NULL DEFAULT 0,
  max_attempts        int NOT NULL DEFAULT 3,
  next_attempt_at     timestamptz NOT NULL DEFAULT now(),
  started_at          timestamptz,
  heartbeat_at        timestamptz,
  finished_at         timestamptz,
  runner_execution_id text,
  created_at          timestamptz NOT NULL DEFAULT now(),
  updated_at          timestamptz NOT NULL DEFAULT now()
);

CREATE INDEX IF NOT EXISTS idx_llm_requests_claim ON llm_requests (id ASC) WHERE status = 'queued';

ALTER TABLE medicine OWNER TO pharmacy_app;
ALTER TABLE processed_events OWNER TO pharmacy_app;
ALTER TABLE pending_actions OWNER TO pharmacy_app;
ALTER TABLE llm_requests OWNER TO pharmacy_app;
EOF
```

---

## 5. Конфигурация сценариев n8n

В n8n перейдите в **Settings** ➔ **Community Nodes** ➔ установите ноду `n8n-nodes-gigachat`.

### 5.1. Учетные записи (Credentials)
1. **`Postgres Pharmacy DB`** (Postgres):
   * Host: `postgres`, DB: `pharmacy`, User: `pharmacy_app`, Password: `<STRONG_PHARMACY_PASSWORD>`, Port: `5432`, SSL: `disable`.
2. **`VK API Auth`** (Query Auth):
   * Name: `access_token` *(строго строчные буквы)*
   * Value: `<VK_GROUP_TOKEN>`
3. **`GigaChat Account`** (GigaChat API):
   * Authorization Key: строка Client Secret / Base64.
   * Scope: `GIGACHAT_API_PERS`.

---

### 5.2. Сценарий 1: `aptecka-vk-main` (Входящий шлюз)

```text
[Webhook: /webhook/vk] ──► [Switch: Type]
                              ├── confirmation ──► [Respond: <VK_CONFIRMATION_CODE>]
                              └── message_new  ──► [If: Secret == <VK_SECRET>] ──► [Respond: "ok"] ──► [Postgres: Dedupe]
                                                                                                              │
                                                                                                              ▼
                                                                                                    [Switch: media_type]
                                                                                                     ├── audio ──► [HTTP Download] ──► [Whisper STT] ──┐
                                                                                                     └── text ─────────────────────────────────────────┤
                                                                                                                                                       ▼
                                                                                                                                           [Code: NormalizeText]
                                                                                                                                                       │
                                                                                                                                           🔴 [Code: Emergency Guardrail]
                                                                                                                                                       │
                                                                                                                                                [If: Опасность?]
                                                                                                                                                 ├── true  ──► [VK Send: 103/112]
                                                                                                                                                 └── false ──► [Code: Маскирование ПДн]
                                                                                                                                                                     │
                                                                                                                                                               [Postgres: Запись очереди]
                                                                                                                                                                     │
                                                                                                                                                               [Sub-wf: aptecka-llm-dispatcher]
```

#### Ключевые узлы Сценария 1:

1. **Dedupe (Postgres):**
   ```sql
   INSERT INTO processed_events (event_id)
   VALUES ($1)
   ON CONFLICT (event_id) DO NOTHING
   RETURNING event_id;
   ```
   *Параметры:* `{{ [ String($('Webhook').item.json.body.event_id || $('Webhook').item.json.body.object.message.conversation_message_id) ] }}`

2. **Whisper STT (HTTP Request):**
   * Method: `POST`
   * URL: `http://whisper:9000/asr?language=ru&task=transcribe&output=text`
   * Body: `Form-Data`
   * Parameter: Type `n8n Binary File`, Name: `audio_file`, Input Field: `data`.

3. 🔴 **Emergency Guardrail (Code):**
   ```javascript
   const item = $input.item.json;
   const text = item.text || '';

   const emergencyRegex = /(кровотечени|кровь (?:при кашле|в мокроте|в рвоте|из носа|из желудка)|кровянист|давит в груди|жжёт в груди|горит в груди|боль в груди|боль за грудиной|удуш|не могу дышать|трудно дышать|одышк|отёк|отек лица|отек горла|анафилакс|потерял сознание|потеря сознания|обморок|судорог|конвульс|передоз|отравлени|температура (?:39|40|41)|высокая температура у ребёнк|лающий кашель|звёздчатая сыпь)/i;

   if (emergencyRegex.test(text)) {
     return [{
       json: {
         is_emergency: true,
         peer_id: item.peer_id,
         reply: "⚠️ Это состояние может быть опасным для жизни! Срочно вызовите скорую помощь: 103 или 112."
       }
     }];
   }

   return [{
     json: {
       is_emergency: false,
       text: text,
       peer_id: item.peer_id,
       from_id: item.from_id,
       msg_id: item.msg_id
     }
   }];
   ```

4. **Маскирование ПДн (Code):**
   ```javascript
   let text = $json.text || '';
   text = text.replace(/(?:\+?7|8)[\s(-]*\d{3}[\s)-]*\d{3}[\s-]*\d{2}[\s-]*\d{2}/g, '[PHONE]');
   text = text.replace(/\b\d+\s*(?:лет|года|год|месяц|месяца|мес\.?)\b/gi, '[AGE]');
   text = text.replace(/\b[А-ЯЁ][а-яё]+\s+[А-ЯЁ][а-яё]+(?:\s+[А-ЯЁ][а-яё]+)?\b/g, '[NAME]');

   return [{
     json: {
       masked_text: text,
       peer_id: $json.peer_id,
       from_id: $json.from_id,
       msg_id: $json.msg_id
     }
   }];
   ```

5. **Постановка в очередь (Postgres):**
   ```sql
   INSERT INTO llm_requests (peer_id, from_id, message_id, input)
   VALUES ($1, $2, $3, jsonb_build_object('text', $4::text))
   RETURNING id;
   ```
   *Параметры:* `{{ [$json.peer_id, $json.from_id, $json.msg_id, $json.masked_text] }}`

---

### 5.3. Сценарий 2: `aptecka-llm-dispatcher` (Бизнес-логика)

```text
[Schedule: 10 сек] / [Sub-workflow Trigger]
                     │
                     ▼
           [Postgres: ClaimTask (FOR UPDATE SKIP LOCKED)]
                     │
             [If: Задача есть?]
                     │ true
                     ▼
           [GigaChat: Классификатор (Temperature 0)]
                     │
           [Code: ParseLLM & Валидация дат]
                     │
             [Switch: intent]
 ├── search ───► [Postgres Поиск / Рецепт] ──► [Code Сверка] ────────┐
 ├── add ──────► [Postgres UPSERT] ──────────► [Code Форматирование] ─┤
 ├── writeoff ─► [Postgres Списание FEFO] ───► [Code Форматирование] ─┼──► [HTTP: VK messages.send] ──► [Postgres: Done]
 ├── expiring ─► [Postgres Контроль 30 дней] ─► [Code Форматирование] ─┤
 └── Fallback ─► [Code Справка] ─────────────────────────────────────┘
```

#### Ключевые узлы Сценария 2:

1. **ClaimTask (Postgres):**
   ```sql
   WITH claimed_task AS (
     SELECT id
     FROM llm_requests
     WHERE (status = 'queued' AND next_attempt_at <= now())
        OR (status = 'processing' AND started_at < now() - interval '5 minutes' AND attempts < max_attempts)
     ORDER BY id ASC
     LIMIT 1
     FOR UPDATE SKIP LOCKED
   )
   UPDATE llm_requests r
   SET status = 'processing',
       started_at = now(),
       attempts = r.attempts + 1,
       runner_execution_id = $1
   FROM claimed_task
   WHERE r.id = claimed_task.id
   RETURNING r.*;
   ```
   *Параметр:* `{{ [ String($execution.id) ] }}`

2. **GigaChat System Prompt:**
   ````text
   Сегодняшняя дата: {{ $now.toFormat('yyyy-MM-dd') }}.
Ты — фармацевт-эксперт. Строго классифицируй запрос и верни JSON без разметки markdown:
{
  "intent": "search" | "add" | "writeoff" | "expiring" | "other",
  "title": "название или ALL или null",
  "expiry": "MM.YYYY или null",
  "purpose": "от чего помогает",
  "symptoms": ["симптом1"],
  "checklist": [{"name": "Препарат", "aliases": ["аналог1", "МНН"]}]
}
Правила:
- При добавлении ("add") текстовые сроки переводи в цифры (например, "до мая 2028" -> "05.2028").
- При сверке списка назначений ("search") заполни checklist аналогами и синонимами.
- "Что есть в аптечке" -> intent: "search", title: "ALL".
   ````

3. **Списание по правилу FEFO (Ветка `writeoff`):**
   ```sql
   WITH target_batch AS (
     SELECT id
     FROM medicine
     WHERE status = 'in_stock'
       AND title_norm = lower(btrim($1))
     ORDER BY expiry_date ASC NULLS LAST, id ASC
     LIMIT 1
     FOR UPDATE
   )
   UPDATE medicine m
   SET qty = GREATEST(0, qty - 1),
       status = CASE WHEN qty - 1 <= 0 THEN 'out_of_stock' ELSE 'in_stock' END
   FROM target_batch
   WHERE m.id = target_batch.id
   RETURNING m.title, m.expiry_label, m.qty, m.status;
   ```
   *Параметр:* `{{ [ $('ParseLLM').item.json.parsed.title || '' ] }}`

4. **Отправка ответа в чат (HTTP Request к API ВКонтакте):**
   * Method: `POST`
   * URL: `https://api.vk.com/method/messages.send`
   * Auth: `Generic Credential Type` ➔ `Query Auth` ➔ `VK API Auth`
   * Body Parameters (Form Urlencoded):
     * `v`: `5.199`
     * `peer_id`: `{{ $('ClaimTask').item.json.peer_id }}`
     * `random_id`: `{{ Math.floor(Math.random() * 2147483647) }}`
     * `message`: `{{ $json.reply }}` *(без пробелов в названии ключа!)*

---

## 6. Автоматическое обслуживание и бэкапы

### 6.1. Резервное копирование (`/opt/scripts/backup.sh`)
```bash
cat << 'EOF' > /opt/scripts/backup.sh
#!/usr/bin/env bash
set -euo pipefail

BACKUP_DIR="/opt/backup"
DATE="$(date +%F)"
DOW="$(date +%u)"

docker exec -e PGPASSWORD="<STRONG_POSTGRES_PASSWORD>" postgres \
  pg_dump -U postgres -Fc pharmacy > "$BACKUP_DIR/daily/pharmacy_${DATE}.dump"

docker exec -e PGPASSWORD="<STRONG_POSTGRES_PASSWORD>" postgres \
  pg_dump -U postgres -Fc n8n > "$BACKUP_DIR/daily/n8n_${DATE}.dump"

if [ "$DOW" = "7" ]; then
  cp "$BACKUP_DIR/daily/pharmacy_${DATE}.dump" "$BACKUP_DIR/weekly/pharmacy_${DATE}.dump"
fi

find "$BACKUP_DIR/daily" -type f -name '*.dump' -mtime +14 -delete
find "$BACKUP_DIR/weekly" -type f -name '*.dump' -mtime +56 -delete
EOF

chmod +x /opt/scripts/backup.sh
```

### 6.2. Очистка очередей (`/opt/scripts/cleanup.sh`)
```bash
cat << 'EOF' > /opt/scripts/cleanup.sh
#!/usr/bin/env bash
docker exec -i postgres psql -U pharmacy_app -d pharmacy << 'SQL'
DELETE FROM processed_events WHERE received_at < now() - interval '7 days';
DELETE FROM pending_actions WHERE expires_at < now() OR completed_at < now() - interval '1 day';
DELETE FROM llm_requests WHERE status IN ('done','failed') AND finished_at < now() - interval '7 days';
SQL
EOF

chmod +x /opt/scripts/cleanup.sh
```

Настройка cron (`crontab -e`):
```cron
0 3 * * * /opt/scripts/backup.sh >> /var/log/aptecka-backup.log 2>&1
0 4 * * * /opt/scripts/cleanup.sh >> /var/log/aptecka-cleanup.log 2>&1
```

---

## 7. Устранение неполадок (Troubleshooting)

| Проблема | Причина | Решение |
| :--- | :--- | :--- |
| **Бот не отвечает в групповом чате** | Бот не назначен Администратором беседы. | ВК не передает входящие боту без прав администратора. Сделайте бота админом беседы. |
| **Ошибка 502 при запросах к Whisper/DB** | Переменные HTTP_PROXY перенаправляют локальные DNS Docker во внешний прокси. | Убедитесь, что в n8n задана переменная `NO_PROXY` со списком имен контейнеров (`whisper,postgres`) и подсетью `172.18.0.0/16`. |
| **Ошибка ВК 100: `message is invalid`** | Случайный пробел в названии параметра `message `. | Проверьте ноду отправки в ВК: название ключа должно быть строго `message` без хвостовых пробелов. |
| **Ошибка Whisper: `no binary field 'data'`** | В поле имени файла остался серый текст плейсхолдера. | Кликните в поле ввода и вручную напечатайте слово `data` (шрифт должен стать темным). |
| **Бот дублирует ответы** | n8n слишком долго отвечает на вебхук ВК (> 5 сек). | Убедитесь, что нода `Respond to Webhook` с телом `ok` срабатывает **до** вызова длительных операций. |
