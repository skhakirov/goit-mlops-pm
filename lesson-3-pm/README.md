# ДЗ №1. PM-пакет: контейнеризація ML-моделі

| | |
|---|---|
| Студент | Шаміль Хакіров |
| Дисципліна | MLOps CI/CD |
| Тема | Тема 3. Контейнеризація ML-моделей |
| Дата | 03.10.2026 |

## Мета

Підготувати управлінський PM-пакет для контейнеризації ML-моделі: поставити задачу AI/ML-команді, визначити критерії приймання, перевірити delivery-артефакти (Docker-образи, залежності, локальний стенд), оцінити ризики й пояснити стан проєкту нетехнічному стейкхолдеру. Код (Dockerfile, inference, Bash-скрипти) у пакет не входить: його готує команда, а AI PM контролює результат.

## Контекст

AI/ML-команда контейнеризує pretrained-модель класифікації зображень ResNet-18 з `torchvision.models` для пілоту автоматичної класифікації фото товарів. Плануються два образи: `ml-infer-fat:1.0` на `python:3.13` для першої перевірки і `ml-infer-slim:1.0` на `python:3.13-slim` з multi-stage build. Головна умова: після оптимізації inference не змінюється. Детальніше — у `project_brief.md`.

## PM-артефакти

| № | Файл | Що всередині | Як використовувати |
|---|---|---|---|
| 1 | `project_brief.md` | Бізнесовий і технічний контекст, модель, fat і slim, де перевіряється результат | Читати першим; на kick-off узгодити з командою розділ «Технічні рамки» |
| 2 | `ai_ml_team_task.md` | Задача для Jira / Linear / GitLab Issues: scope, out of scope, deliverables, обмеження | Скопіювати в трекер як задачу для команди |
| 3 | `acceptance_criteria.md` | 15 перевірюваних критеріїв AC-01…AC-15 і Definition of Done | Приймати роботу команди; посилатися на ID критеріїв у коментарях |
| 4 | `docker_image_review.md` | Checklist на 20 пунктів: Dockerfile, залежності, запуск, відтворюваність | Пройти, коли команда здасть артефакти; заповнити статуси |
| 5 | `optimization_report_review.md` | Таблиця fat vs slim з командами та вердикт щодо оптимізації | Заповнити за даними з `report.md` команди й оновити висновок |
| 6 | `compose_readiness_checklist.md` | Склад локального стенду і checklist на 11 пунктів | Використати на наступному етапі, коли з'явиться `compose.yaml` |
| 7 | `risk_register.md` | 10 ризиків з імовірністю, впливом, mitigation і owner | Переглядати на кожному статусі; оновити ймовірності після `report.md` |
| 8 | `stakeholder_summary.md` | Короткий звіт для нетехнічного стейкхолдера | Надіслати замовнику пілоту або керівнику напряму |
| 9 | `README.md` | Навігація, запитання до команди, команди перевірки | Точка входу в пакет |

## Questions to AI/ML Team

1. Який фінальний model artifact використовується? Підтвердіть: ResNet-18, ваги `ResNet18_Weights.IMAGENET1K_V1`, файл `model/model.pt`.
2. Яка його версія? Підтвердіть `model_version = resnet18-imagenet1k-v1` або запропонуйте свою схему версій.
3. Який input/output format очікується? Підтвердіть контракт з AC-07: на вхід — шлях до зображення, на виході — JSON з `prediction`, `class_id`, `confidence`, `model_version` і `top3`.
4. Чи підтверджена сумісність із `python:3.13`?
5. Чи підтверджена сумісність із `python:3.13-slim`?
6. Чи сумісні torch, torchvision і pillow з обраною версією Python? Які саме версії зафіксовано в `requirements.txt`?
7. У якому форматі збережено `model.pt`: state_dict, TorchScript чи повна модель через `torch.save(model)`? Чи завантажується він обраною версією torch без змін у коді (у torch 2.6+ `torch.load` за замовчуванням працює з `weights_only=True`)?
8. Чи встановлюється torch в обох образах із CPU-індексу, без пакетів `nvidia-*`? Чи справді не потрібен GPU runtime?
9. Чи є `.dockerignore` і що саме він виключає?
10. Чи збігається inference для fat і slim на однаковому test input?
11. Чи є `report.md` із порівнянням fat і slim за всіма метриками?
12. Чи містить README команди build і run для обох образів?
13. Чи потрібен Docker Compose для локального стенду? Склад стенду задано: `model-api`, model runtime, Redis, PostgreSQL, monitoring. Підтвердіть роль Redis (кеш чи черга) і HTTP-режим `model-api` з `/health` і `/predict`.
14. Чи немає секретів у Compose і в репозиторії загалом?
15. Чи використовуєте скрипт підготовки середовища? Якщо так — чи він ідемпотентний і чи описаний у README?

## Команди для перевірки

Команди відомі на рівні очікуваного інтерфейсу; остаточні версії мають бути в README команди.

```bash
# збірка обох образів (час збірки вимірюємо з --no-cache на одній машині)
docker build --no-cache -f Dockerfile.fat -t ml-infer-fat:1.0 .
docker build --no-cache -f Dockerfile.slim -t ml-infer-slim:1.0 .

# inference на однаковому тестовому зображенні, змонтованому лише для читання
docker run --rm -v "$(pwd)/tests:/data:ro" ml-infer-fat:1.0 --image /data/test_input.jpg
docker run --rm -v "$(pwd)/tests:/data:ro" ml-infer-slim:1.0 --image /data/test_input.jpg

# розмір і кількість шарів
docker images | grep ml-infer
docker image inspect -f '{{len .RootFS.Layers}}' ml-infer-slim:1.0
docker history ml-infer-slim:1.0

# вміст образу, CPU-збірка torch, однаковість моделі
docker run --rm --entrypoint ls ml-infer-slim:1.0 -AR /app
docker run --rm --entrypoint python ml-infer-slim:1.0 -c "import torch; print(torch.version.cuda)"
docker run --rm --entrypoint sha256sum ml-infer-slim:1.0 /app/model/model.pt

# security scan
trivy image --severity HIGH,CRITICAL ml-infer-slim:1.0

# локальний стенд (наступний етап)
docker compose up --build
# в іншому терміналі, коли сервіси healthy
curl http://localhost:8000/health
```

Решту команд для fat-образу виконують так само, з тегом `ml-infer-fat:1.0`.
