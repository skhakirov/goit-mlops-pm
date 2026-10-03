# Acceptance Criteria: контейнеризація моделі класифікації зображень

Кожен критерій відповідає на питання «Як ми зрозуміємо, що задача виконана правильно?» Команди використовують імена образів `ml-infer-fat:1.0` і `ml-infer-slim:1.0`.

## Критерії приймання

| ID | Критерій | Як перевіряємо: очікуваний результат | Хто перевіряє |
|---|---|---|---|
| AC-01 | Fat image використовує `python:3.13` | У `Dockerfile.fat` базовий образ — `FROM python:3.13` | AI PM (review), QA |
| AC-02 | Slim image використовує `python:3.13-slim` | У `Dockerfile.slim` фінальний етап починається з `FROM python:3.13-slim` | AI PM (review), QA |
| AC-03 | Slim image зібраний через multi-stage build | У `Dockerfile.slim` щонайменше два `FROM`; фінальний етап отримує з build-етапу лише встановлені пакети (`COPY --from=...`) | AI/ML Engineer, AI PM (review) |
| AC-04 | Обидва образи збираються без помилок | `docker build --no-cache -f Dockerfile.fat -t ml-infer-fat:1.0 .` і така сама команда для `Dockerfile.slim` → `ml-infer-slim:1.0` завершуються з кодом 0 | QA, DevOps (у CI) |
| AC-05 | Обидва образи запускаються через `docker run` | Команда run з README (`docker run --rm -v "$(pwd)/tests:/data:ro" <образ> --image /data/test_input.jpg`) для кожного образу завершується з кодом 0 без ручного встановлення залежностей | QA |
| AC-06 | Для fat і slim використовується однаковий test input | Обидва запуски отримують той самий файл `tests/test_input.jpg`, змонтований із хоста лише для читання; sha256 файлу записаний у `report.md` | QA |
| AC-07 | Inference повертає top-3 prediction | Відповідь — JSON з полями `prediction`, `class_id`, `confidence`, `model_version` і масивом `top3` із рівно трьох елементів `{class_id, label, confidence}`, упорядкованих за спаданням confidence; confidence — softmax-імовірність у діапазоні [0, 1] | QA |
| AC-08 | Результат inference у fat і slim збігається | Однакові `class_id` у тому самому порядку в `top3`, різниця confidence на кожній позиції ≤ 0.001; обидва виводи додані в `report.md` | QA, AI/ML Engineer |
| AC-09 | README містить команди build/run | Дві команди build і дві команди run копіюються з README без змін і виконуються на чистій машині | QA |
| AC-10 | `report.md` містить порівняння fat і slim | Таблиця fat vs slim за всіма метриками з `optimization_report_review.md` разом із командами, якими значення отримано | AI/ML Engineer, AI PM (review) |
| AC-11 | У slim image немає зайвих файлів | `docker run --rm --entrypoint ls ml-infer-slim:1.0 -AR /app` показує лише inference-код, модель і потрібні конфігурації; немає `tests/`, ноутбуків, `.git`, `Dockerfile*`, training-коду | AI PM (review), QA |
| AC-12 | У репозиторії є `.dockerignore` | Файл лежить у корені репозиторію й виключає щонайменше `.git`, `tests/`, ноутбуки, кеші, `.env`, дані для тренування | AI PM (review) |
| AC-13 | Залежності зафіксовані | Кожен пакет у `requirements.txt` має точну версію через `==`; обидва образи встановлюють залежності з цього файлу | AI PM (review) |
| AC-14 | Torch в обох образах — CPU-збірка | `docker run --rm --entrypoint python <образ> -c "import torch; print(torch.version.cuda)"` друкує `None` для fat і slim; у `pip list` немає пакетів `nvidia-*` | AI/ML Engineer, QA |
| AC-15 | Версія й джерело моделі задокументовані | `model_version` присутній у відповіді (AC-07); у README описані джерело моделі (`torchvision`, `ResNet18_Weights.IMAGENET1K_V1`) і sha256 `model/model.pt`; sha256 однаковий в обох образах | AI/ML Engineer |

## Definition of Done

Задача вважається виконаною, якщо будь-який член команди може клонувати репозиторій, виконати команди з README, зібрати fat і slim image, запустити inference на однаковому input і побачити однаковий результат prediction.

Для закриття задачі додатково потрібно:

- AC-01…AC-15 позначені як виконані, а `report.md` доданий до задачі;
- QA відтворив збірку й запуск на чистій машині за README, без допомоги автора;
- AI PM переглянув `docker_image_review.md` і `optimization_report_review.md` та погодив push slim-образу в registry.
