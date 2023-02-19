<br/>

  <h1 align="center">Deploy script приложений</h1>

  <p align="center">
    Описание flow деплоя приложения script
    <br/>
    <br/>
  </p>

## О проекте

Проект состоит из:
- Дешборда построеного на React  <a align="center" href="https://github.com/script696/script-panel" target="_blank">script-panel</a>
- Бэкент построеного на Node.js  <a align="center" href="https://github.com/script696/script-api" target="_blank">script-api</a>

## Задача
Собрать три докер контейнера из фронта, бэка и базы
Доставить контейнеры на вируальную машину
Запустить контейнеры на вируальной машине

### Ручной деплой

**Регистрация на Docker Hub**

1. Регистрируемся на dockerhub
2. На винде 10 пришлось также установить WSL2
3. При наличии в проекте файлов с разделителями отличными от CRLF докер падает с ошибкой


**Подготовка локального окружения**

1. Устанавливаем на локальную машину Docker с офф сайта
2. На винде 10 пришлось также установить WSL2
3. При наличии в проекте файлов с разделителями отличными от CRLF докер падает с ошибкой
4. Логинимся в docker hub
```
docker login
```

**Подготовка виртуальной машины**
1. Устанавлием на VPS  докер в соотв с офф докой 
[docker install ubuntu](https://docs.docker.com/engine/install/ubuntu/)
2. Прописываем SSH ключ

**Описание флоу ручного деплоя**
1. Собираем образы frontend и backend на локальной машине
2. Пушим образы в docker hub
3. Заходим на VPS и в директории /home/scipt696 создаем файл docker-compose.yml
4. Запускаем 
```
docker compose up
```
5. Докер запулит образы фронта, бэка и базы с docker hub  в соответствии с конфигурацией файла 
и соберет кластер

**Создаем образ фронта**
1. Создаем Dockerfile в коре проекта script-panel
```
FROM node:alpine

WORKDIR /app

COPY package*.json .

RUN npm ci --silent

COPY . .

RUN npm run build

EXPOSE 3000

CMD ["npm", "run", "start"]
```
2. Собираем докер image 
```
docker build -t script696/script-panel
```
_script696 - никнейм в моем профиле на doker hub должен быть обязательно такой_
3. Пушим имедж в докерхаб 
```
docker push script696/script-panel
```

**Создаем образ бэка**
1. Создаем Dockerfile в коре проекта script-panel
```
FROM node

WORKDIR /app

COPY package*.json .

RUN npm ci

COPY . .

EXPOSE 5000

CMD ["npm", "run", "start:prod"]
```
2. Собираем докер image
```
docker build -t script696/script-api
```

3. Пушим имедж в докерхаб
```
docker push script696/script-api
```
**Заходим на VPS**
1. Переходим в папку /home/script696
2. Создаем файл docker-compose.yml
3. Открываем файл и пишем инструкции для сбокрки кластера
```
version: '3.9'

services:
  # MongoDB service
  mongo_db:
    container_name: db_container
    image: mongo:latest
    restart: always
    volumes:
      - mongo_db:/data/db
  # Script-api service
  api:
    container_name: script_api_container
    image: script696/api_image:stage
    ports:
      - 5000:5000
    environment:
      PORT: 5000
      MONGODB_URI: mongodb://mongo_db:27017
      DB_NAME: script-api
      ORIGIN: http://46.19.67.82:3000
    depends_on:
      - mongo_db
  # Admin panel service
  front:
    container_name: script_admin_panel
    image : script696/script_panel:latest
    ports:
      - 3000:3000
    environment:
      PORT: 3000
      BROWSER: none
      REACT_APP_BASE_URL: http://46.19.67.82:5000
    depends_on:
      - api

volumes:
  mongo_db : {}
```
4. Запускаем 
```
docker compose up
```

### Автоматический деплой (CD)
**Описание флоу автоматического деплоя**
1. На локальной машине настраиваем github actions workflows
2. Настраиваем срабатывание экшена деплоя на релиз
3. Запускаем экшен на гитхабе
4. В раннере gh создается docker image приложения и пушится на docker hub
5. Далее ранер подключается к VPS по SSH, пулит имейджи с docker hub и запускает docker compose

**Настраиваем экшен для фронта**
1. в ./github/workflows создаем release.yml
```
name: Deploy project
on:
  release:
    types:
      - created

jobs:
  push_to_docker_hub:
    runs-on: ubuntu-latest
    steps:
      -
        name: Checkout
        uses: actions/checkout@v3
      -
        name: Login to Docker Hub
        uses: docker/login-action@v2
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}
      -
        name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v2
      -
        name: Build and push
        uses: docker/build-push-action@v4
        with:
          context: .
          file: ./Dockerfile
          push: true
          tags: ${{ secrets.DOCKERHUB_USERNAME }}/${{ secrets.PROJECT_NAME }}:latest

  deploy_on_vps:
    needs: push_to_docker_hub
    runs-on: ubuntu-latest
    steps:
      - name: executing remote ssh commands using password
        uses: appleboy/ssh-action@v0.1.7
        with:
          host: ${{ secrets.HOST }}
          username: ${{ secrets.USERNAME }}
          key: ${{ secrets.SSH_KEY }}
          port: ${{ secrets.SSH_PORT }}
          passphrase : ${{ secrets.PASSPHRASE }}
          script: |
            cd /home/script696
            docker compose pull
            docker compose up -d --force-recreate --build 
            exit
```
2. переменные типа DOCKERHUB_USERNAME берутс из секретов приложения
добавляем их вручную в settings/Secrets and variables/Actions
3. для HOST указываем айпишник машины
4. для USERNAME указываем логин по которым заходим на машину
5. для SSH_KEY указываем SSH key с локальный машины, ключ должен быть непубличным! вместе с -----BEGIN OPENSSH PRIVATE KEY----------END OPENSSH PRIVATE KEY-----
6. для SSH_PORT указываем 22э
7. для PASSPHRASE указываем пароль при входе
8. для DOCKERHUB_TOKEN указываем токен который созаем в docker hub привязав туда публичный SSH ключ
9. заходим на машину и создаем docker-compose.yml по аналогии с ручным деплоем
