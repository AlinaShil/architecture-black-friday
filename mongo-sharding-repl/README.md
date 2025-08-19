# Mongo Sharding с репликацией (по 3 реплики на шард)

В этом проекте каждый шард — это **реплика-сет из 3 узлов** (shard1a/b/c и shard2a/b/c).
Роутер `mongos_router` подключается к конфиг-серверу `configSrv`.

## 1) Поднять контейнеры
```bash
docker compose up -d
```

## 2) Инициализировать реплика-сеты
Сначала — конфиг-сервер (одиночный RS), затем оба шарда (по 3 члена в каждом).

```bash
# --- config server (RS из 1 узла) ---
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

## 5) (Опционально) Наполнить тестовыми данными (≥1000)
```bash
docker compose exec -T mongos_router mongosh --port 27020 --quiet <<'EOF'
use somedb
const N = 1000, B = 200, buf = [];
for (let i = 0; i < N; i++) {
  buf.push({ name: "user_"+i, n: i, createdAt: new Date() });
  if (buf.length === B) { db.helloDoc.insertMany(buf); buf.length = 0; }
}
if (buf.length) db.helloDoc.insertMany(buf);
print("total:", db.helloDoc.countDocuments());
EOF
```

## 6) Проверка распределения и состояния
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

---

### Примечания
- Все члены одного шарда слушают **один и тот же порт** (27018 для shard1, 27019 для shard2). Это нормально: внутри сети Compose важны **имена сервисов**, а не порты хоста.
- Если `mongos_router` стартовал раньше, чем `config_server` получил PRIMARY, просто выполни шаги инициализации — `mongos` станет работоспособным автоматически.
- При желании можно добавить healthchecks/задержки старта, но для учебной среды так проще.
