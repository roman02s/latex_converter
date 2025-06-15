# PDF → LaTeX Converter

**Содержание**  
1. [Обзор](#обзор)  
2. [Структура проекта](#структура-проекта)  
3. [Установка и зависимости](#установка-и-зависимости)  
4. [Конфигурация](#конфигурация)  
   - [Файл `config/application.yaml`](#файл-configapplicationyaml)  
   - [Шаблоны prompt’ов (config/prompts)](#шаблоны-promptов-configprompts)  
   - [Примеры настроек для разных стадий](#примеры-настроек-для-разных-стадий)  
   - [Добавление собственных обработчиков](#добавление-собственных-обработчиков)  
   - [Переменные окружения и файл `.env`](#переменные-окружения-и-файл-env)  
5. [Архитектура и Pipeline](#архитектура-и-pipeline)  
6. [API – FastAPI](#api--fastapi)  
7. [Запуск и пример использования](#запуск-и-пример-использования)  
8. [Очистка и временные файлы](#очистка-и-временные-файлы)  
9. [Кеш и директория `cache/`](#кеш-и-директория-cache)  
10. [Результирующий ZIP-архив](#результирующий-zip-архив)  

---

## Обзор

Это сервис на FastAPI, который:

- Принимает на вход PDF-файл (скан или документ).  
- Преобразует каждую страницу в изображение, проводит OCR (текст, таблицы, формулы).  
- Опционально — прогоняет результат через AI-модель от OpenAI (например, `o4-mini`) или вашу собственную кастомную модель по настраиваемому prompt’у.  
- Собирает LaTeX-разметку по страницам и упаковывает всё в ZIP-архив с `.tex`-файлами.  

---

## Структура проекта

```

.
├── app/                    # Исходники сервиса
│   ├── main.py             # Точка входа FastAPI
│   ├── api/                # Определение HTTP-роутов
│   └── converter/          # Логика конвертации
│       ├── service.py      # Класс Converter + сохранение/загрузка
│       ├── pipeline/       # Определение Pipeline и Stage’ов
│       └── utils/          # Вспомогательные функции
├── config/
│   ├── application.yaml    # Основной YAML-конфиг пайплайна
│   └── prompts/            # Шаблоны prompt’ов для AIExtractor
├── cache/                  # Кеш извлечённых формул и др.
│   └── formulas/           # Файлы кеша для FormulasExtractor
├── downloads/              # Результирующие ZIP-архивы
├── temp/                   # Временные файлы (изображения страниц и пр.)
├── .env                    # Ваши переменные окружения
├── .gitignore
├── requirements.txt        # Python-зависимости
└── README.md               # Эта документация

````

---

## Установка и зависимости

1. **Клонируйте репозиторий** и перейдите в папку:
   ```bash
   git clone https://github.com/kr4cket/latex_converter
   cd latex_converter
    ```

2. **Установка и запуск**

   ```bash
   docker compose up -d --build
   ```
---

## Конфигурация

### Файл `config/application.yaml`

```yaml
# Шаблоны prompt’ов в config/prompts/
prompts_dir: "config/prompts"

pipeline:
  preprocessors:
    - name: ImagePreprocessor
      params:
        directory: "temp/img"
        output_prefix: "page_"
    - name: PDFPreprocessor
      params:
        directory: "temp/pdf"
        output_prefix: "page_"

  stages:
    - name: MarkerPdfTextExtractor
      params:
        use_llm: false
        output_format: "json"
        languages: "en,ru"
        force_ocr: true

    - name: MarkerPdfTablesExtractor
      params:
        use_llm: false
        output_format: "json"
        languages: "en,ru"
        force_ocr: true

    - name: FormulasExtractor
      params:
        required_symbols: [ '\\', '{', '}', '_', '^' ]
        math_keywords: [ "int", "sum", "lim", "alpha", "beta" ]
        cache_dir: "cache/formulas"

    - name: AIExtractor
      params:
        model: "o4-mini"             # OpenAI-модель
        api_key: "${API_KEY}"        # из .env
        proxies:
          http: "${HTTP_PROXY}"      # опционально
        text: "MarkerPdfTextExtractor" # ключ обработчика, который необходим для подстановки в промпт
        tables: "MarkerPdfTablesExtractor" # ключ обработчика, который необходим для подстановки в промпт
        formulas: "FormulasExtractor" # ключ обработчика, который необходим для подстановки в промпт

    - name: TexExporter
      params:
        output_format: "zip"         
        output_prefix: "page_"
```

### Шаблоны prompt’ов (config/prompts)

Вся логика AIExtractor берет свои шаблоны из папки `config/prompts/`.

* **Где хранятся**
  Файлы лежат в `config/prompts/` рядом с `application.yaml`.

Все шаблоны prompt’ов для моделей (в данной реализации **AIExtractor**) называются в точности так же, как ваши модели (а не по типам «text», «tables», «formulas»).

  Каталог:  
```
    config/
    └── prompts/
    ├──── o4-mini.json
    ├──── custom-v1.json
    └──── my-model.json
```

---

### Примеры настроек для разных стадий

* **ImagePreprocessor**
  - directory — куда сохранять PNG-изображения
  - output_prefix — префикс имени файла

* **MarkerPdfTextExtractor / MarkerPdfTablesExtractor**
  - use_llm — включить LLM-проверку (true/false)
  - output_format — формат вывода (json, text)
  - languages — языки для OCR (en,ru)
  - force_ocr — принудительно запускать OCR, даже при наличии встроенного текста

* **FormulasExtractor**
  - required_symbols — символы, характерные для формул
  - math_keywords — слова-ключи (int, sum, lim и др.)
  - cache_dir — папка для кеширования распознанных формул

* **AIExtractor**
  - model — название модели (локальная o4-mini или OpenAI)
  - api_key, proxies — доступ к OpenAI API
  - text, tables, formulas — имена prompt-файлов из config/prompts

* **TexExporter**
  - directory — выводная папка для .tex
  - output_prefix — префикс имени страницы
  - result_data_key — из какого ключа брать LaTeX-код

---

### Добавление собственных обработчиков

1. Наследуйте базовый класс `Stage`:

   ```python
   from app.converter.stage.stage import Stage

   class CustomStage(Stage):
       def __init__(self, foo: str, bar: int):
           self.foo = foo
           self.bar = bar

       def process(self, page_data: dict) -> Any:
           # ваша логика
           return result
   ```
2. Зарегистрируйте в `app/converter/stage/container.py`:

   ```python
   CLASS_MAP["CustomStage"] = "app.converter.pipeline.my_module"
   ```
3. Добавьте в `application.yaml`:

   ```yaml
   pipeline:
     stages:
       - name: CustomStage
         params:
           foo: "value"
           bar: 123
   ```

---

### Переменные окружения и файл `.env`

Переименуйте `.env-example` в **`.env`** и укажите:

```dotenv
API_KEY=<ваш_ключ_OpenAI>
HTTP_PROXY=http://proxy:3128
```

Переменные можно вставлять в `application.yaml` через `${VARNAME}`.

---

## Архитектура и Pipeline

```python
class Stage(ABC):
    @abstractmethod
    def process(self, data: dict) -> Any: ...
    def get_name(self) -> str: return self.__class__.__name__

pipeline = Pipeline()
pipeline.set_file_path("document.pdf")
pages_data = pipeline.prepare()  # препроцессоры → изображения/страницы
pages_data = pipeline.run()      # stages → OCR/AI/форматирование
```

---

## API – FastAPI

**POST** `/api/v1/convert`

* Принимает `file` (multipart/form-data) — PDF
* Возвращает **201**:

  ```json
  {
    "description": "Data converted!",
    "download_url": "/api/v1/convert/<file_id>"
  }
  ```

**GET** `/api/v1/convert/{file_id}`

* Возвращает ZIP-архив (`application/zip`) с готовыми `.tex`-файлами

Ошибки: `500 Internal Server Error`

---

## Запуск и пример использования

```bash
uvicorn app.main:app --reload --host localhost --port 8080 --timeout-keep-alive 7200

# Конвертация
curl -F "file=@document.pdf" http://localhost:8000/api/v1/convert

# Скачивание результата
curl http://localhost:8000/api/v1/convert/<file_id> --output result.zip
```

---

## Очистка и временные файлы

* `temp/` хранит промежуточные изображения и PDF.
* `Converter.cleanup()` удаляет временные папки и файлы.
* Финальные ZIP-архивы в `downloads/` **не удаляются** автоматически.

---

## Кеш и директория `cache/`

* `FormulasExtractor` кеширует результаты в `cache/formulas/`.
* Путь кеша можно настроить через `cache_dir` в `application.yaml`.

---

## Результирующий ZIP-архив

После конвертации получаете единственный ZIP-файл в папке `downloads/`, внутри которого для каждой страницы лежит соответствующий `.tex`-файл:

```
downloads/
└── <file_id>.zip
    ├── page_1.tex
    ├── page_2.tex
    └── …
```

---

> **Гибкость и расширяемость**
> Модульная структура позволяет подключать любые OCR-движки, модели OpenAI и кастомные LLM, свои шаблоны prompt’ов и добавлять новые стадии без изменения ядра.

**Удачной конвертации!**


