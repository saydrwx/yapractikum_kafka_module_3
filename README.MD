# Задание по оптимизации параметров JDBC Source коннектора
- Команда для удаления коннектора `curl -X DELETE http://localhost:8083/connectors/postgres-source`
- Результаты
  - Сообщения весят 126 байт, `batch.max.rows` равен 100, 126*100=12600. Дефолтное значение достаточно близко, поэтому для `batch.size` можно вернуть его
  - В целом, я бы оставил параметры из 4 прогона

| `batch.size` | `linger.ms` | `buffer.memory` | `compression.type` | write rate (kops/sec) |
| ------------ | ----------- | --------------- | ------------------ | --------------------- |
| 500          | 1000        | 33554432        | none               | 49.0                  |
| 16384        | 1000        | 33554432        | none               | 152                   |
| 16384        | 1000        | 33554432        | zstd               | 150                   |
| 16384        | 1000        | 33554432        | snappy             | 152                   |
| 16384        | 1000        | 67108864        | snappy             | 152                   |
| 16384        | 2000        | 67108864        | snappy             | 152                   |

# Задание по коннетору Debezium
- Запускаем docker compose
```bash
docker compose -f infra_template/docker-compose.yaml up -d
```
- Создаем таблицы в БД
```bash
docker exec -it postgres psql -h 127.0.0.1 -U postgres-user -d customers

CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    email VARCHAR(100),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    user_id INT REFERENCES users(id),
    product_name VARCHAR(100),
    quantity INT,
    order_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```
- Запускаем коннектор Debezium
```bash
curl -X PUT \
-H "Content-Type: application/json" \
--data '{
"connector.class": "io.debezium.connector.postgresql.PostgresConnector",
"database.hostname": "postgres",
"database.port": "5432",
"database.user": "postgres-user",
"database.password": "postgres-pw",
"database.dbname": "customers",
"database.server.name": "customers",
"table.whitelist": "public.users,public.orders",
"transforms": "unwrap",
"transforms.unwrap.type": "io.debezium.transforms.ExtractNewRecordState",
"transforms.unwrap.drop.tombstones": "false",
"transforms.unwrap.delete.handling.mode": "rewrite",
"topic.prefix": "customers",
"topic.creation.enable": "true",
"topic.creation.default.replication.factor": "-1",
"topic.creation.default.partitions": "-1",
"skipped.operations": "none"
}' \
http://localhost:8083/connectors/pg-connector/config
```
- Проверка статуса коннектора
```bash
curl -s http://localhost:8083/connectors/pg-connector/status | jq .
```
- Добавляем данные в таблицы
```bash
docker exec -it postgres psql -h 127.0.0.1 -U postgres-user -d customers

-- Добавление пользователей
INSERT INTO users (name, email) 
VALUES ('John Doe', 'john@example.com'),
       ('Jane Smith', 'jane@example.com'),
       ('Alice Johnson', 'alice@example.com'),
       ('Bob Brown', 'bob@example.com');

-- Добавление заказов
INSERT INTO orders (user_id, product_name, quantity) 
VALUES (1, 'Product A', 2),
       (1, 'Product B', 1),
       (2, 'Product C', 5),
       (3, 'Product D', 3),
       (4, 'Product E', 4);
```
- Проверка работоспособности
  - Вычитываем топики
  ```bash
  docker exec -it kafka-0 /bin/bash
  /bin/kafka-console-consumer --bootstrap-server localhost:9092 --topic customers.public.users --from-beginning
  /bin/kafka-console-consumer --bootstrap-server localhost:9092 --topic customers.public.orders --from-beginning
  ```
  - Проверяем топики в Kafka UI http://localhost:8085/ui/clusters/zookeeper-cluster/all-topics?perPage=25
  - Проверяем дэшборд в Grafana http://localhost:3000/d/kafka-connect-overview-0/kafka-connect-overview-0?orgId=1&from=now-30m&to=now