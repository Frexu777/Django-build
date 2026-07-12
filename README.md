# 🚀 Django REST API + Docker Compose

Ten projekt przedstawia konfigurację backendu opartego na **Django + Django REST Framework**, uruchamianego w kontenerach Docker z bazą PostgreSQL.

# 📁 Project structure

```text

├── mysite/
│   ├── manage.py
│   ├── mysite/
│   │   ├── __init__.py
│   │   ├── settings.py
│   │   ├── urls.py
│   │   ├── asgi.py
│   │   └── wsgi.py
│   │
│   ├── polls/
│   │   ├── __init__.py
│   │   ├── models.py
│   │   ├── views.py
│   │   ├── serializers.py
│   │   ├── urls.py
│   │   └── apps.py
│   │
│   ├── requirements.txt
|   ├── docker-compose.yml
|    ├── .env
│   └── Dockerfile    
└── README.md
```

## 🐳 Uruchomienie projektu

### 1. Build i start kontenerów

```bash
docker compose up --build

docker compose exec web python manage.py makemigrations
docker compose exec web python manage.py migrate
docker compose exec web python manage.py createsuperuser


| Usługa       | URL                                                        |
| ------------ | ---------------------------------------------------------- |
| Django API   | [http://localhost:8000](http://localhost:8000)             |
| Django Admin | [http://localhost:8000/admin](http://localhost:8000/admin) |
| PostgreSQL   | localhost:5432                                             |


DEBUG=1
SECRET_KEY=supersecretkey

DB_NAME=mydb
DB_USER=myuser
DB_PASSWORD=mypassword
DB_HOST=db
DB_PORT=5432


FROM python:3.11-slim

ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

WORKDIR /app

RUN apt-get update && apt-get install -y \
    build-essential \
    libpq-dev \
    && rm -rf /var/lib/apt/lists/*

COPY requirements.txt .

RUN pip install --no-cache-dir -r requirements.txt

COPY . .

CMD ["gunicorn", "myproject.wsgi:application", "--bind", "0.0.0.0:8000"]


version: "3.9"

services:
  web:
    build: ./backend
    container_name: django_api
    command: python manage.py runserver 0.0.0.0:8000
    volumes:
      - ./backend:/app
    ports:
      - "8000:8000"
    env_file:
      - .env
    depends_on:
      - db

  db:
    image: postgres:15
    container_name: postgres_db
    environment:
      POSTGRES_DB: mydb
      POSTGRES_USER: myuser
      POSTGRES_PASSWORD: mypassword
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"

volumes:
  postgres_data:


Django>=5.0
djangorestframework
gunicorn
psycopg2-binary

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': os.getenv('DB_NAME'),
        'USER': os.getenv('DB_USER'),
        'PASSWORD': os.getenv('DB_PASSWORD'),
        'HOST': os.getenv('DB_HOST'),
        'PORT': os.getenv('DB_PORT'),
    }
}


INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',

## Architektura CI/CD (GitHub Actions)

Projekt wykorzystuje GitHub Actions do automatyzacji testów, dbania o jakość kodu oraz wdrażania aplikacji (CI/CD). Pipeline został zoptymalizowany pod kątem szybkości i składa się z równoległych zadań.

### 1. Opis Workflowów
*   **Django CI (`ci.yml`)**: Buduje aplikację, konfiguruje bazę PostgreSQL, wykonuje migracje i uruchamia podstawowe testy Django.
*   **Testy PostgreSQL (`tests.yml`)**: Uruchamia dedykowany kontener w celu przetestowania połączenia i operacji bezpośrednio na bazie danych.
*   **Celery i Redis CI (`celery.yml`)**: Stawia środowisko asynchroniczne (Redis + worker Celery w tle) i weryfikuje poprawne wykonywanie tasków.
*   **Linting i Formatowanie (`lint.yml`)**: Sprawdza jakość kodu Pythona, wyszukując błędy składniowe i weryfikując standardy formatowania.
*   **Skanowanie bezpieczeństwa (`security.yml`)**: Analizuje kod pod kątem potencjalnych luk w zabezpieczeniach.
*   **Automatyczny Deploy (`deploy.yml`)**: Buduje docelowy obraz Dockerowy i wdraża aplikację na serwer produkcyjny.

### 2. Wymagane Sekrety (GitHub Secrets)
Aby pipeline mógł połączyć się z usługami, w repozytorium (`Settings` -> `Secrets and variables` -> `Actions`) muszą znajdować się następujące klucze:
*   `POSTGRES_DB` - nazwa bazy danych do testów.
*   `POSTGRES_USER` - użytkownik bazy danych.
*   `POSTGRES_PASSWORD` - hasło użytkownika.
*   `SECRET_KEY` - klucz szyfrujący środowiska Django.

### 3. Proces Deployu (Wdrożenie)
Wdrożenie aplikacji jest w pełni zautomatyzowane, ale daje też pełną kontrolę manualną:
*   **Automatycznie (Push):** Każda zmiana wypchnięta na główną gałąź (`main`) automatycznie uruchamia proces budowy i wysyłki obrazu. System ignoruje jedynie zmiany w plikach tekstowych (np. `.md`).
*   **Ręcznie / Rollback:** Wdrożenie można wywołać ręcznie z zakładki **Actions** -> **Automatyczny Deploy** -> **Run workflow**. Użytkownik ma tam możliwość podania wartości **SHA commita**, aby wykonać natychmiastowy Rollback (powrót do starszej, stabilnej wersji).

### 4. Uruchamianie Pipeline'u (Dla nowych programistów)
Aby uruchomić pełen zestaw testów, nie musisz instalować lokalnie żadnych narzędzi CI. Wystarczy, że:
1. Sklonujesz repozytorium na swój komputer.
2. Wprowadzisz zmiany w kodzie na własnej gałęzi lub na `main`.
3. Wykonasz standardowe komendy: `git add .`, `git commit -m "opis zmian"`, a następnie `git push`.
4. Przejdziesz do zakładki **Actions** na GitHubie – pipeline uruchomi się i przetestuje Twój kod automatycznie! Starsze, nieukończone testy z Twoich poprzednich commitów zostaną automatycznie anulowane, aby oszczędzać zasoby.

### 5. Debugowanie błędów
Jeśli któryś z etapów CI/CD zakończy się błędem (czerwony status):
1. Przejdź do zakładki **Actions** i kliknij w nieudany workflow.
2. Wybierz konkretne zadanie (Job) z menu po lewej stronie.
3. Rozwiń krok oznaczony czerwonym krzyżykiem.
4. Przeanalizuj logi z terminala. Do najczęstszych problemów należą: literówki w kodzie (wykrywane przez Linter), brak pakietu w pliku `requirements.txt` lub niezaktualizowane ścieżki do plików.
    'rest_framework',
    'api',
]
