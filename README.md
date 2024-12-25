# Система предсказания спроса на образовательные программы

Система предназначена для хранения, анализа и прогнозирования спроса на образовательные программы. 

Центральная часть системы – таблицы **EDUCATIONAL_PROGRAMS**, **HISTORICAL_ENROLLMENTS**, **PREDICTION_MODELS**, **PREDICTION_RUNS** и **PREDICTION_RESULTS**, обеспечивающие механизм сбора исторических данных по зачислениям, запуск прогнозных моделей и хранение результатов.

Дополнительные таблицы **STUDENTS**, **DEPARTMENTS** и **PROGRAM_CATEGORIES** хранят справочную информацию: данные о студентах, кафедрах/факультетах и типах программ.

```mermaid
erDiagram
    %% Основные таблицы
    DEPARTMENTS ||--o{ EDUCATIONAL_PROGRAMS : "offers"
    DEPARTMENTS {
        uuid department_id PK
        varchar(128) department_name
        text description
        boolean is_active
        timestamptz created_at
    }

    PROGRAM_CATEGORIES ||--o{ EDUCATIONAL_PROGRAMS : "classifies"
    PROGRAM_CATEGORIES {
        uuid category_id PK
        varchar(128) category_name
        text description
        boolean is_active
        timestamptz created_at
    }

    EDUCATIONAL_PROGRAMS ||--o{ HISTORICAL_ENROLLMENTS : "recorded_in"
    EDUCATIONAL_PROGRAMS {
        uuid program_id PK
        uuid department_id FK
        uuid category_id FK
        varchar(128) program_name
        text description
        int2 duration_months
        boolean is_active
        timestamptz created_at
    }
    
    STUDENTS ||--o{ HISTORICAL_ENROLLMENTS : "enrolls"
    STUDENTS {
        uuid student_id PK
        varchar(128) full_name
        date birth_date
        varchar(128) email
        timestamptz registered_at
        boolean is_active
        timestamptz last_update
    }
    
    HISTORICAL_ENROLLMENTS {
        uuid enrollment_id PK
        uuid student_id FK
        uuid program_id FK
        timestamptz enrollment_date
        boolean completed
        timestamptz completion_date
        text grade
    }

    %% Таблицы для предиктивных моделей
    PREDICTION_MODELS ||--o{ PREDICTION_RUNS : "used_in"
    PREDICTION_MODELS {
        uuid model_id PK
        varchar(128) model_name
        text model_description
        text model_parameters
        boolean is_active
        timestamptz created_at
        timestamptz last_updated
    }
    
    PREDICTION_RUNS ||--o{ PREDICTION_RESULTS : "generates"
    PREDICTION_RUNS {
        uuid run_id PK
        uuid model_id FK
        timestamptz run_date
        varchar(16) status
        text run_details
    }
    
    PREDICTION_RESULTS {
        uuid result_id PK
        uuid run_id FK
        uuid program_id FK
        integer predicted_demand
        decimal confidence_score
        text additional_info
    }
```
## Сценарий добавления новой образовательной программы
При создании новой образовательной программы требуется указать кафедру (факультет), категорию программы, длительность обучения и другую необходимую информацию. Система проверяет, существует ли указанная кафедра, и в случае отсутствия выдает ошибку. Если программа успешно создается, данные вносятся в таблицы EDUCATIONAL_PROGRAMS.

```mermaid
 sequenceDiagram
    participant Adm as Admin (Methodist)
    participant App as Application
    participant DB as Database

    Adm->>+App: createNewProgram(program_data)
    App->>DB: CHECK department_exists
    alt Department Not Found
        App-->>Adm: Error: DepartmentNotFound
    else Department Found
        App->>DB: CHECK category_exists
        alt Category Not Found
            App-->>Adm: Error: CategoryNotFound
        else Category Found
            App->>DB: INSERT INTO EDUCATIONAL_PROGRAMS
            App-->>Adm: NewProgramCreated
        end
    end
```
# Сценарий запуска предиктивной модели
Процесс запуска прогнозной модели спроса на образовательную программу включает подготовку исторических данных, выбор активной модели и создание записи о запуске (PREDICTION_RUNS). После расчёта результатов система сохраняет их в PREDICTION_RESULTS.
```mermaid
 sequenceDiagram
    participant S as Scheduler
    participant A as Analytics Service
    participant M as Model Engine
    participant DB as Database

    S->>+A: initiatePredictionRun(model_id)
    A->>DB: SELECT model_info FROM PREDICTION_MODELS
    alt Model Inactive or Not Found
        A-->>S: Error: ModelNotFoundOrInactive
    else
        A->>DB: INSERT INTO PREDICTION_RUNS (status='in_progress')
        A->>DB: GET historical_enrollments, programs_data
        A->>+M: runPrediction(model_parameters, historical_data)
        M-->>A: predictions_list

        A->>DB: UPDATE PREDICTION_RUNS (status='completed')
        
        loop Each Prediction
            A->>DB: INSERT INTO PREDICTION_RESULTS (predicted_demand, confidence_score, ...)
        end

        A-->>-S: PredictionsReady
    end
```

# Краткое описание основных таблиц

- **DEPARTMENTS**:
  Хранит информацию о кафедрах (факультетах), на которых открываются образовательные программы.

- **PROGRAM_CATEGORIES**:
Позволяет классифицировать образовательные программы по типу (бакалавриат, магистратура, курсы повышения квалификации и т. д.).

- **EDUCATIONAL_PROGRAMS**:
Основная сущность, описывающая каждую образовательную программу (название, описание, длительность, статус).

- **STUDENTS**:
Содержит данные о студентах, которые в дальнейшем анализируются для выявления исторических данных по зачислениям.

- **HISTORICAL_ENROLLMENTS**:
Исторические записи о зачислениях и прохождении студентами программ. Используются для обучения моделей.

- **PREDICTION_MODELS**:
Хранит информацию о доступных моделях прогнозирования (название, параметры, последнее обновление).

- **PREDICTION_RUNS**:
Запись о каждом запуске модели (дата запуска, статус и дополнительные детали).

- **PREDICTION_RESULTS**:
Содержит результаты работы модели по каждой программе (прогнозируемый спрос, степень уверенности и др.).


