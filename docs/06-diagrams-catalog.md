# Каталог диаграмм

Все диаграммы находятся в директории [`diagrams`](../diagrams).

## 1. Список диаграмм

| Файл | Тип | Назначение |
|---|---|---|
| [01-bpmn-as-is.png](../diagrams/01-bpmn-as-is.png) | BPMN-style activity | Процесс ручной проверки до внедрения |
| [02-bpmn-to-be.png](../diagrams/02-bpmn-to-be.png) | BPMN-style activity | Процесс автоматизированной проверки после внедрения |
| [03-er-data-model.png](../diagrams/03-er-data-model.png) | ER diagram | Структура данных |
| [04-system-architecture.png](../diagrams/04-system-architecture.png) | Architecture diagram | Распределенная архитектура |
| [05-uml-components.png](../diagrams/05-uml-components.png) | UML component | Компоненты системы и зависимости |
| [06-uml-sequence-access.png](../diagrams/06-uml-sequence-access.png) | UML sequence | Сценарий попытки прохода |
| [07-uml-activity-enrollment.png](../diagrams/07-uml-activity-enrollment.png) | UML activity | Регистрация биометрии сотрудника |

## 2. Генерация PNG/SVG

Пример генерации всех диаграмм:

```bash
plantuml -tpng diagrams/*.puml
plantuml -tsvg diagrams/*.puml
```

Если используется Docker:

```bash
docker run --rm -v "$PWD:/work" -w /work plantuml/plantuml -tpng diagrams/*.puml
docker run --rm -v "$PWD:/work" -w /work plantuml/plantuml -tsvg diagrams/*.puml
```

## 3. Рекомендация для репозитория

Рекомендуемая структура после генерации:

```text
.
├── README.md
├── docs/
├── diagrams/
│   ├── *.puml
│   ├── *.png
│   └── *.svg
```

В GitHub можно оставить `.puml` как исходники и дополнительно закоммитить `.png` или `.svg`, чтобы преподавателю было удобно смотреть диаграммы без локальной генерации.

