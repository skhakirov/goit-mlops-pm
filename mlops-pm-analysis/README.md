# ДЗ №4. Аналіз MLflow runs з позиції AI PM

Шаміль Хакіров · MLOps CI/CD, Тема 9 · гілка `lesson-9-pm`. Повний аналіз — у [`pm_report.md`](pm_report.md).

## Задача

Експеримент `iris-classification`: класифікація ірисів за чотирма вимірами квітки, три види, 150 квіток, тест — 45 (`test_size=0.3`, `split_random_state=7`). Датасет навчальний, тому бізнес-контекст у звіті — умовний сценарій маркування партій у розсаднику, позначений як припущення.

## Які дані використано

Скриншоти й опис задачі від менторки Діани Кобякової; доступу до UI не було. Grafana в наборі немає, тому `grafana_metrics.png` — це графік `mlflow_accuracy` у Prometheus.

| Файл у `screenshots/` | Що показує |
|---|---|
| `mlflow_runs.png` | Таблиця 5 runs: параметри, метрики, source |
| `mlflow_compare.png` | Порівняльні графіки accuracy і loss |
| `mlflow_run_rf_depth3.png` | Сторінка кандидата: run ID, параметри, теги |
| `mlflow_artifacts.png` | Артефакти rf-depth3 (модель збережено) |
| `mlflow_artifacts_rf_small.png` | Артефакти rf-small (порожньо) |
| `grafana_metrics.png` | Prometheus: графік `mlflow_accuracy` |
| `prometheus_loss_graph.png` | Prometheus: графік `mlflow_loss` |
| `prometheus_accuracy_table.png` | Prometheus: таблиця `mlflow_accuracy` |
| `prometheus_loss_table.png` | Prometheus: таблиця `mlflow_loss` |

## Кандидат

Run `rf-depth3` (`2ddc4f9d563c43e5a7c6c8523a580db3`): accuracy 0,9556, loss 0,1415, єдиний зі збереженою моделлю.

## PM-рішення

**Need more experiments.** Перевага над `logreg-baseline` за accuracy — одна квітка з 45, а в кандидата порожній source. Якщо критерій із розд. 5 звіту виконано, модель із повторного run іде в staging без нового циклу погоджень.

## Виявлені ризики

Вибір лише за accuracy; випадковість малого тесту; неможливість відтворити rf-depth3; відсутні rollback-план, drift monitoring і owner рішення; неперевірена пара versicolor і virginica; навчальні дані, не схожі на реальні партії. Усі 10 ризиків — у розд. 6 звіту.

## Що зробити перед production

1. Призначити owner-а рішення, затвердити пороги.
2. Повторити run через `train.py`, провести крос-валідацію, залогувати recall по видах, зареєструвати модель у Model Registry.
3. Перевірити модель у staging на реальних вимірах, описати rollback, увімкнути моніторинг drift і якості з алертами.
