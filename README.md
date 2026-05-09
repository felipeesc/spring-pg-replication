# spring-pg-replication

Aplicacao de replicacao de dados baseada em **Change Data Capture (CDC)** utilizando o mecanismo de **Logical Replication do PostgreSQL** para capturar mudancas em tempo real e propaga-las para um **banco de dados PostgreSQL destino**.

## Stack

- **Java 21**
- **Spring Boot**
- **PostgreSQL** -- fonte e destino dos dados via Logical Replication (WAL / pgoutput)
- **Debezium Embedded Engine** -- leitura do WAL dentro da propria aplicacao

## Modelo de consistencia -- PC/EC (PACELC)

O sistema adota o quadrante **PC/EC** do teorema PACELC:

- **Em particao de rede (P):** prioriza **Consistencia (C)** sobre disponibilidade -- o sistema pode ficar indisponivel a fim de evitar dados divergentes.
- **Em operacao normal (E):** prioriza **Consistencia (C)** sobre latencia -- operacoes de replicacao aguardam confirmacao antes de avancar.

Essa escolha e adequada porque:
- Transacoes **ACID** sao requisito -- o PostgreSQL garante isso na fonte.
- Inconsistencia entre fonte e destino nao e tolerada.
- Latencia adicional de coordenacao e aceitavel.

## Arquitetura

```
PostgreSQL fonte (WAL)
    |
    |  replication slot (pgoutput)
    v
Spring Boot
  +-- DebeziumEngine     -- le INSERT/UPDATE/DELETE das tabelas configuradas
  +-- EventProcessor     -- filtra tabelas do subset, mapeia para DTO
  +-- ReplicationWriter  -- aplica os eventos no PostgreSQL destino
    |
    v
PostgreSQL destino
```

### Fluxo de replicacao

1. PostgreSQL fonte grava mudancas no WAL com `wal_level = logical`.
2. O Debezium Embedded Engine consome o slot de replicacao logica.
3. Apenas as tabelas configuradas no subset sao processadas.
4. Cada evento e transformado e aplicado no banco destino dentro de uma transacao.
5. O offset (LSN) e persistido -- em caso de falha, a leitura retoma do ponto exato.

### Garantia de entrega

- **At-least-once** com confirmacao da escrita no destino antes de avancar o offset.
- O banco destino deve tratar reprocessamento de forma idempotente (upsert por PK).
- Falhas na escrita ativam retry com backoff -- o slot de replicacao e retido ate confirmacao.

## Configuracao do PostgreSQL fonte

```sql
-- postgresql.conf
wal_level = logical
max_replication_slots = 5
max_wal_senders = 5
```

```sql
-- permissao para o usuario da aplicacao
ALTER ROLE replication_user REPLICATION LOGIN;
```

## Configuracao da aplicacao

```yaml
# application.yml
debezium:
  connector.class: io.debezium.connector.postgresql.PostgresConnector
  database.hostname: localhost
  database.port: 5432
  database.dbname: source_db
  database.user: replication_user
  database.password: secret
  plugin.name: pgoutput
  slot.name: spring_pg_replication_slot
  table.include.list: public.orders,public.payments  # subset de tabelas
  offset.storage: org.apache.kafka.connect.storage.FileOffsetBackingStore
  offset.storage.file.filename: /data/offsets.dat

replication:
  target:
    url: jdbc:postgresql://localhost:5433/target_db
    username: target_user
    password: secret
```

## Ambiente local com Docker

O ambiente de simulacao utiliza dois containers PostgreSQL isolados na mesma rede Docker, replicando um cenario real de dois bancos independentes.

```
docker network: replication-net
  |
  +-- postgres-source  (porta 5432)
  |     banco: source_db
  |     wal_level = logical  <-- obrigatorio para CDC
  |
  +-- postgres-target  (porta 5433)
        banco: target_db
        banco de destino convencional
```

### Detalhes dos containers

| Container | Porta | Banco | Usuario | Funcao |
|---|---|---|---|---|
| `postgres-source` | 5432 | `source_db` | `replication_user` | Fonte -- WAL habilitado |
| `postgres-target` | 5433 | `target_db` | `target_user` | Destino -- escrita via JDBC |

### Como rodar

```bash
# sobe os dois bancos
docker compose up -d

# valida que o slot de replicacao esta disponivel no fonte
docker exec postgres-source psql -U replication_user -d source_db \
  -c "SELECT * FROM pg_replication_slots;"

# roda a aplicacao
./mvn spring-boot:run
```

### Simulando mudancas

Com os containers rodando, insira dados no banco fonte para acionar a replicacao:

```bash
docker exec postgres-source psql -U replication_user -d source_db -c \
  "INSERT INTO orders (id, description) VALUES (1, 'teste replicacao');"
```

Em seguida verifique se o registro apareceu no destino:

```bash
docker exec postgres-target psql -U target_user -d target_db -c \
  "SELECT * FROM orders;"
```

## Monitoramento critico

| Metrica | O que observar |
|---|---|
| `pg_replication_slots` lag | Slot acumulando = app parada ou destino lento |
| Retry count | Alta taxa indica instabilidade na escrita no destino |
| LSN offset | Deve avancar continuamente |

> **Atencao:** slots de replicacao retidos por muito tempo podem encher o disco do PostgreSQL fonte. Monitore o `confirmed_flush_lsn` do slot.
