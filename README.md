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

# 2. Импорт базы данных и запросов на выборку и обновление данных

## 2.1 Импорт данных
Cкопировать файл в контейнер:
```sh
docker cp ~/MongoDB-RS/airbnb_data.json mongo1:/airbnb_data.json
```
Выполните команду, чтобы зайти в контейнер:
```sh
docker exec -it mongo1 bash
```
Проверить, есть ли файл:
```sh
ls -l /airbnb_data.json
```
Использовать mongoimport, чтобы загрузить JSON в базу:
```sh
docker exec -it mongo1 mongoimport --db airbnb --collection booking --file /airbnb_data.json
```

## 2.2 Запросы на выборку и обновление данных
Выполнить, чтобы использовать нужную БД
```sh
show dbs
use airbnb
```
Вывести все записи:
```sh
db.booking.find({})
```
Вывести 5 запсией:
```sh
db.booking.find({}).limit(5)
```
Вывести записи, у которых значение цены (price) меньше 500:
```sh
db.booking.find({ price: { $lte: 500 } })
```
Вывести записи, у которых отстутсвуют отзывы (rewies):
```sh
db.booking.find({ $or: [{ reviews: { $exists: false } }, { reviews: { $size: 0 } }] })
```
Вывести запсии, у которых страна Испания (Spain):
```sh
db.booking.find({'host.host_location': 'Spain'})
```
Вывести записи, у которых страна Испания (Spain) и рейтинг отзывов больше 90 (review_scores_rating):
```sh
db.booking.find({$and: [{'host.host_location': 'Spain'}, {'review_scores.review_scores_rating': {$gte:90}}]})
```
Сделать вставку в БД:
```sh
db.booking.insertOne({
    _id: "99999999",
    listing_url: "https://www.airbnb.com/rooms/99999999",
    name: "Центр Москвы: уютные апартаменты",
    summary: "Уютные апартаменты в центре Москвы, рядом с Кремлем. Отличный вариант для туристов!",
    space: "Просторная квартира с современным ремонтом и всеми удобствами.",
    description: "Апартаменты расположены в центре Москвы, рядом с Красной площадью. В шаговой доступности рестораны, магазины и достопримечательности.",
    neighborhood_overview: "Рядом с домом находятся исторические места, лучшие кафе и бары Москвы.",
    transit: "До станции метро 'Охотный ряд' — 5 минут пешком.",
    access: "Гости могут пользоваться всей квартирой.",
    interaction: "Я всегда на связи, готов помочь с любыми вопросами.",
    property_type: "Apartment",
    room_type: "Entire home/apt",
    bed_type: "Real Bed",
    minimum_nights: "2",
    maximum_nights: "30",
    cancellation_policy: "moderate",
    last_scraped: ISODate("2025-03-22T05:00:00.000Z"),
    accommodates: 4,
    bedrooms: 1,
    beds: 2,
    number_of_reviews: 10,
    bathrooms: Decimal128("1.0"),
    amenities: [
        "Wi-Fi",
        "Kitchen",
        "Washer",
        "TV",
        "Heating",
        "Air conditioning",
        "Elevator",
        "Essentials"
    ],
    price: Decimal128("5000.00"),
    extra_people: Decimal128("500.00"),
    guests_included: Decimal128("2"),
    images: {
        thumbnail_url: "",
        medium_url: "",
        picture_url: "https://example.com/moscow_apartment.jpg",
        xl_picture_url: ""
    },
    host: {
        host_id: "123456789",
        host_url: "https://www.airbnb.com/users/show/123456789",
        host_name: "Иван",
        host_location: "Москва, Россия",
        host_response_time: "within an hour",
        host_thumbnail_url: "https://example.com/host.jpg",
        host_picture_url: "https://example.com/host_large.jpg",
        host_response_rate: 95,
        host_is_superhost: true,
        host_has_profile_pic: true,
        host_identity_verified: true,
        host_listings_count: 5
    },
    address: {
        street: "Москва, Россия",
        suburb: "Центр",
        government_area: "Тверской",
        market: "Москва",
        country: "Russia",
        country_code: "RU",
        location: {
            type: "Point",
            coordinates: [ 37.6173, 55.7558 ],
            is_location_exact: true
        }
    },
    availability: {
        availability_30: 20,
        availability_60: 45,
        availability_90: 80,
        availability_365: 300
    },
    review_scores: {
        review_scores_accuracy: 9,
        review_scores_cleanliness: 10,
        review_scores_checkin: 10,
        review_scores_communication: 10,
        review_scores_location: 10,
        review_scores_value: 9,
        review_scores_rating: 98
    },
    reviews: [
        {
            _id: "567890",
            date: ISODate("2025-03-10T05:00:00.000Z"),
            reviewer_id: "987654",
            reviewer_name: "Анна",
            comments: "Отличное место! Очень уютно и удобно, рядом с метро."
        }
    ]
});
```



