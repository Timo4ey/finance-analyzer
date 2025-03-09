
## Introduction
GOAL: Create an application for checking financial from multiple sources.

### Functional requirements
* Authentication
	* need mail and password
	* set JWT-token in cookies
- Construct reports 
	- report 1: all transactions grouped by type for set period
	- report 2:  biggest transaction value
	- report 3: transactions that more than it for previous period
	- report 4: transactions that have never met before
- Support downloading data from difference source (web-api, .csv, etc)
-  Reports could be export in .csv, excel, formats
- System can give recommends through analyzing expenses (base on ML models(probably any LLM))
- Combine data from multiple sources to single format

### Non-functional requirements
* The system response time to a user request should not exceed 2 seconds.
* The system should be available 99.9% of the time.
* All confidential data should be encrypted during transmission and storage.
* The system interface should be intuitive and convenient for users of all age groups.

### Components:
#### **1️⃣ Backend**

- **API Gateway**
    - **FastAPI**
    - **Redis** (optional, for caching and rate limiting)
- **Auth**
    - **Auth API**
    - **PostgreSQL**
    - **JWT / OAuth2** for authentication
- **ETL**
    - **ETL-Server**
    - **Orchestration** (Airflow, Dagster, Prefect)
    - **Monitoring** (Prometheus, Grafana, Sentry)
    - **Queue (RabbitMQ, Kafka)** — queue for ETL tasks
    - **Storage**
        - **ClickHouse** - aggregated data
        - **PostgreSQL** - raw data
    - **Redis** (for caching intermediate data, if needed)
    - **Worker** for retrieving data from bank's API (worker for each bank)
- **ML**
    - **Analyze-Worker** — analyzes expenses and provides recommendations
    - **LLM-Worker** — calls **LLM models** for category matching or other data processing
    - **Queue (RabbitMQ, Kafka)** — queue for ML tasks
    - **Retry mechanisms** (for handling failed tasks)


#### **2️⃣ Graph and alalyze

- **React + Chart.js / D3.js** — visualization data
- **Plotly / Matplotlib** — for getting reports (pdf, .csv, excel)

### Requirements
#### Backend  
* Auth
	* API
		* sing on
		* sing in
			* set JWT token to cookies
* API-GATEWAY
	* has API 
		* for integrating with bank's by web-API 
		* for downloading data from bank's account by web-API or .csv file
		* for asking LLM model about dashbord data
		* for getting aggregated data from ETL-Service
			* scope
				* date
				* categories
				* type of source
					* sources:
						* bank (bank name)
						* draft (if pay by cash)
		* for changing categories for transaction
	* communicate with ETL-Services by queue
		* queue: 
			* etl-\<bank-name\>-auth-data
			* etl-draft
	* communicate with LLM-Workers by queue

* ETL-Service
	* Listening queue
	* Extract data from resources 
	* Transform it to general format
		* Mark type of a transaction
		* Join data by transactions' type
	* Load it to databases
		* Save raw data to Postgres 
			* single database for each bank and raw
		*  Join transformed data from resources to single table in ClickHouse
	* Interface to retrieve data by source, transactions' date, transactions' type by queue
* ML
	* Listening queue
	* Analyze-Worker
		* analyzes expenses and provides recommendations
	* **LLM-Worker** — calls **LLM models** for category matching or other data processing
	

#### Frontend
* Interface to retrieve data by source, transactions' date, transactions' type
* Show graphs:
- general - show all transactions grouped by type for set period
- highlights - type of  graphs that show strange transactions, biggest transaction value, transactions that more than it for previous period, transactions that have never met before
- Balance - income minus expenses

#### C4 Context Diagram
```mermaid
%% C4 Context Diagram
C4Context
    Person(user, "User", "Interacts with the financial application to manage and analyze expenses.")
    System(apiGateway, "API Gateway", "FastAPI", "Handles API requests and routes them to appropriate services.")
    System(authAPI, "Auth API", "FastAPI", "Manages user authentication and authorization.")
    System(etlService, "ETL Service", "Python", "Extracts, transforms, and loads financial data.")
    System(mlService, "ML Service", "Python", "Analyzes expenses and provides recommendations using ML models.")
    System(frontend, "Frontend", "React + Chart.js", "Provides a user interface for data visualization and interaction.")
    SystemDb(postgresql, "PostgreSQL", "Stores raw financial data.")
    SystemDb(clickhouse, "ClickHouse", "Stores aggregated financial data.")
    SystemQueue(rabbitmq, "RabbitMQ", "Message broker for queuing ETL and ML tasks.")

    Rel(user, apiGateway, "Uses")
    Rel(apiGateway, authAPI, "Forwards authentication requests")
    Rel(apiGateway, etlService, "Routes data processing requests")
    Rel(apiGateway, mlService, "Sends data for analysis")
    Rel(apiGateway, frontend, "Delivers data to")
    Rel(etlService, postgresql, "Stores raw data")
    Rel(etlService, clickhouse, "Stores aggregated data")
    Rel(etlService, rabbitmq, "Sends processing tasks")
    Rel(mlService, rabbitmq, "Receives analysis tasks")
    Rel(mlService, clickhouse, "Reads aggregated data for analysis")

```

#### C4 Component Diagram for the ETL Service
```mermaid%% C4 Component Diagram for ETL Service
C4Component
    Container(etlService, "ETL Service", "Python", "Extracts, transforms, and loads financial data.")

    Component(etlServer, "ETL Server", "Python", "Coordinates and manages ETL processes.")
    Component(queueListener, "Queue Listener", "Python", "Listens for new tasks in RabbitMQ queues.")
    Component(dataExtractor, "Data Extractor", "Python", "Extracts data from various sources like bank APIs and CSV files.")
    Component(dataTransformer, "Data Transformer", "Python", "Transforms raw data into a standardized format.")
    Component(dataLoader, "Data Loader", "Python", "Loads transformed data into storage systems.")
    Component(bankAPI1, "Bank API 1", "API", "Interface to Bank API 1 for data extraction.")
    Component(bankAPI2, "Bank API 2", "API", "Interface to Bank API 2 for data extraction.")
    Component(csvImport, "CSV Import", "Python", "Handles CSV file imports for data extraction.")
    Component(transactionCategorizer, "Transaction Categorizer", "Python", "Categorizes financial transactions based on predefined rules.")
    Component(dataAggregator, "Data Aggregator", "Python", "Aggregates data for reporting purposes.")

    %% Define external systems for clarity
    SystemDb(postgresql, "PostgreSQL", "Database", "Stores raw financial transaction data.")
    SystemDb(clickhouse, "ClickHouse", "Database", "Stores aggregated financial data.")
    SystemQueue(rabbitmq, "RabbitMQ", "Message Broker", "Manages queues for ETL tasks.")

    Rel(etlService, etlServer, "Manages ETL processes")
    Rel(etlService, queueListener, "Monitors RabbitMQ for tasks")
    Rel(etlService, dataExtractor, "Extracts data from various sources")
    Rel(etlService, dataTransformer, "Transforms extracted data")
    Rel(etlService, dataLoader, "Loads data into storage")
    Rel(dataExtractor, bankAPI1, "Extracts data from Bank API 1")
    Rel(dataExtractor, bankAPI2, "Extracts data from Bank API 2")
    Rel(dataExtractor, csvImport, "Imports data from CSV files")
    Rel(dataTransformer, transactionCategorizer, "Categorizes transactions")
    Rel(dataTransformer, dataAggregator, "Aggregates data")
    Rel(dataLoader, postgresql, "Stores raw data")
    Rel(dataLoader, clickhouse, "Stores aggregated data")
    Rel(queueListener, rabbitmq, "Listens for new tasks")

```


#### C4 Container Diagram
```mermaid
%% C4 Container Diagram
C4Container
    Person(user, "User", "A user of the financial application")
    Container(frontend, "Frontend", "React + Chart.js", "User interface for financial data management and visualization")
    Container(apiGateway, "API Gateway", "FastAPI", "Handles incoming API requests and routes them to backend services")
    Container(authAPI, "Auth API", "FastAPI", "Manages user authentication and authorization processes")
    Container(etlService, "ETL Service", "Python", "Extracts, transforms, and loads financial data from various sources")
    Container(mlService, "ML Service", "Python", "Analyzes financial data and provides recommendations using ML models")
    ContainerDb(postgresql, "PostgreSQL", "Database", "Stores raw financial transaction data")
    ContainerDb(clickhouse, "ClickHouse", "Database", "Stores aggregated financial data for quick retrieval")
    ContainerQueue(rabbitmq, "RabbitMQ", "Message Broker", "Manages queues for ETL and ML tasks")
    Container(redis, "Redis", "Cache", "Caches intermediate data to improve performance")

    Rel(user, frontend, "Uses")
    Rel(frontend, apiGateway, "Sends API requests")
    Rel(apiGateway, authAPI, "Forwards authentication requests")
    Rel(apiGateway, etlService, "Routes data processing requests")
    Rel(apiGateway, mlService, "Sends data for analysis")
    Rel(etlService, postgresql, "Stores raw data")
    Rel(etlService, clickhouse, "Stores aggregated data")
    Rel(etlService, rabbitmq, "Pushes processing tasks")
    Rel(mlService, rabbitmq, "Pulls analysis tasks")
    Rel(mlService, clickhouse, "Accesses aggregated data for analysis")
    Rel(etlService, redis, "Caches intermediate data")
    Rel(mlService, redis, "Utilizes cached data for analysis")

```

```
financial-app/
│── backend/                 # Бэкенд-сервисы
│   ├── api-gateway/         # API Gateway (FastAPI)
│   ├── auth-service/        # Authentication service (FastAPI + PostgreSQL)
│   ├── etl-service/         # ETL сервис (Python + Airflow/Dagster/Prefect)
│   ├── ml-service/          # ML сервис (LLM, рекомендации, классификация категорий)
│   ├── shared/              # Общий код (модели, utils)
│── frontend/                # Фронтенд-приложение
│   ├── src/                 # Исходный код (React + Chart.js / D3.js)
│   ├── public/              # Статика
│   ├── tests/               # Тесты
│── infra/                   # Инфраструктура проекта
│   ├── docker/              # Dockerfiles и docker-compose
│   ├── k8s/                 # Kubernetes manifests
│   ├── terraform/           # Terraform (если облако)
│   ├── monitoring/          # Prometheus, Grafana, Sentry
│── scripts/                 # Утилитарные скрипты (запуск, миграции, отладка)
│── docs/                    # Документация (C4-модели, API, требования)
│── .github/                 # CI/CD пайплайны GitHub Actions
│── Makefile                 # Команды для сборки и деплоя
│── README.md                # Описание проекта
│── .gitignore               # Исключения для Git
│── .env.example             # Пример переменных окружения

```