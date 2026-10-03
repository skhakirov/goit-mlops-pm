# Optimization Report Review: fat vs slim

Оцінка результату оптимізації за вимірюваними метриками, а не за фактом «зроблено». Даних від команди ще немає, тому значення — `TBD`; для кожної метрики вказано, хто надає значення і якою командою його отримати.

Теги в прикладах команд з умови (`ml-infer:fat`, `ml-infer:slim`) адаптовано до імен образів сценарію: `ml-infer-fat:1.0` і `ml-infer-slim:1.0`.

## Порівняльна таблиця

| Метрика | Fat | Slim | Коментар AI PM: хто надає і як отримати |
|---|---|---|---|
| Розмір image | TBD | TBD | AI/ML Engineer: `docker images \| grep ml-infer`. Це локальний розпакований розмір; DevOps бере з registry розмір, який завантажується на staging. |
| Кількість шарів | TBD | TBD | AI/ML Engineer: `docker image inspect -f '{{len .RootFS.Layers}}' ml-infer-fat:1.0` (і для slim). `docker history ml-infer-slim:1.0` — щоб побачити, які інструкції дають найбільші шари. |
| Час збірки | TBD | TBD | AI/ML Engineer локально, DevOps — у CI: `time docker build --no-cache -f Dockerfile.slim -t ml-infer-slim:1.0 .` (і для fat). Обидві збірки — на тій самій машині, інакше порівняння некоректне. |
| Час запуску | TBD | TBD | QA: `time docker run --rm -v "$(pwd)/tests:/data:ro" ml-infer-slim:1.0 --image /data/test_input.jpg` (і для fat), три запуски, у звіт — середнє. Вимірює старт контейнера, завантаження моделі й один inference. |
| Inference result | TBD | TBD | QA: `docker run --rm -v "$(pwd)/tests:/data:ro" ml-infer-fat:1.0 --image /data/test_input.jpg > fat.json` і та сама команда для slim у `slim.json`, потім порівняння `top3` у двох файлах. Має збігатися: ті самі `class_id` у тому самому порядку, різниця confidence ≤ 0.001 (AC-08). Обидва файли — у `report.md`. |
| Security findings | TBD | TBD | DevOps: `trivy image --severity HIGH,CRITICAL ml-infer-fat:1.0` (і для slim). У звіті — кількість HIGH і CRITICAL та рішення щодо кожної CRITICAL. |
| Зайві файли | TBD | TBD | AI/ML Engineer, перевіряє AI PM: `docker run --rm --entrypoint ls ml-infer-slim:1.0 -AR /app` (і для fat). Очікування для slim — лише код, модель і конфігурації (AC-11). |

Окрема умова чесного порівняння: обидва образи встановлюють torch із CPU-індексу (AC-14). Якщо fat отримає стандартну збірку torch із CUDA-пакетами, різницю в розмірі дадуть саме вони, а не slim-база й multi-stage, і висновок про ефект оптимізації буде хибним.

## Висновок

Вердикт на поточний стан, коли команда ще не здала `report.md`:

| Питання | Відповідь зараз | Що блокує | Що змінить відповідь |
|---|---|---|---|
| Чи є оптимізація вимірюваною? | Так, за методикою: для всіх 7 метрик визначено команду й відповідального. Значень поки немає. | `report.md` не здано (AC-10) | Заповнена таблиця від AI/ML Engineer, DevOps і QA |
| Чи зменшився розмір image? | Не можна стверджувати: вимірювання немає | AC-10; порівнянність образів (AC-14) | Розмір обох образів після підтвердження, що torch у них — CPU-збірка |
| Чи зберігся inference? | Не підтверджено | AC-06, AC-08 | Виводи fat і slim на однаковому test input від QA |
| Чи є ризик, що оптимізація зламала модель? | Так, ризик відкритий (R-02 у `risk_register.md`) | AC-08 | Збіг top-3 і confidence у межах 0.001; розмір цей ризик не закриває |
| Чи достатньо даних для переходу до staging? | Ні | AC-01…AC-15, security scan, стенд Docker Compose (`compose_readiness_checklist.md`) | Виконані AC, `report.md`, розгляд кожної CRITICAL із security scan і готовий локальний стенд |

Рішення AI PM на цей момент: перехід до staging не погоджено. Повторний review — одразу після отримання `report.md`.
