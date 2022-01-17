---
title: App Development for Ecto
summary: Learn how to build a simple Elixir application using TiDB and Ecto.
---

# App Development for Elixir

This tutorial shows you how to build a simple Elixir application based on TiDB and Ecto. 

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

2. Grant necessary permissions to the SQL user you have just created:

    {{< copyable "sql" >}}

    ```sql
    GRANT ALL ON DATABASE friends_repo TO <username>;
    ```

3. Set `tidb_enable_noop_functions=1` to support Ecto Migrations:

    {{< copyable "sql" >}}

    ```sql
    SET GLOBAL tidb_enable_noop_functions=1;
    ```

## Step 3. Create a Elixir project

1. Creates a new Elixir project called `friends`.

    {{< copyable "" >}}

    ```bash
    $ mix new friends --sup
    ```

2. Edit `mix.exs` to add dependencies.

    {{< copyable "" >}}

    ```bash
    defp deps do
        [
        {:ecto_sql, "~> 3.0"},
        {:myxql, "~> 0.5.0"}
        ]
    end
    ```

3. Run `mix deps.get` to install dependencies.

    {{< copyable "" >}}

    ```shell
    $ mix deps.get
    ```

4. Run `mix ecto.gen.repo -r Friends.Repo` to setup configuration for Ecto.

    {{< copyable "" >}}

    ```shell
    $ mix ecto.gen.repo -r Friends.Repo
    ```

    Modify the `config/config.exs` as follows. This is used for connection to TiDB.

    {{< copyable "" >}}

    ```
    config :friends, Friends.Repo,
      database: "friends_repo",
      username: "root",
      password: "",
      hostname: "localhost",
      port: 4000
    ```

    Update `lib/friends/repo.ex` to use `Ecto.Adapters.MyXQL`

    {{< copyable "" >}}

    ```
    defmodule Friends.Repo do
      use Ecto.Repo,
        otp_app: :friends,
        adapter: Ecto.Adapters.MyXQL
    end
    ```

5. Update `lib/friends/application.ex` to add `Friends.Repo` as a supervisor.

    {{< copyable "" >}}

    ```
    def start(_type, _args) do
      children = [
        Friends.Repo,
      ]
    ....
    ```

## Step 4. Set up and run the Ecto Migration

Now the application configurations are ready, we can run Ecto Migrations.

1. Run `mix ecto.create` to create `friends_repo` at TiDB

    {{< copyable "" >}}

    ```shell
    $ mix ecto.create
    Compiling 3 files (.ex)
    Generated friends app
    The database for Friends.Repo has already been created
    ```

    {{< copyable "" >}}
    ```sql
    mysql> show databases;
    +--------------------+
    | Database           |
    +--------------------+
    | INFORMATION_SCHEMA |
    | METRICS_SCHEMA     |
    | PERFORMANCE_SCHEMA |
    | friends_repo       |
    | mysql              |
    | test               |
    +--------------------+
    6 rows in set (0.01 sec)
    ```

2. Run `mix ecto.gen.migration create_people` to generate a migration file.

    {{< copyable "" >}}

    ```shell
    $ mix ecto.gen.migration create_people
    * creating priv/repo/migrations
    * creating priv/repo/migrations/20220117063503_create_people.exs
    ```

3. Edit `priv/repo/migrations/<timestamp>_create_people.exs` to add a `people` table.

    {{< copyable "" >}}

    ```
    defmodule Friends.Repo.Migrations.CreatePeople do
      use Ecto.Migration

      def change do
        create table(:people) do
          add :first_name, :string
          add :last_name, :string
          add :age, :integer
        end
      end
    end
    ```

4. Run `mix ecto.migrate` to create a `people` table.

    {{< copyable "" >}}

    ```
    $ mix ecto.migrate
    15:38:26.013 [info]  == Running 20220117063503 Friends.Repo.Migrations.CreatePeople.change/0 forward
    15:38:26.019 [info]  create table people
    15:38:26.139 [info]  == Migrated 20220117063503 in 0.1s
    ```

    {{< copyable "sql" >}}

    ```sql
    mysql> use friends_repo
    mysql> show create table people\G
    *************************** 1. row ***************************
           Table: people
    Create Table: CREATE TABLE `people` (
      `id` bigint(20) unsigned NOT NULL AUTO_INCREMENT,
      `first_name` varchar(255) DEFAULT NULL,
      `last_name` varchar(255) DEFAULT NULL,
       `age` int(11) DEFAULT NULL,
      PRIMARY KEY (`id`) /*T![clustered_index] CLUSTERED */
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_bin
    1 row in set (0.00 sec)
    ```

