# SSO_IAM_Keycloak_DataPlatform

Инженерный проект по проектированию и практической реализации защищенного периметра управления идентификацией и доступом (SSO / IAM) для платформы телеметрии и аналитики.

Решение демонстрирует сквозную архитектуру защиты конфиденциальных данных:
- централизованное управление жизненным циклом учетных записей и ролевой моделью (RBAC) на базе Keycloak;
- защита публичных клиентов от компрометации авторизационного кода через протокол OAuth 2.0 PKCE (S256);
- криптографическая бессерверная валидация асимметричной подписи JWT (RS256) на уровне сервисов данных;
- реализация принципа Zero-Trust к параметрам клиентских запросов для исключения несанкционированного доступа к чужим записям.

---

## 1. Архитектурный периметр безопасности

Платформа реализует концепцию **Zero-Trust к входящим пользовательским параметрам**:
- **Принудительный PKCE (RFC 7636):** Браузерное приложение (React SPA) взаимодействует с Keycloak исключительно с криптографическим подтверждением ключа (`pkceMethod: 'S256'`). Прямой парольный вход (`directAccessGrantsEnabled`) отключен.
- **Бессерверная валидация JWT (RS256):** Reporting API (FastAPI) валидирует асимметричную цифровую подпись токена через публичные ключи Keycloak (JWKS) без повторных сетевых запросов к IdP.
- **Zero-Trust изоляция на уровне данных (Claims Filtering):** Параметры `user_id`, передаваемые клиентом в query-строке, полностью игнорируются. Микросервис извлекает идентификатор субъекта строго из клейма `sub` проверенного JWT и выполняет параметризованный запрос к ClickHouse (`WHERE user_id = :token_sub`), предотвращая атаки горизонтальной эскалации привилегий (BOLA/IDOR).
- **Пакетная OLAP-аналитика:** Регулярная агрегация и денормализация потоков событий из источников выполняются через Apache Airflow DAG с последующим Batch INSERT в ClickHouse MergeTree.

---

## 2. Архитектурная схема (C4 Container)

- **Контур IAM и авторизации:** [`schemas/task1_c4_identity.png`](./schemas/task1_c4_identity.png)
- **Контур сбора данных и аналитики:** [`schemas/task2-report.png`](./schemas/task2-report.png)

Подробные архитектурные спецификации:
- [Архитектура управления доступом и безопасность токенов](./docs/identity_and_security.md)
- [Аналитический пайплайн и витрина данных ClickHouse](./docs/analytics_and_etl.md)

---

## 3. Структура проекта

```
SSO_IAM_Keycloak_DataPlatform/
├── .github/
│   └── workflows/
│       └── ci.yml               # CI-проверка окончаний строк (LF) и валидности compose
├── .gitignore                   # Исключения Git (включая .notes/, volumes)
├── .gitattributes               # Фиксация eol=lf для кроссплатформенности (Win/macOS)
│
├── .notes/                      # [Локально / в .gitignore] Рабочие заметки, шпаргалки и конспекты
│
├── docs/                        # Архитектурные спецификации и RFC
│   ├── identity_and_security.md # Спецификация IAM, OAuth 2.0 PKCE, RBAC и токенов
│   └── analytics_and_etl.md     # Спецификация ClickHouse OLAP, Airflow ETL и API
│
├── schemas/                     # C4-диаграммы архитектуры (drawio / png)
│   ├── task1_c4_identity.png    # C4 Container: контур IAM, Keycloak Broker, BFF
│   └── task2-report.png         # C4 Container: аналитический пайплайн ClickHouse/Airflow
│
├── api/                         # Reporting API (FastAPI)
│   ├── Dockerfile
│   ├── requirements.txt
│   └── main.py                  # Эндпоинт /reports, RS256 JWT-валидация, ClickHouse-клиент
│
├── clickhouse/                  # Аналитическое хранилище OLAP
│   └── init_mart.sql            # DDL схемы витрины MergeTree
│
├── airflow/                     # Оркестрация пакетного сбора данных
│   └── dags/
│       └── telemetry_crm_mart_dag.py # DAG пакетной выгрузки телеметрии и CRM
│
├── frontend/                    # Клиентский интерфейс мониторинга (React SPA)
│   ├── Dockerfile
│   ├── package.json
│   └── src/
│       ├── App.tsx              # Инициализация Keycloak с PKCE S256
│       └── components/
│           └── ReportPage.tsx   # Компонент экрана выгрузки отчетов
│
├── keycloak/                    # Провайдер аутентификации и авторизации
│   └── realm-export.json        # Конфигурация Realm, клиентов, ролей и тестовых УЗ
│
├── docker-compose.yaml          # Оркестрация локального контура сервисов
└── README.md                    # Архитектурная витрина проекта
```


---
## 3. Состав сервисов и порты

| Сервис | Порт хоста | Назначение |
|---|---|---|
| **Frontend UI** | `3000` | Веб-интерфейс пользователя (React SPA) |
| **Keycloak IdP** | `8080` | Провайдер единого входа (SSO) и аутентификации |
| **Reporting API** | `8000` | Защищенный шлюз данных к аналитической витрине |
| **Airflow Web UI**| `8081` | Оркестратор пакетных ETL-пайплайнов |
| **ClickHouse HTTP**| `8123` | Колоночная аналитическая СУБД (OLAP) |
| **Keycloak DB** | `5433` | Хранилище метаданных Keycloak (PostgreSQL) |

---

## 4. Быстрый запуск

### Требования
- Docker Desktop с поддержкой Compose v2.
- 4 ГБ свободной оперативной памяти.

### Команды развертывания
1. Клонирование репозитория:
   ```bash
   git clone [https://github.com/your-account/SSO_IAM_Keycloak_DataPlatform.git](https://github.com/your-account/SSO_IAM_Keycloak_DataPlatform.git)
   cd SSO_IAM_Keycloak_DataPlatform

```

2. Запуск локального контура:
```bash
docker compose up -d --build

```

---

## 5. Тестовые учетные записи

* **Администрирование:**
* Keycloak Console (`http://localhost:8080`): `admin` / `admin`
* Airflow Web UI (`http://localhost:8081`): `admin` / `admin`


* **Пользователи Realm (`reports-realm`):**
* `prothetic1` / `prothetic123` (роль `prothetic_user`) — закреплены телеметрические данные.
* `user1` / `password123` (роль `user`) — пользователь без привязанных устройств (проверка изоляции).


## 6. Сквозной сценарий верификации

1. **Проверка изоляции до запуска ETL:**
Войти на `http://localhost:3000` под `prothetic1`. Нажать «Download Report» — отчёт содержит 0 записей (витрина ClickHouse чистая).
2. **Формирование аналитической витрины:**
В Airflow (`http://localhost:8081`) запустить DAG `telemetry_crm_mart_etl`. Дождаться статуса `success`.
3. **Получение изолированных данных:**
Вернуться на `http://localhost:3000`, повторно нажать «Download Report» — отобразятся строки отчета, относящиеся строго к `prothetic1`.
4. **Проверка безопасности:**
* Неавторизованный запрос: `curl -i http://localhost:8000/reports` вернет `401 Unauthorized`.
* Вход под `user1`: чужие данные `prothetic1` недоступны.
