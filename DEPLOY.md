# Deployment Guide

## Описание проекта

Это небольшой Go API-сервис с PostgreSQL, содержащий несколько endpoint'ов:

- /health
- /hello
- GET /user/{id}
- POST /user

Приложение контейнеризовано с помощью Docker и автоматически деплоится на виртуальную машину через GitHub Actions.

## Архитектура решения

### Что где работает

- Приложение Go API запускается в Docker-контейнере
- PostgreSQL установлен на хосте виртуальной машины, а не в контейнере
- CI/CD реализован через GitHub Actions
- Для деплоя используется self-hosted runner, установленный прямо на ВМ

### Сетевое решение

Так как PostgreSQL работает не в контейнере, а на хосте, контейнеру нельзя использовать localhost для подключения к БД.

Внутри контейнера localhost указывает на сам контейнер, а не на хостовую систему.

Поэтому для подключения к PostgreSQL используется адрес Docker bridge host:

```env
DB_HOST=172.17.0.1
```

Этот адрес позволяет контейнеру обращаться к хосту, где запущен PostgreSQL.


## Установка и настройка окружения

### Установить Docker:
```bash
sudo apt update
sudo apt upgrade -y
sudo apt install docker.io -y
sudo systemctl start docker
sudo systemctl enable docker
```

### Добавить пользователя в группу docker:
```bash
sudo usermod -aG docker $USER
```

### Перезапустить сессию (или WSL):
```bash
wsl --shutdown
```

### Установить GO:
```bash
sudo snap install go --classic
```

### Установить PostgreSQL:
```bash
sudo apt install postgresql postgresql-contrib -y
sudo systemctl start postgresql
sudo systemctl enable postgresql
```

### Настроить доступ (pg_hba.conf):
```conf
host    all     all     172.17.0.0/16    md5
```

### Включить прослушивание всех адресов (postgresql.conf):
```conf
listen_addresses = '*'
```

### Создать базу данных:
```bash
sudo -u postgres psql
CREATE DATABASE testdb;
ALTER USER postgres WITH PASSWORD 'postgres';
\q
```

### Перезапустить PostgreSQL:
```bash
sudo service postgresql restart
```

### Применить миграции:
```bash
sudo -u postgres psql -d testdb -f migrations/001_create_users.sql
```

### Установить зависимости:
```bash
go mod tidy
```


## Dockerfile (объяснение)

Используется multistage build:

1 этап (builder):
- Используется образ golang:1.26-alpine
- Скачиваются зависимости (go mod download)
- Собирается бинарник приложения

2 этап (runtime):
- Используется минимальный образ alpine
- Копируется только готовый бинарник
- Запускается приложение

Преимущества:
- Маленький размер образа
- Отсутствие лишних зависимостей
- Быстрая сборка


### Сборка Docker-образа
```bash
docker build -t go .
```


### Запуск контейнера
```bash
docker run -d --name go \
-p 8080:8080 \
-e DB_HOST=172.17.0.1 \
-e DB_PORT=5432 \
-e DB_USER=postgres \
-e DB_PASSWORD=postgres \
-e DB_NAME=testdb \
go
```


## Проверка работы

### Проверка контейнера:
```bash
docker ps
```

### Проверка API:
```bash
curl http://localhost:8080/health
curl http://localhost:8080/hello
curl http://localhost:8080/user/1
curl -X POST http://localhost:8080/user \
  -H "Content-Type: application/json" \
  -d '{"name":"John Doe","email":"john@example.com"}'
```


## CI/CD Pipeline (GitHub Actions)

Pipeline запускается при каждом push в ветку main.

### Этапы:

1. lint
- Проверка форматирования (gofmt)
- Анализ кода (go vet)

2. build
- Сборка Go-приложения

3. docker
- Сборка Docker-образа

4. deploy
- Остановка старого контейнера
- Запуск нового контейнера с актуальной версией


### Использование секретов

В GitHub добавлены secrets:

- DB_HOST
- DB_PORT
- DB_USER
- DB_PASSWORD
- DB_NAME
- DB_SSLMODE

Они передаются в контейнер как переменные окружения:

```yaml
-e DB_HOST=${{ secrets.DB_HOST }}
```

Это позволяет не хранить пароли в репозитории.


## Self-hosted runner

### Установка runner
```bash
mkdir -p ~/actions-runner && cd ~/actions-runner
curl -o actions-runner-linux-x64-2.333.1.tar.gz -L https://github.com/actions/runner/releases/download/v2.333.1/actions-runner-linux-x64-2.333.1.tar.gz
tar xzf ./actions-runner-linux-x64-2.333.1.tar.gz
```

### Настройка и запуск
```bash
./config.sh --url https://github.com/rusyandr/go-api-postgres --token APOFXTPMUSJVRQQ6GFJPDN3JZZSIU
./run.sh
```


## Как работает деплой (объяснение)

1. Разработчик изменяет код (например, endpoint /hello)
2. Делает commit и push в GitHub
3. GitHub запускает pipeline
4. Self-hosted runner получает задачу
5. Runner:
- скачивает код
- собирает приложение
- собирает Docker-образ
- перезапускает контейнер
6. Новый контейнер начинает работать с обновлённым кодом


# Итог

В результате получена система, где:

- Приложение работает в Docker
- База данных работает на хосте
- Контейнер подключается к базе через bridge-сеть
- Деплой полностью автоматизирован через CI/CD