# MongoDB Sharding (2 shards) + API

## 1. Поднять контейнеры

```bash
docker compose up -d
```

---

## 2. Инициализировать реплика-сеты (config + shard1 + shard2)

```bash
# config server
docker compose exec -T configSrv mongosh --port 27017 --quiet <<'EOF'
rs.initiate({
  _id: "config_server",
  configsvr: true,
  members: [{ _id: 0, host: "configSrv:27017" }]
})
EOF

# shard1
docker compose exec -T shard1 mongosh --port 27018 --quiet <<'EOF'
rs.initiate({
  _id: "shard1",
  members: [{ _id: 0, host: "shard1:27018" }]
})
EOF

# shard2
docker compose exec -T shard2 mongosh --port 27019 --quiet <<'EOF'
rs.initiate({
  _id: "shard2",
  members: [{ _id: 0, host: "shard2:27019" }]
})
EOF
```

> Необходимо подождать 5–10 секунд, чтобы узлы стали `PRIMARY`.

---

## 3. Подключить шарды к роутеру и включить шардинг

```bash
# добавить два шарда
docker compose exec -T mongos_router mongosh --port 27020 --quiet <<'EOF'
sh.addShard("shard1/shard1:27018")
sh.addShard("shard2/shard2:27019")
sh.status()
EOF

# включить шардинг для БД somedb и зашардировать коллекцию helloDoc по _id (hashed)
docker compose exec -T mongos_router mongosh --port 27020 --quiet <<'EOF'
sh.enableSharding("somedb")
sh.shardCollection("somedb.helloDoc", { _id: "hashed" })
EOF
```

---

## 4. Проверить вставку и распределение данных

```bash
# вставим документы через роутер
docker compose exec -T mongos_router mongosh --port 27020 --quiet <<EOF
use somedb
for(var i = 0; i < 1000; i++) db.helloDoc.insertOne({age:i, name:"ly"+i})
EOF

# проверить содержимое шарда 1
docker compose exec -T shard1 mongosh --port 27018 --quiet <<'EOF'
use somedb
db.helloDoc.countDocuments()
EOF

# проверить содержимое шарда 2
docker compose exec -T shard2 mongosh --port 27019 --quiet <<'EOF'
use somedb
db.helloDoc.countDocuments()
EOF

# проверить общее количество файлов через роутер
docker compose exec -T mongos_router mongosh --port 27019 --quiet <<'EOF'
use somedb
db.helloDoc.countDocuments()
EOF
```

---

## 5. Проверка через API

- Открыть главную страницу приложения:  
  http://localhost:8080/  
  В ответе должно быть поле `mongo_topology_type: "Sharded"` и два шарда в списке.

- Проверить количество документов:
  ```
  GET http://localhost:8080/helloDoc/count
  ```

- Вставить документ через API:
  ```bash
  curl -X POST http://localhost:8080/helloDoc/users         -H "Content-Type: application/json"         -d '{"name":"Dave","age":42}'
  ```
