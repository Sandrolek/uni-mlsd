# Архитектура production-системы

## 1. Общая схема

Диаграмма: [04-system-architecture.png](../diagrams/04-system-architecture.png)

Система состоит из четырех уровней:

1. **Уровень КПП** — RGB+IR+Depth терминал, турникет/дверной контроллер, Edge Inference Node, локальный кэш шаблонов и конфигурации.
2. **Центральный backend** — Access Decision API, Employee/Access Rights Service, Device Management Service, Biometric Template Service, Audit/Event Service, Operator Console Backend.
3. **ML/MLOps** — Face embedding model, Liveness/PAD model, Model Registry, Threshold Configuration Service, Drift/quality monitoring.
4. **Хранилища** — PostgreSQL, Encrypted Template Vault / vector index, Event storage, Object storage, Metrics/logs/traces storage.

---

## 2. Основные компоненты

### RGB+IR+Depth Terminal

Назначение:

- получает изображение лица;
- строит или передает depth map;
- собирает IR-кадр;
- контролирует качество кадра;
- не принимает финальное решение самостоятельно.

Требования: защищенный boot / firmware integrity, mTLS-связь с edge-узлом, синхронизация времени, healthcheck.

### Edge Inference Node

Назначение:

- выполняет локальный ML inference;
- проверяет liveness/PAD;
- строит embedding;
- обращается к локальному кэшу шаблонов;
- отправляет запрос на Access Decision API;
- сохраняет временные события при потере связи.

Почему нужен edge:

- уменьшает latency на КПП;
- снижает зависимость от центральной сети;
- позволяет продолжать работу в degradation mode;
- разгружает центральный ML-кластер.

### Access Decision API

Назначение:

- принимает результат ML;
- проверяет бизнес-правила доступа;
- принимает итоговое решение: allow / deny / manual_review;
- возвращает команду для СКУД;
- пишет событие доступа.

Decision logic:

```text
if liveness_score < PAD_THRESHOLD:
    deny(reason="PAD_FAILED")
elif match_score < IDENTIFICATION_THRESHOLD:
    manual_review(reason="LOW_MATCH_CONFIDENCE")
elif employee.status != "ACTIVE":
    deny(reason="EMPLOYEE_NOT_ACTIVE")
elif not access_rights.allow(employee, checkpoint, time):
    deny(reason="ACCESS_RULE_DENIED")
else:
    allow(reason="OK")
```

### Biometric Template Service

Назначение: создает и отзывает шаблоны, управляет версиями, передает edge-узлам только разрешенный и зашифрованный набор данных, не отдает шаблоны в UI.

### Operator Console

Назначение: показывает спорные события, позволяет дежурному принять ручное решение, отображает статусы устройств, поддерживает расследование инцидентов.

### MLOps Pipeline

Назначение: хранит версии моделей, проводит offline evaluation, публикует модели в staging/prod, поддерживает rollback, отслеживает drift и качество.

---

## 3. Инфраструктура

### Центральный контур

Рекомендуемый вариант:

- Kubernetes-кластер для backend и ML-сервисов;
- PostgreSQL для транзакционных данных;
- Kafka или RabbitMQ для событий;
- ClickHouse или PostgreSQL partitioning для access logs;
- S3-compatible object storage для model artifacts и короткоживущих captures;
- Prometheus + Grafana для метрик;
- Loki/ELK для логов;
- Vault/KMS для секретов и ключей.

### Edge-контур

На каждой проходной:

- промышленный mini-server или appliance;
- локальная embedded DB для кэша;
- inference runtime;
- device adapter для терминалов;
- буфер событий на случай offline-mode.

---

## 4. Масштабируемость

Исходные параметры: 5000 сотрудников, 10 проходных, пиковый поток утром/вечером, целевой latency P95 ≤ 1,5 секунды.

Горизонтальное масштабирование:

- Access Decision API — stateless, масштабируется репликами;
- Event Service — через брокер сообщений;
- Operator Console Backend — stateless;
- ML inference — edge-first, центральный fallback при необходимости;
- Template sync — инкрементальные обновления, а не полная рассылка.

---

## 5. SLA/SLO

| Показатель | Цель |
|---|---:|
| Central API availability | 99,5% |
| Edge node availability на КПП | 99,0% |
| P95 decision latency | ≤ 1,5 секунды |
| P99 decision latency | ≤ 3 секунды |
| Время восстановления edge-узла | ≤ 30 минут |
| Потеря access events | 0 при штатном восстановлении связи |
| Актуализация уволенного сотрудника на edge | ≤ 5 минут после изменения статуса |

---

## 6. Режимы деградации

| Сценарий | Поведение |
|---|---|
| Нет связи edge -> central до 5 минут | Использовать локальный кэш, буферизовать события |
| Нет связи дольше допустимого окна | Перевести КПП в manual_review или ручной режим |
| Недоступен template vault | Использовать последнюю валидную edge-реплику |
| Отказ PAD-модуля | Не открывать автоматически, только ручная проверка |
| Отказ терминала | Ручной fallback через дежурного |
| Высокая очередь | Оповещение оператора, включение дополнительного КПП/ручного режима |

---

## 7. Безопасность архитектуры

Минимальный набор мер:

- mTLS между терминалами, edge и central;
- device certificates;
- signed model artifacts;
- RBAC и least privilege;
- шифрование biometric templates;
- аудит доступа к шаблонам;
- раздельные роли operator/admin/ml-engineer;
- централизованное логирование security events;
- регулярный тест spoofing/PAD;
- запрет хранения raw captures по умолчанию.

