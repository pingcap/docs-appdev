---
title: App Development for Prisma
summary: Learn how to build a simple Prisma Migrate using TiDB and Prisma.
---

# App Development for Elixir

This tutorial shows you how to build a simple Prisma Migrate based on TiDB and Prisma. 

## Step 1. Start a TiDB cluster

Start a pseudo TiDB cluster on your local storage:

{{< copyable "" >}}

```bash
docker run pingcap/tidb:v5.1.0 -p 127.0.0.1:$LOCAL_PORT:4000
```

The above command starts a temporary and single-node cluster with mock TiKV. The cluster listens on the port `$LOCAL_PORT`. After the cluster is stopped, any changes already made to the database are not persisted.

> **Note:**
>
> To deploy a "real" TiDB cluster for production, see the following guides:
>
> + [Deploy TiDB using TiUP for On-Premises](https://docs.pingcap.com/tidb/v5.1/production-deployment-using-tiup)
> + [Deploy TiDB on Kubernetes](https://docs.pingcap.com/tidb-in-kubernetes/stable)
>
> You can also [use TiDB Cloud](https://pingcap.com/products/tidbcloud/), a fully-managed Database-as-a-Service (DBaaS), which offers free trial.

## Step 2. Create a database

1. Create a SQL user for your application:

    {{< copyable "sql" >}}

    ```sql
    CREATE USER <username> WITH PASSWORD <password>;
    ```

    Take note of the username and password. You will use them in your application code when initializing the project.

2. Set `tidb_enable_noop_functions=1` to support Prisma Migrate:

    {{< copyable "sql" >}}

    ```sql
    SET GLOBAL tidb_enable_noop_functions=1;
    ```

## Step 3. Create a Prisma project

1. Creates a new Elixir project called `hello-prisma`.

    {{< copyable "" >}}

    ```bash
    $ mkdir hello-prisma
    $ cd hello-prisma
    ```

2. Initialize a TypeScript project and add the Prisma CLI

    {{< copyable "" >}}

    ```bash
    $ npm init -y
    $ npm install prisma typescript ts-node @types/node --save-dev
    ```

3. Create a `tsconfig.json` file

    {{< copyable "" >}}

    ```shell
    {
      "compilerOptions": {
        "sourceMap": true,
        "outDir": "dist",
        "strict": true,
        "lib": ["esnext"],
        "esModuleInterop": true
      }
    }
    ```

4. Execute Prisma CLI and create your Prisma schema file

    {{< copyable "" >}}

    ```shell
    $ npx prisma
    $ npx prisma init
    ```

## Step 4. Connect TiDB

Now the application configurations are ready, we can run Prisma Migration.

1. Edit `prisma/schema.prisma` file and change the provide to "mysql" and update the ".env" file.

    {{< copyable "" >}}

    ```shell
    datasource db {
      provider = "mysql"
      url      = env("DATABASE_URL")
    }
    ```

    {{< copyable "" >}}

    ```sql
    DATABASE_URL="mysql://root@localhost:4000/test"
    ```

2. Edit `prisma/schema.prisma` to create the database schema

    {{< copyable "" >}}

    ```
    model Post {
      id        Int      @id @default(autoincrement())
      createdAt DateTime @default(now())
      updatedAt DateTime @updatedAt
      title     String   @db.VarChar(255)
      content   String?
      published Boolean  @default(false)
      author    User     @relation(fields: [authorId], references: [id])
      authorId  Int
    }

    model Profile {
      id     Int     @id @default(autoincrement())
      bio    String?
      user   User    @relation(fields: [userId], references: [id])
      userId Int     @unique
    }

    model User {
      id      Int      @id @default(autoincrement())
      email   String   @unique
      name    String?
      posts   Post[]
      profile Profile?
    }
    ```

3. Run `npx prisma migrate dev --name init` to perform migration.

    {{< copyable "" >}}

    ```
    $ npx prisma migrate dev --name init
    ```

    {{< copyable "sql" >}}

    ```sql
    mysql> use test
    mysql> show tables;
    +--------------------+
    | Tables_in_test     |
    +--------------------+
    | Post               |
    | Profile            |
    | User               |
    | _prisma_migrations |
    +--------------------+
    4 rows in set (0.00 sec)

    mysql> show create table Post\G
    *************************** 1. row ***************************
          Table: Post
    Create Table: CREATE TABLE `Post` (
      `id` int(11) NOT NULL AUTO_INCREMENT,
      `createdAt` datetime(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
      `updatedAt` datetime(3) NOT NULL,
      `title` varchar(255) COLLATE utf8mb4_unicode_ci NOT NULL,
      `content` varchar(191) COLLATE utf8mb4_unicode_ci DEFAULT NULL,
      `published` tinyint(1) NOT NULL DEFAULT '0',
      `authorId` int(11) NOT NULL,
      PRIMARY KEY (`id`) /*T![clustered_index] CLUSTERED */,
      CONSTRAINT `Post_authorId_fkey` FOREIGN KEY (`authorId`) REFERENCES `User` (`id`) ON DELETE RESTRICT ON UPDATE CASCADE
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci
    1 row in set (0.00 sec)

    mysql>
    ```
