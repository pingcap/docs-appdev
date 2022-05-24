---
title: App Development for Ent
summary: Learn how to build a simple Golang application based on TiDB and Ent.
---

# App Development for Ent

This tutorial shows you how to build a simple Golang application based on TiDB and Ent. The sample application to build here is a simple Todo app where you can add, query, and update Todos.

## Step 1. Start a TiDB cluster

Start a pseudo TiDB cluster on your local storage:

{{< copyable "" >}}

```bash
docker run -p 127.0.0.1:$LOCAL_PORT:4000 pingcap/tidb:v6.0.0
```

The above command starts a temporary and single-node cluster with mock TiKV. The cluster listens on the port `$LOCAL_PORT`. After the cluster is stopped, any changes already made to the database are not persisted.

> **Note:**
>
> To deploy a "real" TiDB cluster for production, see the following guides:
>
> + [Deploy TiDB using TiUP for On-Premises](https://docs.pingcap.com/tidb/v5.4/production-deployment-using-tiup)
> + [Deploy TiDB on Kubernetes](https://docs.pingcap.com/tidb-in-kubernetes/stable)
>
> You can also [use TiDB Cloud](https://pingcap.com/products/tidbcloud/), a fully-managed Database-as-a-Service (DBaaS), which offers free trial.

## Step 2. Create a database

1. In the SQL shell, create the `entgo` database that your application will use:

    {{< copyable "" >}}

    ```sql
    CREATE DATABASE entgo;
    ```

2. Create a SQL user for your application:

    {{< copyable "" >}}

    ```sql
    CREATE USER <username> IDENTIFIED BY <password>;
    ```

    Take note of the username and password. You will use them in your application code when initializing the project.

3. Grant necessary permissions to the SQL user you have just created:

    {{< copyable "" >}}

    ```sql
    GRANT ALL ON entgo.* TO <username>;
    ```

## Step 3. Get and run the application code

### Prerequisites
Make sure you have [Go](https://golang.org/doc/install) installed (1.17 or above).  
Create a folder for this demo, and initialize using `go mod`:

{{< copyable "" >}}

```bash
mkdir todo
cd todo
go mod init todo
```
### Initialize project
First, we need to install Ent and initialize the project structure. run:

{{< copyable "" >}}

```bash
go run -mod=mod entgo.io/ent/cmd/ent init Todo
```
After running the above, your project directory should look like this:
```bash
.
├── ent
│   ├── generate.go
│   └── schema
│       └── todo.go
├── go.mod
└── go.sum
```
Open `schema/todo.go` and add some fields to the `Todo` entity:

{{< copyable "" >}}

```go
func (Todo) Fields() []ent.Field {
	return []ent.Field{
		field.String("title"),
		field.String("content"),
		field.Bool("done").
			Default(false),
		field.Time("created_at"),
	}
}
```
Finally, run:

{{< copyable "" >}}

```bash
go generate ./ent
```

### Run the example application
Create a file named `main.go` in your project's root folder and copy to it the the following code:

{{< copyable "" >}}

```go
package main

import (
	"context"
	"fmt"
	"log"
	"time"
	"todo/ent"
	"todo/ent/todo"

	"entgo.io/ent/dialect/sql/schema"
	_ "github.com/go-sql-driver/mysql"
)

func main() {
	// Connect to TiDB.
	client, err := ent.Open("mysql", "root@tcp(127.0.0.1:4000)/entgo?parseTime=true")
	if err != nil {
		log.Fatal("error opening ent client", err)
	}
	defer client.Close()
	// Migrate the local schema with the DB.
	if err := client.Schema.Create(
		context.Background(),
		// TiDB support is possible with Atlas engine only.
		schema.WithAtlas(true),
	); err != nil {
		log.Fatal("error migrating DB", err)
	}

	// Create some todos
	client.Todo.Create().
		SetTitle("buy groceries").
		SetContent("tomato, lettuce and cucumber").
		SetCreatedAt(time.Now()).
		SaveX(context.Background())
	client.Todo.Create().
		SetTitle("See my doctor").
		SetContent("periodic blood test").
		SetCreatedAt(time.Now()).
		SaveX(context.Background())

	// query todo with 'where' filter
	todos := client.Todo.Query().
		Where(todo.ContentContains("tomato")).
		AllX(context.Background())
	fmt.Printf("The following todos contain 'tomato' in their content: %v\n", todos)

	// after buying the groceries, set the todo as done.
	client.Todo.UpdateOne(todos[0]).SetDone(true).ExecX(context.Background())

	// delete todos that are done.
	deletedCount := client.Todo.Delete().Where(todo.Done(true)).ExecX(context.Background())
	fmt.Printf("deleted %d todos\n", deletedCount)
}
```
### Run the code

Run the `main.go` code:

{{< copyable "" >}}

```bash
go run main.go
```

The expected output is as follows:
```
The following todos contain 'tomato' in their content:
[Todo(id=1, title=buy groceries, content=tomato, lettuce and cucumber, done=false, created_at=Mon May  2 14:32:20 2022)]
deleted 1 todos
```

## What's next?

Learn more about how to use [Ent Framework](https://entgo.io/).

You might also be interested in the following:
* Automatically generate feature-rich, performant, and clean [GraphQL](https://entgo.io/docs/graphql/), [REST](https://entgo.io/blog/2021/07/29/generate-a-fully-working-go-crud-http-api-with-ent/), and [gRPC](https://entgo.io/docs/grpc-intro/) servers using Ent
* [Versioned Migrations with Ent](https://entgo.io/docs/versioned-migrations/)
* [Extending Ent with the Extension API](https://entgo.io/blog/2021/09/02/ent-extension-api)
