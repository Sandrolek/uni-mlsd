# Структура данных и распределенное хранение

## 1. Выбор вида базы данных

Для production-системы предлагается комбинированное хранение:

| Тип данных | Хранилище | Причина |
|---|---|---|
| Сотрудники, устройства, права доступа | Реляционная БД, например PostgreSQL | Целостность, транзакции, связи |
| Биометрические шаблоны / embeddings | Зашифрованное хранилище шаблонов + vector index | Быстрый similarity search и отдельный security boundary |
| События доступа | Event storage / append-only таблицы / Kafka + ClickHouse/PostgreSQL | Большой поток событий, аудит, аналитика |
| Raw captures для спорных случаев | Object Storage с коротким TTL | Не хранить постоянно чувствительные изображения |
| Модели ML и конфигурации | Model Registry + object storage | Версионирование, откат, reproducibility |
| Edge-кэш на КПП | Локальная БД/embedded storage | Работа при кратковременной потере связи |

ER-диаграмма: [03-er-data-model.puml](../diagrams/03-er-data-model.puml)

---

## 2. Почему данные распределены

Данные нельзя хранить только в одном центральном месте по нескольким причинам:

1. **10 проходных** создают географически/сетево распределенные точки принятия решения.
2. КПП должно продолжать работу при кратковременном сбое связи с центральным кластером.
3. Биометрические шаблоны требуют отдельного защищенного контура хранения.
4. События доступа генерируются на edge-уровне, но должны попадать в центральный аудит.
5. Raw captures нельзя хранить долго и массово; они должны жить в отдельном объектном хранилище с TTL.
6. ML-модель и конфигурация порогов должны версионироваться и раскатываться на edge-узлы контролируемо.

---

## 3. Основные сущности

### employee

Сотрудник предприятия.

Ключевые поля: `employee_id`, `personnel_number`, `full_name`, `department`, `employment_status`, `access_group_id`, `created_at`, `updated_at`.

### biometric_template

Зашифрованный биометрический шаблон лица.

Ключевые поля: `template_id`, `employee_id`, `template_version`, `embedding_ref`, `quality_score`, `enrollment_device_id`, `model_version_id`, `status`, `created_at`, `revoked_at`.

Важно: в прикладной БД не хранится сам embedding в открытом виде. Хранится ссылка на защищенное хранилище или зашифрованный blob.

### checkpoint

Проходная предприятия: `checkpoint_id`, `name`, `location`, `security_level`, `status`.

### device

Биометрический терминал или edge-узел: `device_id`, `checkpoint_id`, `device_type`, `vendor`, `firmware_version`, `last_seen_at`, `status`.

### access_event

Событие попытки прохода: `event_id`, `checkpoint_id`, `device_id`, `candidate_employee_id`, `decision`, `reason_code`, `match_score`, `liveness_score`, `depth_quality_score`, `model_version_id`, `created_at`.

### operator_override

Ручное решение дежурного: `override_id`, `event_id`, `operator_id`, `decision`, `comment`, `created_at`.

### model_version

Версия ML-модели: `model_version_id`, `face_model_name`, `pad_model_name`, `artifact_uri`, `threshold_config`, `status`, `created_at`, `deployed_at`.

---

## 4. Политика хранения данных

| Данные | Срок хранения | Обоснование |
|---|---:|---|
| Биометрический шаблон активного сотрудника | Пока сотрудник работает и есть правовое основание | Нужен для доступа |
| Шаблон уволенного сотрудника | Удалить/отозвать в регламентный срок | Минимизация данных |
| Access events | 6–12 месяцев или по политике безопасности | Расследование инцидентов |
| Raw captures спорных событий | 7–30 дней | Разбор отказов и апелляций |
| Debug captures штатных проходов | Не хранить по умолчанию | Снижение privacy-риска |
| Model artifacts | До вывода из эксплуатации + период аудита | Reproducibility и расследования |

---

## 5. Индексация и производительность

Для 5000 сотрудников 1:N search технически не является тяжелым, но архитектура должна учитывать рост.

Рекомендуемые индексы:

- `employee.personnel_number`
- `employee.employment_status`
- `biometric_template.employee_id`
- `biometric_template.status`
- `access_event.created_at`
- `access_event.checkpoint_id`
- `access_event.candidate_employee_id`
- vector index по embedding в защищенном search-компоненте

---

## 6. Защита данных

Требования:

- шифрование шаблонов в покое;
- шифрование трафика между edge и центральными сервисами;
- отдельный KMS/HSM для ключей;
- RBAC для операторов, администраторов и ML-инженеров;
- запрет прямого доступа к biometric template storage из UI;
- аудит чтения и изменения шаблонов;
- удаление или отзыв шаблона при увольнении;
- хранение model version в каждом access event для расследуемости.

