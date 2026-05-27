# ПО для распознавания эмоций и извлечения аспектных групп из отзывов клиентов

MVP для ВКР: локальный backend, браузерное расширение и веб-отчет.

## Что умеет

- запуск анализа с текущей страницы через расширение Chrome/Edge;
- определение объекта анализа по `title`, `h1`, meta-description и тексту страницы;
- глубокий сбор доступных отзывов на странице: расширение пытается открыть раздел отзывов, прокручивать список и нажимать "Показать ещё";
- поиск релевантных отзывов в демонстрационной базе;
- честный статус источника данных: видимые отзывы, демо-база или отзывы не извлечены;
- распознавание эмоций в отзывах;
- извлечение аспектных групп;
- расчет сводной аналитики;
- открытие полного отчета в браузере.

## Структура

```text
backend/
  analyzer.py          NLP baseline: эмоции, тональность, аспекты
  server.py            локальный HTTP API и веб-отчет
  demo_reviews.json    демонстрационная база отзывов
extension/
  manifest.json        расширение Chrome/Edge Manifest V3
  popup.html           интерфейс расширения
  popup.css
  popup.js
  content.js           извлечение данных с открытой страницы
web/
  report.html          страница полного отчета
  report.css
  report.js
models/
  baseline_review_model.json  обученная baseline-модель
  rureviews_sentiment_model.json  модель тональности на готовом датасете RuReviews
scripts/
  train_baseline_model.py     сбор датасета из отчетов и обучение модели
  download_rureviews.py       скачивание открытого датасета RuReviews
  train_rureviews_sentiment_model.py  обучение тональности на готовом датасете
  train_rubert_sentiment_gpu.py  обучение RuBERT/RuBERT-tiny на видеокарте
  download_sentirueval.py     скачивание SentiRuEval для аспектного анализа
  train_sentirueval_aspect_model.py  обучение аспектной модели
```

## Запуск backend

```powershell
cd D:\VKR\review-emotion-aspect-analyzer
.\scripts\start_backend.ps1
```

Backend запустится на:

```text
http://localhost:8765
```

Если PowerShell запретит запуск скрипта, можно запустить так:

```powershell
powershell -ExecutionPolicy Bypass -File .\scripts\start_backend.ps1
```

Проверка:

```text
http://localhost:8765/health
```

## Установка расширения

1. Открой `chrome://extensions` или `edge://extensions`.
2. Включи режим разработчика.
3. Нажми "Загрузить распакованное расширение".
4. Выбери папку:

```text
D:\VKR\review-emotion-aspect-analyzer\extension
```

## Демонстрация

1. Запусти backend.
2. Открой любую страницу товара, услуги или организации.
3. Нажми иконку расширения.
4. Нажми "Анализировать страницу".
5. Открой полный отчет.

Для стабильной защиты можно использовать запросы:

- `Кофейня у Ашота Екатеринбург`
- `Подгузники детские Агуша`

Важно: MVP не обходит защиту сайтов и не получает закрытые данные напрямую с серверов площадок. Расширение собирает отзывы, которые доступны пользователю на открытой странице и могут быть загружены прокруткой или кнопками "Показать ещё".

## Отчет и фильтры

Полный отчет открывается по ссылке вида:

```text
http://localhost:8765/report/<id>
```

В отчете доступны:

- поиск по тексту отзывов;
- фильтрация по тональности, эмоции, аспектной группе, источнику и оценке;
- сортировка отзывов;
- пересчет показателей по текущей выборке;
- экспорт отфильтрованных отзывов в CSV;
- копирование ссылки на отчет.

## Обучение baseline-модели

Скрипт обучения собирает размеченные отзывы из сохраненных отчетов `data/analyses/*.json`, добавляет демонстрационные отзывы и обучает простую модель Multinomial Naive Bayes для трех задач:

- тональность;
- эмоция;
- основная аспектная группа.

Запуск:

```powershell
cd D:\VKR\review-emotion-aspect-analyzer
python .\scripts\train_baseline_model.py
```

Если команда `python` не найдена, в текущем окружении можно использовать bundled Python:

```powershell
C:\Users\aryti\.cache\codex-runtimes\codex-primary-runtime\dependencies\python\python.exe .\scripts\train_baseline_model.py
```

Результат сохраняется в:

```text
models/baseline_review_model.json
```

Текущий прогон на накопленных отчетах:

```text
Размер датасета: 1143
Тональность accuracy: 0.686
Эмоции accuracy: 0.651
Аспекты accuracy: 0.721
```

Для ВКР это можно описывать как обучаемый baseline. Следующий качественный шаг — собрать ручную разметку и заменить baseline на RuBERT/transformers.

## Обучение на готовом датасете RuReviews

RuReviews — открытый корпус русскоязычных товарных отзывов с тремя классами тональности. Источник: https://github.com/sismetanin/rureviews, лицензия Apache-2.0.

Скачать датасет:

```powershell
C:\Users\aryti\.cache\codex-runtimes\codex-primary-runtime\dependencies\python\python.exe .\scripts\download_rureviews.py
```

Датасет сохраняется в:

```text
data/external/rureviews/women-clothing-accessories.3-class.balanced.csv
```

Обучить модель тональности:

```powershell
C:\Users\aryti\.cache\codex-runtimes\codex-primary-runtime\dependencies\python\python.exe .\scripts\train_rureviews_sentiment_model.py
```

Результат сохраняется в:

```text
models/rureviews_sentiment_model.json
```

Текущий прогон:

```text
Датасет: 89063 отзывов
Train: 71249
Test: 17814
Accuracy: 0.715
негатив: precision=0.731 recall=0.629 f1=0.676
нейтрально: precision=0.591 recall=0.671 f1=0.628
позитив: precision=0.841 recall=0.845 f1=0.843
```

Если файл `models/rureviews_sentiment_model.json` существует, backend использует эту обученную модель для определения тональности. Словарный анализ остается fallback-вариантом.

## Нейросетевое обучение на RTX 3070

Для компьютера с RTX 3070 лучше обучать нейросетевую модель локально, а не в Colab: данные остаются на машине, эксперимент можно повторить, а в ВКР можно показать реальные параметры обучения и метрики на GPU.

Рекомендуемая схема:

1. Установить свежий драйвер NVIDIA и проверить видеокарту:

```powershell
nvidia-smi
```

2. Создать отдельное окружение Python:

```powershell
cd D:\VKR\review-emotion-aspect-analyzer
python -m venv .venv-gpu
.\.venv-gpu\Scripts\Activate.ps1
python -m pip install --upgrade pip
```

3. Установить PyTorch с CUDA по актуальной команде с официального сайта PyTorch: https://pytorch.org/get-started/locally/. Для RTX 3070 обычно выбирается Windows, Pip, Python и CUDA-версия, которую предлагает установщик PyTorch.

4. Установить зависимости проекта для обучения:

```powershell
pip install -r requirements-gpu.txt
```

5. Если RuReviews еще не скачан:

```powershell
python .\scripts\download_rureviews.py
```

6. Быстрый пробный прогон на части датасета:

```powershell
python .\scripts\train_rubert_sentiment_gpu.py --limit 15000 --epochs 2 --batch-size 16
```

7. Финальный прогон для ВКР:

```powershell
python .\scripts\train_rubert_sentiment_gpu.py --epochs 3 --batch-size 16
```

По умолчанию используется `cointegrated/rubert-tiny2`: он быстро обучается и хорошо подходит для демонстрации на RTX 3070. Если нужно качество выше и есть время, можно запустить более тяжелый вариант:

```powershell
python .\scripts\train_rubert_sentiment_gpu.py --model-name DeepPavlov/rubert-base-cased --epochs 3 --batch-size 8 --gradient-accumulation-steps 2
```

Результат сохраняется в:

```text
models/rubert_sentiment/
```

В этой папке будет сама модель Hugging Face и файл:

```text
models/rubert_sentiment/training_metadata.json
```

В `training_metadata.json` сохраняются размер датасета, train/validation/test-разбиение, устройство обучения, параметры запуска, classification report и confusion matrix. Это удобно вставлять в раздел ВКР про экспериментальную оценку.

Если папка `models/rubert_sentiment/` существует и backend запущен в окружении, где установлены `torch` и `transformers`, приложение автоматически использует RuBERT для тональности. Если модель или зависимости недоступны, backend возвращается к текущей Naive Bayes модели `models/rureviews_sentiment_model.json`.

## Обучение аспектной модели на SentiRuEval

SentiRuEval-2015 subtask 1 — корпус для аспектно-ориентированного анализа русскоязычных отзывов о ресторанах и автомобилях. Источник: https://github.com/searayeah/russian-sentiment-emotion-datasets/tree/main/SentiRuEval-2015-subtask-1.

Дополнительное предметное обоснование аспектного анализа можно взять из статьи Яндекса на Хабре: https://habr.com/ru/companies/yandex/articles/763832/. В ней описан открытый датасет из 500 тысяч отзывов организаций на Яндекс Картах и показано, что из отзывов выделяются аспекты вроде еды, персонала, обслуживания и атмосферы.

## Дообучение на датасете Яндекс Карт

Geo Reviews Dataset 2023 — открытый датасет Яндекса с 500 000 отзывов организаций. Официальный репозиторий: https://github.com/yandex/geo-reviews-dataset-2023, зеркало с файлом `geo-reviews-dataset-2023.tskv`: https://www.kaggle.com/datasets/kyakovlev/yandex-geo-reviews-dataset-2023.

Положи `geo-reviews-dataset-2023.tskv` или `geo-reviews-dataset-2023.csv` в:

```text
data/external/yandex_geo_reviews/
```

Обучить легкую модель тональности по слабой разметке из оценки:

```powershell
.\.venv-gpu\Scripts\python.exe .\scripts\train_yandex_geo_sentiment_model.py --limit 150000
```

Скрипт создает:

```text
models/yandex_geo_sentiment_model.json
```

Backend автоматически предпочитает эту модель, если файл существует. Разметка строится по правилу: 1-2 звезды — негатив, 3 — нейтрально, 4-5 — позитив. Для аспектов этот датасет можно использовать как доменную адаптацию и источник частотных терминов, но готовых aspect-labels в нем нет, поэтому размеченной базой для аспектов остается SentiRuEval.

Для тяжелого дообучения RuBERT на этом же датасете:

```powershell
.\.venv-gpu\Scripts\python.exe .\scripts\train_rubert_sentiment_gpu.py --dataset .\data\external\yandex_geo_reviews\geo-reviews-dataset-2023.tskv --dataset-format yandex-geo --limit 60000 --epochs 2 --batch-size 16
```

Полный датасет большой, поэтому для ВКР и демонстрации лучше сначала запускать `--limit 60000`, а потом при необходимости увеличить лимит.

Скачать XML-файлы:

```powershell
C:\Users\aryti\.cache\codex-runtimes\codex-primary-runtime\dependencies\python\python.exe .\scripts\download_sentirueval.py
```

Данные сохраняются в:

```text
data/external/sentirueval2015/
```

Обучить аспектную модель:

```powershell
C:\Users\aryti\.cache\codex-runtimes\codex-primary-runtime\dependencies\python\python.exe .\scripts\train_sentirueval_aspect_model.py
```

Результат сохраняется в:

```text
models/sentirueval_aspect_model.json
```

Текущий прогон:

```text
Датасет аспектных терминов: 18961
Train: 15166
Test: 3795
Accuracy: 0.778
атмосфера и место: precision=0.876 recall=0.711 f1=0.785
качество продукта: precision=0.731 recall=0.923 f1=0.816
персонал: precision=0.876 recall=0.775 f1=0.822
цена: precision=0.942 recall=0.655 f1=0.772
```

Категории SentiRuEval сопоставляются с аспектными группами приложения. Например: `Food` → `качество продукта`, `Interior` → `атмосфера и место`, `Service` → `персонал`, `Price/Costs` → `цена`. Если файл модели существует, backend подмешивает обученные аспектные признаки к словарному извлечению аспектов.

## Как развивать дальше

- заменить словарный baseline на модель RuBERT для эмоций;
- добавить sentence-transformers для группировки аспектов;
- подключить реальные источники через официальные API;
- добавить импорт CSV/XLSX;
- сохранить историю анализов в SQLite.

## Текущий статус для защиты

Проект развернут в:

```text
D:\VKR\review-emotion-aspect-analyzer
```

Backend запускается через GPU-окружение:

```powershell
cd D:\VKR\review-emotion-aspect-analyzer
powershell -ExecutionPolicy Bypass -File .\scripts\start_backend.ps1
```

Скрипт сначала выбирает интерпретатор:

```text
D:\VKR\review-emotion-aspect-analyzer\.venv-gpu\Scripts\python.exe
```

Поэтому приложение использует обученную RuBERT-модель, если папка `models/rubert_sentiment/` существует.

Расширение для Яндекс Браузера устанавливается через:

```text
browser://extensions
```

Нужно включить режим разработчика и выбрать папку:

```text
D:\VKR\review-emotion-aspect-analyzer\extension
```

Парсер расширения выделяет отзывы из нескольких источников: 2ГИС, Яндекс Еда, TopHotels, Отзовик, IRecommend и Avito. Для Avito используется отдельный строгий сборщик отзывов продавца/сделки, чтобы не смешивать отзывы с обычным текстом объявления.

### RuBERT-модель тональности

Финальная модель обучена локально на RTX 3070:

```text
base model: cointegrated/rubert-tiny2
dataset: RuReviews
dataset size: 89063
train / validation / test: 62344 / 13359 / 13360
device: NVIDIA GeForce RTX 3070
epochs: 3
batch size: 16
test accuracy: 0.7695
test macro F1: 0.7717
test precision macro: 0.7749
test recall macro: 0.7697
```

Файлы модели:

```text
models/rubert_sentiment/model.safetensors
models/rubert_sentiment/config.json
models/rubert_sentiment/tokenizer.json
models/rubert_sentiment/training_metadata.json
```

В полном отчете и popup расширения выводится блок модели: `RuBERT-tiny`, статус `active`, метрики и GPU. Это нужно показывать на защите как подтверждение, что тональность определяется обученной transformer-моделью, а не только словарными правилами.

### Демонстрационный сценарий

1. Запустить backend.
2. Открыть страницу с отзывами в Яндекс Браузере.
3. Открыть расширение `Review Intelligence`.
4. Убедиться, что статус `Backend: подключен`.
5. Нажать `Анализировать страницу`.
6. Показать найденное количество отзывов, проблемные аспекты, сильные стороны.
7. Открыть полный отчет и показать блок `ML pipeline` с метриками RuBERT.
