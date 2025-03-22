# 1. Настройка WSL и Docker Compose

## 1.1 Установка WSL
```sh
wsl --install -d Ubuntu
```
Запустить Ubuntu и выполнить начальную настройку.

## 1.2 Установка Docker и Docker Compose
Если Docker не установлен, выполнить:
```sh
sudo apt update
sudo apt install docker.io docker-compose -y
sudo systemctl enable --now docker
```

Проверить установку:
```sh
docker --version
docker compose version
```

## 1.3 Настройка Docker Compose
Создать файл docker-compose.yml с настройками для 3 узлов Replica Set:
```sh
nano docker-compose.yml
```
```yaml
services:
  mongo1:
    image: mongo:latest
    container_name: mongo1
    hostname: mongo1
    restart: always
    ports:
      - "27017:27017"
    networks:
      - mongo-cluster
    volumes:
      - mongo1_data:/data/db
    command: ["mongod", "--replSet", "rs0", "--bind_ip_all"]

  mongo2:
    image: mongo:latest
    container_name: mongo2
    hostname: mongo2
    restart: always
    ports:
      - "27018:27017"
    networks:
      - mongo-cluster
    volumes:
      - mongo2_data:/data/db
    command: ["mongod", "--replSet", "rs0", "--bind_ip_all"]

  mongo3:
    image: mongo:latest
    container_name: mongo3
    hostname: mongo3
    restart: always
    ports:
      - "27019:27017"
    networks:
      - mongo-cluster
    volumes:
      - mongo3_data:/data/db
    command: ["mongod", "--replSet", "rs0", "--bind_ip_all"]

networks:
  mongo-cluster:

volumes:
  mongo1_data:
  mongo2_data:
  mongo3_data:
```
Запустить контейнер:
```sh
docker-compose up -d
```
Проверить работу MongoDB:
```sh
docker ps
```
Подключиться к первому узлу:
```sh
docker exec -it mongo1 mongosh
```
Инициализировать кластер:
```sh
rs.initiate({
    _id: "rs0",
    members: [
        { _id: 0, host: "mongo1:27017" },
        { _id: 1, host: "mongo2:27017" },
        { _id: 2, host: "mongo3:27017" }
    ]
})
```
Проверьте статус кластера:
```sh
rs.status()
```
## 1.4 Тестирование отказоустойчивости
Выключить Primary (mongo1):
```sh
docker stop mongo1
```
Проверить новый Primary:
```sh
docker exec -it mongo2 mongosh --eval "rs.status()"
```
Запустить обратно:
```sh
docker start mongo1
```


