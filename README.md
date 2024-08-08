# Introduction

Setup for Kafka with debezium and two PostgreSQL databases to use for CDC. This setup creates two databases with named volumes, you would need to apply some commands after all the containers are up.

## Docker

```
docker-compose up -d
```


## Databases

- The target DB is exposed via port `5333`
- The source DB is exposed via port `5433`

### Target DB

Kafka publication:
```sql
CREATE PUBLICATION kafka_test FOR ALL tables;
```

A dummy table:
```
CREATE TABLE public.auth_user (
	id serial4 NOT NULL,
	"password" varchar(128) NOT NULL,
	username varchar(150) NOT NULL,
	first_name varchar(150) NOT NULL,
	last_name varchar(150) NOT NULL,
	created_at timestamp NULL,
	updated_at timestamp NULL,
	CONSTRAINT auth_user_pkey PRIMARY KEY (id),
	CONSTRAINT auth_user_username_key UNIQUE (username)
);

CREATE INDEX auth_user_username_6821ab7c_like ON public.auth_user USING btree (username varchar_pattern_ops);
```

### source DB

A dummy table:
```
CREATE TABLE public.auth_user (
	id serial4 NOT NULL,
	"password" varchar(128) NOT NULL,
	username varchar(150) NOT NULL,
	first_name varchar(150) NOT NULL,
	last_name varchar(150) NOT NULL,
	created_at timestamp NULL,
	updated_at timestamp NULL,
	CONSTRAINT auth_user_pkey PRIMARY KEY (id),
	CONSTRAINT auth_user_username_key UNIQUE (username)
);
```

## Connectors

Make sure that the source and target connectors data are correct, if you used the above setup for the DBs then you would only need to change the machine IP.

You can get it via:
```
ifconfig | grep "inet " | grep -v 127.0.0.1
```

Once you have the IP: 
- Change it in `connector-source-postgres.json` file for the key `database.hostname`
- Change it in `onnector-target-postgres.json` file for the key `connection.url`


Now you can add the two connectors for Kafka with Curl:

```
curl -i -X POST -H "Accept:application/json" -H  "Content-Type:application/json" http://localhost:8083/connectors/ -d @connector-source-postgres.json
```

```curl
curl -i -X POST -H "Accept:application/json" -H  "Content-Type:application/json" http://localhost:8083/connectors/ -d @connector-target-postgres.json
```

You can navigate to `http://localhost:8080/` to verify from the Kafka UI.


## Tests
Tail the logs on the Kafka container to see the topics incoming:
```
docker-compose -f docker-compose.yml exec kafka1 kafka-console-consumer \
--bootstrap-server kafka1:9092 \
--from-beginning \
--property print.key=true \
--topic "auth.public.auth_user"
```


Run an insertion query on the target DB:
```sql
INSERT INTO public.auth_user (
	"password",
	username,
	first_name,
	last_name,
	created_at,
	updated_at
) VALUES
	('asdfad', 'test', 'foo', 'bar', NOW(), NOW()),
	('asdfad', 'test_2', 'foo_2', 'bar_2', NOW(), NOW()),
	('asdfad', 'test_3', 'foo_3', 'bar_3', NOW(), NOW()),
	('asdfad', 'test_4', 'foo_4', 'bar_4', NOW(), NOW());
```

Now check if the source DB has the same data!
