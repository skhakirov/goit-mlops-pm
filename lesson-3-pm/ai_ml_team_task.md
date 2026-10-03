# Title: Контейнеризація моделі класифікації зображень — образи `ml-infer-fat:1.0` і `ml-infer-slim:1.0`

## Background

Для пілоту автоматичної класифікації фото товарів потрібен відтворюваний delivery-артефакт моделі. Модель — pretrained ResNet-18 з `torchvision.models` (ваги `IMAGENET1K_V1`), файл `model/model.pt`; inference повертає top-3 класи з confidence.

Модель має однаково запускатися локально, у CI/CD і на staging. Спершу збираємо простий fat-образ як еталон, потім оптимізований slim-образ і доводимо, що оптимізація не змінила inference. Повний контекст — у `project_brief.md`.

## Scope

- `Dockerfile.fat` на базі `python:3.13` → образ `ml-infer-fat:1.0`.
- `Dockerfile.slim` на базі `python:3.13-slim` з multi-stage build → образ `ml-infer-slim:1.0`.
- `requirements.txt` із зафіксованими версіями (`==`), спільний для обох образів.
- `.dockerignore`, який не пускає в build context `.git`, `tests/`, ноутбуки, кеші, `.env` і дані.
- `app/inference.py`: завантажує `model/model.pt`, приймає шлях до зображення параметром `--image` і повертає JSON із top-3 prediction; запускається як `ENTRYPOINT` образу. Це пропонований CLI-контракт: якщо команда обере інший, вона оновлює README і команди перевірки.
- Перевірка однаковості inference: обидва образи запускаються на тому самому test input, результати порівнюються.
- CI-pipeline (DevOps): збірка обох образів і порівняння inference fat і slim на test input після кожної зміни Dockerfile, `requirements.txt` чи коду.
- Підтвердження складу майбутнього Compose-стенду: роль кожного сервісу (зокрема Redis — кеш чи черга) і HTTP-режим `model-api`; відповідь — коментарем до задачі.
- `README.md` з командами build і run для обох образів.
- `report.md` з порівнянням fat і slim.

## Out of Scope

- Model retraining.
- Покращення accuracy.
- Production Kubernetes.
- Full monitoring: алерти, дашборди якості моделі, відстеження drift у production.

**Наступний етап (окрема задача):** локальний стенд Docker Compose з `model-api`, Redis, PostgreSQL і локальним моніторингом. Checklist готовності підготовлено заздалегідь (`compose_readiness_checklist.md`); рішення про staging без готового стенду не ухвалюється.

## Expected Deliverables

| Артефакт | Що має містити |
|---|---|
| `Dockerfile.fat` | Одноетапна збірка на `python:3.13` |
| `Dockerfile.slim` | Multi-stage build; фінальний етап на `python:3.13-slim` |
| `.dockerignore` | Виключення всього, що не потрібне для inference |
| `requirements.txt` | Лише runtime-залежності, кожна з `==` |
| `model/model.pt` | Ваги ResNet-18; джерело й sha256 описані в README |
| `app/inference.py` | Завантаження моделі, preprocessing, top-3 у JSON |
| `README.md` | Команди build і run для обох образів, опис input/output, джерело й версія моделі |
| `report.md` | Таблиця fat vs slim за метриками з `optimization_report_review.md`; виводи обох запусків; sha256 тестового зображення |
| `tests/test_input.jpg` | Тестове зображення, спільне для fat і slim; готує AI/ML Engineer |
| CI-job | Збірка обох образів і порівняння inference fat і slim; результат видно в pipeline |
| Образи `ml-infer-fat:1.0`, `ml-infer-slim:1.0` | Зібрані локально; push у registry — після погодження AI PM |

## Technical Constraints

- Python 3.13 в обох образах: `python:3.13` для fat і `python:3.13-slim` для slim.
- Один `requirements.txt` на обидва образи; версії зафіксовані через `==`; сумісність torch, torchvision і pillow з Python 3.13 підтверджує AI/ML Engineer.
- Torch і torchvision — CPU-збірки з CPU-індексу PyTorch в обох образах; GPU runtime не потрібен. Перевірка: `torch.version.cuda` повертає `None`, у `pip list` немає пакетів `nvidia-*`.
- Той самий `model/model.pt` (однаковий sha256) і той самий preprocessing (трансформації ваг `IMAGENET1K_V1`) в обох образах.
- Тестове зображення не вшивається в образ: воно монтується з хоста лише для читання.
- Жодних секретів в образі, `Dockerfile` чи репозиторії.
- Формат відповіді — JSON-контракт з `acceptance_criteria.md` (AC-07).

## Acceptance Criteria Summary

Повний перелік із методами перевірки — у `acceptance_criteria.md`. Коротко:

- обидва образи збираються й запускаються командами з README (AC-04, AC-05, AC-09);
- бази образів і multi-stage для slim відповідають вимогам (AC-01, AC-02, AC-03);
- на однаковому test input обидва образи повертають top-3 у погодженому форматі, і результати збігаються (AC-06, AC-07, AC-08);
- у slim немає зайвих файлів, є `.dockerignore`, залежності зафіксовані, torch — CPU-збірка (AC-11, AC-12, AC-13, AC-14);
- версія й джерело моделі задокументовані (AC-15), `report.md` містить порівняння fat і slim (AC-10).
