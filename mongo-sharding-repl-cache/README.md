# Sharding + Replication + Redis Cache

В этом проекте каждый шард — это **реплика-сет из 3 узлов** (shard1a/b/c и shard2a/b/c).
Роутер `mongos_router` подключается к конфиг-серверу `configSrv`. Добавляется инстанс **Redis**, чтобы кешировать ответы приложения.

## 1) Поднять сервисы
```bash
docker compose up -d
```

## 2) Инициализировать реплика-сеты
```bash
# --- config server ---
docker compose exec -T configSrv mongosh --port 27017 --quiet <<'EOF'
rs.initiate({
  _id: "config_server",
  configsvr: true,
  members: [{ _id: 0, host: "configSrv:27017" }]
})
EOF

# --- shard1 (3 узла) ---
docker compose exec -T shard1a mongosh --port 27018 --quiet <<'EOF'
rs.initiate({
  _id: "shard1",
  members: [
    { _id: 0, host: "shard1a:27018", priority: 2 },
    { _id: 1, host: "shard1b:27018", priority: 1 },
    { _id: 2, host: "shard1c:27018", priority: 1 }
  ]
})
EOF

# --- shard2 (3 узла) ---
docker compose exec -T shard2a mongosh --port 27019 --quiet <<'EOF'
rs.initiate({
  _id: "shard2",
  members: [
    { _id: 0, host: "shard2a:27019", priority: 2 },
    { _id: 1, host: "shard2b:27019", priority: 1 },
    { _id: 2, host: "shard2c:27019", priority: 1 }
  ]
})
EOF
```

## 3) Проверить, что появились PRIMARY
```bash
# 1 → PRIMARY, 2 → SECONDARY, 0 → STARTUP
docker compose exec -T configSrv mongosh --quiet --port 27017 --eval "rs.status().myState"
docker compose exec -T shard1a  mongosh --quiet --port 27018 --eval "rs.status().myState"
docker compose exec -T shard2a  mongosh --quiet --port 27019 --eval "rs.status().myState"
```

## 4) Подключить шарды к роутеру, включить шардинг и зашардировать коллекцию
Выполнять команды на `mongos_router`:
```bash
# добавить оба шарда (реплика-сеты)
docker compose exec -T mongos_router mongosh --port 27020 --quiet <<'EOF'
sh.addShard("shard1/shard1a:27018,shard1b:27018,shard1c:27018")
sh.addShard("shard2/shard2a:27019,shard2b:27019,shard2c:27019")
sh.status()
EOF

# включить шардинг и зашардировать коллекцию по _id (hashed)
docker compose exec -T mongos_router mongosh --port 27020 --quiet <<'EOF'
sh.enableSharding("somedb")
sh.shardCollection("somedb.helloDoc", { _id: "hashed" })
EOF
```

## 5) Наполнить тестовыми данными (≥1000)
```bash
docker compose exec -T mongos_router mongosh --port 27020 --quiet <<EOF
use somedb
for(var i = 0; i < 1000; i++) db.helloDoc.insertOne({age:i, name:"ly"+i})
EOF
```

## 6) Проверить распределение и состояние
```bash
# суммарное количество
docker compose exec -T mongos_router mongosh --port 27020 --quiet <<'EOF'
use somedb
db.helloDoc.countDocuments()
db.helloDoc.getShardDistribution()
EOF

# состояние RS (кратко)
docker compose exec -T shard1a mongosh --quiet --port 27018 --eval "rs.status().members.map(m=>m.stateStr)"
docker compose exec -T shard2a mongosh --quiet --port 27019 --eval "rs.status().members.map(m=>m.stateStr)"
```

## 7) Включить кеширование приложения
В `compose.yaml` уже добавлен сервис `redis` и у `pymongo_api` проставлена переменная:
```
REDIS_URL=redis://redis:6379
```
Этого достаточно, чтобы приложение включило кеширование для эндпоинта:
```
GET /<collection_name>/users
# например:
GET /helloDoc/users
```

## 8) Проверить, что кеш работает и ускоряет повторные запросы
```bash
# первый запрос (cache miss)
curl -s -w "\nTime: %{time_total}s\n" http://localhost:8080/helloDoc/users > /dev/null

# второй запрос (cache hit)
curl -s -w "\nTime: %{time_total}s\n" http://localhost:8080/helloDoc/users > /dev/null
```
У второго запроса значение `time_total` меньше - значит, он быстрее.
```

---

