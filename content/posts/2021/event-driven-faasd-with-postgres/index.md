---
title: "Event driven functions powered by Postgres"
date: 2021-01-30T00:00:00+02:00
tags:
  - programming
  - openfaas
  - serverless
  - postgres
  - wal
  - event driven
draft: false
coverImage: images/cover1.jpg
---

An event-driven architecture can let you seamlessly extend your application or improve the scalability, if you can handle the eventual consistency. But your app may not be ready for this yet, or you don't own the code in the app. A recently added a feature to [`faasd`](https://github.com/openfaas/faasd) got me thinking about event driven architecture powered by the Postgres WAL. Which means we can seamlessly extend your app without needing to change the app! 

This post will show you how to quickly deploy Postgresql along with an event listener and some custom functions. From there it’s up to you what you build.

<!--more-->

The core components of [`faasd`](https://github.com/openfaas/faasd) are defined and deployed via a [Compose spec](https://www.compose-spec.io/). Yes that Compose made famous by Docker Compose. This is great because it allows us to add our own custom sidecar containers to deploy along side `faasd` and more importantly along side and exposed to our OpenFaaS functions.

> faasd is [OpenFaaS](https://github.com/openfaas/) reimagined, but without the cost and complexity of Kubernetes. It runs on a single host with very modest requirements, making it fast and easy to manage.

This is super cool, but not the feature that got me thinking.  It may not sound like much, but [`faasd` 0.10.0+](https://github.com/openfaas/faasd/releases/tag/0.10.0) now has support to set the container user! This is important because if I want to run a database next to my functions and use a local folder to provide durable persistence, I need to set the container user for Postgres.

Now that I can run Postgres and functions together, I started thinking "I should have easy access to the WAL". Cue evil laughter.

I started looking at [`wal2json`](https://github.com/eulerto/wal2json) and but ultimately ended up looking at [`wal-listener`](https://github.com/ihippik/wal-listener) because I looked at the OpenFaaS ecosystem and we already have event driven functions via the [`nats-connector`](https://github.com/openfaas/nats-connector) and `faasd` _already_ installs NATS. With `wal-listenr` plus `Postgres just work directly on top of that.

## Get started

If you have `faasd` up and running, then you can skip this part, but if not, then I recommend you grab [`multipass`](https://multipass.run/), start a new VM and run this

```sh
git clone https://github.com/openfaas/faasd
cd faasd

./hack/install.sh
```

You can also get started with cloud-config, [check out this tutorial](https://github.com/openfaas/faasd#deploy-faasd)

## Add postgres to the mix

I created a small proof-of-concept repo that adds postgres, wal-listener, and a simple "receiver" function that just prints all of the events it receives. It doesn't do anything, but you can simply replace the function with any logic you want, first thing that comes to my mind _webhooks_ Parse the event and forward it to another function or service.


1. clone the example repo
    ```sh
    git clone https://github.com/LucasRoesler/openfaas-wal-listener.git
    cd openfaas-wal-listener
    ```
2. Copy the new docker-compose
   ```sh
   make install 
   # or manually run 
   # sudo cp -rf postgres /var/lib/faasd
   # sudo mkdir -p /var/lib/faasd/postgres/pgdata
   # sudo mkdir -p /var/lib/faasd/postgresql/run
   # sudo cp -rf wal_listener /var/lib/faasd
   # sudo cp docker-compose.yaml /var/lib/faasd
   # sudo chown -R 1000:1000 /var/lib/faasd/postgres
   ```
3. Restart faasd
   ```sh
   make restart
   # or manually: sudo systemctl restart faasd faasd-provider
   ```

4. Deploy a function to receive database events and print them out

   ```sh
   faas-cli deploy --name receive-event \
      --image theaxer/receive-message:latest \
      --fprocess='./handler' \
      --annotation topic="sample_app"
   ```

You can then verify that the connector is sending events to tne receiver fucntion by manually publishing the topic (you need to install the [natscli](https://github.com/nats-io/natscli))

```sh
$ nats pub sample_app "manual push"
15:21:39 Published 11 bytes to "sample_app"
$ faas-cli logs receive-event
2021-01-24T14:21:17Z 2021/01/24 14:21:17 Started logging stderr from function.
2021-01-24T14:21:17Z 2021/01/24 14:21:17 Started logging stdout from function.
2021-01-24T14:21:17Z 2021/01/24 14:21:17 OperationalMode: http
2021-01-24T14:21:17Z 2021/01/24 14:21:17 Timeouts: read: 10s, write: 10s hard: 10s.
2021-01-24T14:21:17Z 2021/01/24 14:21:17 Listening on port: 8080
2021-01-24T14:21:17Z 2021/01/24 14:21:17 Writing lock-file to: /tmp/.lock
2021-01-24T14:21:17Z 2021/01/24 14:21:17 Metrics listening on port: 8081
2021-01-24T14:21:17Z Forking - ./handler []
2021-01-24T14:21:39Z 2021/01/24 14:21:39 POST / - 200 OK - ContentLength: 27
2021-01-24T14:21:39Z 2021/01/24 14:21:39 stderr: 2021/01/24 14:21:39 received "manual push"
```


## Verify db events are sent
The example repo contains a small SQL script that will create a `User` table and upsert several users, the events from `wal-listener` will look like 

```json
{
    "id": "cd03a0b0-8c98-4809-9347-9f469c773de0",
    "schema": "public",
    "table": "users",
    "action": "UPDATE",
    "data": {
        "created_at": "2021-01-23 16:50:53.543372+00",
        "updated_at": "2021-01-23 16:50:53.543372+00",
        "name": "Sarah Walker",
        "email": "walker@nerdherd.com",
        "id": "3879dc32-78cd-456d-8a64-fe7fab540a7f"
    },
    "commitTime": "2021-01-24T15:05:00.007715Z"
}
```

Let's see it in action:

1. Generate some changes in the db

    ```sql
    CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

    CREATE TABLE IF NOT EXISTS "users" (
        id uuid PRIMARY KEY DEFAULT uuid_generate_v4(),
        created_at timestamptz NOT NULL DEFAULT NOW(),
        updated_at timestamptz NOT NULL DEFAULT NOW(),
        name text,
        email text UNIQUE
    );

    CREATE UNIQUE INDEX IF NOT EXISTS users_email_idx ON users (email);

    INSERT INTO users (name, email)
        VALUES 
            ('John Wick', 'wick@trouble.com'),
            ('James Bond', 'bond@mi6.com'),
            ('Elim Garak', 'garak@ds9.com'),
            ('Sarah Walker', 'walker@nerdherd.com')
        ON CONFLICT (email) DO 
        UPDATE SET name = EXCLUDED.name
        RETURNING *;
    ```
    This script is safe to run several time, on the repeated runs it will generate UPDATE events for the function.

    You can run it with the Makefile (or manually), the db password is `supersecret`

    ```sh
    make init-db
    # psql -U postgres -h localhost -d app -f postgres/init_app.sql
    ```

2. You can then see the logs as wal-listener reacts to the inserts by using 

   ```sh
   sudo journalctl -f -t openfaas:wal-listener --since="5 minutes ago"
   ```

   These are cool, but magic is when we look at the function logs.

3. Finally, see that our receiver function was invoked

   ```sh
   $ faas-cli logs receive-event
   POST / - 200 OK - ContentLength: 386
   stderr: 2021/01/24 15:04:25 received "{\"id\":\"92c431a7-9027-4656-9d39-f52ee90d5dd6\",\"schema\":\"public\",\"table\":\"users\",\"action\":\"UPDATE\",\"data\":{\"created_at\":\"2021-01-23 16:50:53.543372+00\",\"updated_at\":\"2021-01-23 16:50:53.543372+00\",\"name\":\"John Wick\",\"email\":\"wick@trouble.com\",\"id\":\"99a9c7bf-8f31-4c30-998e-9740f87bdaa0\"},\"commitTime\":\"2021-01-24T15:04:25.58604Z\"}"
   POST / - 200 OK - ContentLength: 383
   stderr: 2021/01/24 15:04:25 received "{\"id\":\"80a8d33a-0ae5-4aed-825d-1a7f9e24adfb\",\"schema\":\"public\",\"table\":\"users\",\"action\":\"UPDATE\",\"data\":{\"name\":\"James Bond\",\"email\":\"bond@mi6.com\",\"id\":\"5a6a9672-722a-40c9-8f10-17cf527a6b41\",\"created_at\":\"2021-01-23 16:50:53.543372+00\",\"updated_at\":\"2021-01-23 16:50:53.543372+00\"},\"commitTime\":\"2021-01-24T15:04:25.58604Z\"}"
   POST / - 200 OK - ContentLength: 384
   stderr: 2021/01/24 15:04:25 received "{\"id\":\"326ea179-dddb-4c04-bd56-9f02bdec9aaa\",\"schema\":\"public\",\"table\":\"users\",\"action\":\"UPDATE\",\"data\":{\"created_at\":\"2021-01-23 16:50:53.543372+00\",\"updated_at\":\"2021-01-23 16:50:53.543372+00\",\"name\":\"Elim Garak\",\"email\":\"garak@ds9.com\",\"id\":\"b1f3cc48-6366-4bff-b0c5-3f5beed02f44\"},\"commitTime\":\"2021-01-24T15:04:25.58604Z\"}"
   POST / - 200 OK - ContentLength: 392
   stderr: 2021/01/24 15:04:25 received "{\"id\":\"15755624-195a-4617-8d60-6b7cb748caf9\",\"schema\":\"public\",\"table\":\"users\",\"action\":\"UPDATE\",\"data\":{\"id\":\"3879dc32-78cd-456d-8a64-fe7fab540a7f\",\"created_at\":\"2021-01-23 16:50:53.543372+00\",\"updated_at\":\"2021-01-23 16:50:53.543372+00\",\"name\":\"Sarah Walker\",\"email\":\"walker@nerdherd.com\"},\"commitTime\":\"2021-01-24T15:04:25.58604Z\"}"
   POST / - 200 OK - ContentLength: 387
   stderr: 2021/01/24 15:05:00 received "{\"id\":\"38f4ae55-86a1-466a-8ebd-c2f5a1054439\",\"schema\":\"public\",\"table\":\"users\",\"action\":\"UPDATE\",\"data\":{\"updated_at\":\"2021-01-23 16:50:53.543372+00\",\"name\":\"John Wick\",\"email\":\"wick@trouble.com\",\"id\":\"99a9c7bf-8f31-4c30-998e-9740f87bdaa0\",\"created_at\":\"2021-01-23 16:50:53.543372+00\"},\"commitTime\":\"2021-01-24T15:05:00.007715Z\"}"
   stderr: 2021/01/24 15:05:00 received "{\"id\":\"bce01097-815d-4d73-90b9-89230ddca39b\",\"schema\":\"public\",\"table\":\"users\",\"action\":\"UPDATE\",\"data\":{\"email\":\"bond@mi6.com\",\"id\":\"5a6a9672-722a-40c9-8f10-17cf527a6b41\",\"created_at\":\"2021-01-23 16:50:53.543372+00\",\"updated_at\":\"2021-01-23 16:50:53.543372+00\",\"name\":\"James Bond\"},\"commitTime\":\"2021-01-24T15:05:00.007715Z\"}"
   POST / - 200 OK - ContentLength: 384
   POST / - 200 OK - ContentLength: 385
   stderr: 2021/01/24 15:05:00 received "{\"id\":\"45f0fc1f-a44f-4546-a7dd-2fec7079667d\",\"schema\":\"public\",\"table\":\"users\",\"action\":\"UPDATE\",\"data\":{\"name\":\"Elim Garak\",\"email\":\"garak@ds9.com\",\"id\":\"b1f3cc48-6366-4bff-b0c5-3f5beed02f44\",\"created_at\":\"2021-01-23 16:50:53.543372+00\",\"updated_at\":\"2021-01-23 16:50:53.543372+00\"},\"commitTime\":\"2021-01-24T15:05:00.007715Z\"}"
   POST / - 200 OK - ContentLength: 393
   stderr: 2021/01/24 15:05:00 received "{\"id\":\"cd03a0b0-8c98-4809-9347-9f469c773de0\",\"schema\":\"public\",\"table\":\"users\",\"action\":\"UPDATE\",\"data\":{\"created_at\":\"2021-01-23 16:50:53.543372+00\",\"updated_at\":\"2021-01-23 16:50:53.543372+00\",\"name\":\"Sarah Walker\",\"email\":\"walker@nerdherd.com\",\"id\":\"3879dc32-78cd-456d-8a64-fe7fab540a7f\"},\"commitTime\":\"2021-01-24T15:05:00.007715Z\"}"
   ```

   I trimmed log timestamps for legibility.

## Next steps

One tiny service and we have integrated Postgres into OpenFaaS. But, this example is just the tip of the iceberg, the example doesn't _do_ anything. If you have a postgres application, you could easily deploy `wal-listener` for your database, it will work with your local self-hosted Postgres _and_ cloud hosted Postgres like RDS.  You could send events from your application to an an automation system like Zapier or [`n8n`](https://n8n.io/)

If you don't have Postgres, let's say you are a MySQL fan or a NoSQL fan using MongoDB, don't worry I won't hold it against you. Also, you can mimic the same workflow by just swapping out `wal-listener`. The [lapidus](https://github.com/JarvusInnovations/lapidus) project supports both MySQL _and_ MongoDB, but I am not a MySQL or Mongo expert, so YMMV.  

Another connector library was also recommended to me, [debezium](https://debezium.io/). It has several [connectors](https://debezium.io/documentation/reference/connectors/index.html) including Postgre, MySQL, MongoDB, Oracle, and Cassandra.

You can also find a walk-through of how to deploy and customize `faasd` in Alex's new book [Serverless for Everyone Else](https://gumroad.com/l/serverless-for-everyone-else).

If you build something awesome, let me know on [Twitter](https://twitter.com/theaxer) or stop by the [OpenFaas Slack](https://docs.openfaas.com/community/#slack-workspace) and share it with the community.