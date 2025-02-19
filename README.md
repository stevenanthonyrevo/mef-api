# Monero Economic Forum API
## Written in *Gin* Go Web Framework

By [Charlie](https://home.civdev.xyz) (CM-IV)

Dockerized REST API and PostgreSQL database used featuring sqlc type-safe Go generator.
Hactoberfest 2022 contributers welcome for helping with documentation!

---

**Table of Contents**

1. [CRUD Functionality Quick Peek](#crud-functionality-quick-peek) (You are here)
2. [Makefile Usage](#makefile-usage)
3. [PostgreSQL Queries and Migrations](#postgresql-queries-and-migrations)
4. [sqlc Generator](#sqlc-generator)
5. [Custom CRUD Implementation](#custom-crud-implementation)
6. [Unit Tests and RSG Util](#unit-tests-and-rsg-util) (Documentation In Progress)
7. [Mocked DB for HTTP API testing](#unit-tests-and-rsg-util) (Documentation In Progress)
8. [Minimum Required Ports](#minimum-required-ports) (Minimum Required Ports)

---

### CRUD Functionality Quick Peek

HTTP protocol is used to exchange representations of resources between the client frontend and the server API backend.  Post data is retrieved and accessed using URIs, here are the endpoints along with their operations:

*server.go*
```go
api := router.Group("/api")
	{

		authRoutes := router.Group("/api")
		authRoutes.Use(authMiddleware(server.tokenMaker))
		{
			//PROTECTED ENDPOINTS
			//POSTS ENDPOINTS
			authRoutes.PUT("/posts/:id", server.updatePost)
			authRoutes.DELETE("/posts/:id", server.deletePost)
			authRoutes.POST("/posts", server.createPost)
		}
		api.GET("/posts/:id", server.getPost)
		api.GET("/posts", server.listPost)

		//USERS ENDPOINTS
		api.POST("/users", server.createUser)
		api.POST("/users/login", server.loginUser)
	}
```

---

### Makefile Usage

The Makefile is to be created and contains all the needed commands for everything from bringing up the database (DB) to seeding it with data and performing up/down migrations.

Bring up the database container with the Makefile command `make composeup`.

The container can be stopped with the `make composestop` command.  This frees up the terminal to be used for other things once the DB is restarted with `make composestart`.

In order to run migrations, use the `make migrateup` and `make migratedown` commands.  The docker-compose file is setup to where the sh script runs the migrations for you on startup, however.

Create the Postgres DB itself with a root username and owner.  The DB is named "meforum", this command can be utilized with `make createdb`.  In order to drop the DB, `make dropdb` is used.

Additional commands for DB migrations, sqlc code generation, and testing will be explained later on in their respective sections.

*Makefile*

```makefile
network:
	docker network create mef-network

postgres:
	docker run --name postgres --network mef-network -p 5432:5432 -e POSTGRES_USER=root -e POSTGRES_PASSWORD=postgres -d postgres:13-alpine

composeup:
	docker-compose up

composestart:
	docker-compose start

composestop:
	docker-compose stop	

composedown:
	docker-compose down

createdb:
	docker exec -it mef-api-postgres-1 createdb --username=root --owner=root meforum

dropdb:
	docker exec -it mef-api-postgres-1 dropdb meforum

migrateup:
	migrate -path db/migration -database "postgresql://root:postgres@localhost:5432/meforum?sslmode=disable" -verbose up

migratedown:
	migrate -path db/migration -database "postgresql://root:postgres@localhost:5432/meforum?sslmode=disable" -verbose down

migrateup1:
	migrate -path db/migration -database "postgresql://root:postgres@localhost:5432/meforum?sslmode=disable" -verbose up 1

migratedown1:
	migrate -path db/migration -database "postgresql://root:postgres@localhost:5432/meforum?sslmode=disable" -verbose down 1

sqlc:
	sqlc generate

test-insert:
	go test -count=1 -v ./db/sqlc

test:
	go test -v -cover ./db/sqlc

server:
	go run main.go

mock:
	mockgen -package mockdb -destination db/mock/store.go github.com/CM-IV/mef-api/db/sqlc Store


.PHONY: composeupup composeupdown composeupstart composeupstop createdb dropdb migrateup migratedown  migrateup1 migratedown1 sqlc test server mock postgres network
```

---

### PostgreSQL Queries and Migrations

**Migrations**

The DB up/down Migrations are an automated and useful way of updating or changing the DB SQL tables themselves.

Shown within the Makefile in the previous section, the DB table is created with the `make migrateup` command and the table can be dropped with the `make migratedown` command.  The migration commands have the path to the local Postgres instance with the ports used within them.

The up migration creates the "posts" table along with a "title" index:

*000001_init_schema.up.sql*
```sql
CREATE TABLE "posts" (
  "id" bigserial PRIMARY KEY,
  "image" varchar NOT NULL,
  "title" varchar NOT NULL,
  "subtitle" varchar NOT NULL,
  "content" text NOT NULL,
  "created_at" timestamptz NOT NULL DEFAULT (now())
);

CREATE INDEX ON "posts" ("title");
```
The second up migration created with [golang-migrate](https://github.com/golang-migrate/migrate) allows us to implement the user entity, along with creating the `uuid-ossp` database extension so that the PostgreSQL database can create UUIDs for each new user.  This second migration file, called `000002_add_users.up.sql`, creates the foreign key inside of the `posts` table that references the `user_name` entry within the `users` table.  It is also important to note the composite unique index constraint that has been added, this will prevent the same user from making multiple entries with the same title.

*000002_add_users.up.sql*
```sql
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

CREATE TABLE "users" (
  "id" uuid DEFAULT uuid_generate_v4(),
  "user_name" varchar UNIQUE NOT NULL,
  "hashed_password" varchar NOT NULL,
  "full_name" varchar NOT NULL,
  "email" varchar UNIQUE NOT NULL,
  "created_at" timestamptz NOT NULL DEFAULT (now()),
  PRIMARY KEY (id)
);

ALTER TABLE "posts" ADD FOREIGN KEY ("owner") REFERENCES "users" ("user_name");

-- CREATE UNIQUE INDEX ON "posts" ("title", "owner");
ALTER TABLE "posts" ADD CONSTRAINT "owner_title_key" UNIQUE ("title", "owner");
```

**Queries**

The [sqlc documentation](https://docs.sqlc.dev/en/latest/tutorials/getting-started.html) was very helpful in the creation of the SQL queries.  Sqlc itself will be expanded on in the next section, but this should be mentioned.  The `CreatePost`, `GetPost`, `ListPosts`, `UpdatePost`, and `DeletePost` SQL queries are all included here:


*post.sql*
```sql
-- name: CreatePost :one
INSERT INTO posts (
  image,
  title,
  subtitle,
  content
) VALUES (
  $1, $2, $3, $4
)
RETURNING *;


-- name: GetPost :one
SELECT * FROM posts
WHERE id = $1 LIMIT 1;

-- name: ListPosts :many
SELECT * FROM posts
ORDER BY id
LIMIT $1
OFFSET $2;

-- name: UpdatePost :one
UPDATE posts
SET content = $2
WHERE id = $1
RETURNING *;

-- name: DeletePost :exec
DELETE FROM posts
WHERE id = $1;

```

Shown within this file are the crucial comments that help sqlc do its work in generating the type-safe and idiomatic interfaces to the SQL queries.  The query annotations `one`, `many`, and `exec` tell sqlc that the query returns that many rows.

```sql
-- name: CreateUser :one
INSERT INTO users (
  user_name,
  hashed_password,
  full_name,
  email
) VALUES (
  $1, $2, $3, $4
)
RETURNING *;

-- name: GetUser :one
SELECT * FROM users
WHERE user_name = $1 LIMIT 1;

-- name: ListUsers :many
SELECT * FROM users
ORDER BY id
LIMIT $1
OFFSET $2;
```
Sqlc aids in generating the database interface code so that we can deal with the main authentication and user functionality.  Shown above is the newly added `CreateUser`, `GetUser`, and `ListUsers` sql queries.

---

### sqlc Generator

Sqlc is configured using the sqlc.yaml file which must be in the directory that the sqlc command itself is run.  See their [config reference documentation](https://docs.sqlc.dev/en/latest/reference/config.html) for more details about what each key does.

*sqlc.yaml*
```yaml
version: "1"
packages:
  - name: "db"
    path: "./db/sqlc"
    queries: "./db/query/"
    schema: "./db/migration/"
    engine: "postgresql"
    emit_json_tags: true
    emit_prepared_queries: false
    emit_interface: false
    emit_exact_table_names: false
    emit_empty_slices: true

```

The aforementioned sqlc [docs](https://docs.sqlc.dev/en/latest/howto/select.html) shed light on a Queries struct with DB access methods which is created using the `New` method.  This is located within the db.go file.

*db.go*
```go
package db

import (
	"context"
	"database/sql"
)

type DBTX interface {
	ExecContext(context.Context, string, ...interface{}) (sql.Result, error)
	PrepareContext(context.Context, string) (*sql.Stmt, error)
	QueryContext(context.Context, string, ...interface{}) (*sql.Rows, error)
	QueryRowContext(context.Context, string, ...interface{}) *sql.Row
}

func New(db DBTX) *Queries {
	return &Queries{db: db}
}

type Queries struct {
	db DBTX
}

func (q *Queries) WithTx(tx *sql.Tx) *Queries {
	return &Queries{
		db: tx,
	}
}
```

The pointer to sql.DB `*sql.DB` is within the store.go file to be used with the `NewStore(db *sql.DB)` method which returns a pointer reference in the Store instance that is created.

*store.go*
```go
package db

import (
	"database/sql"
)

//Store will allow DB execute queries and transactions for all functions
//Composition extending struct functionality
type Store struct {
	*Queries
	db *sql.DB
}

func NewStore(db *sql.DB) *Store {

	return &Store{

		db:      db,
		Queries: New(db),
	}

}

```

The models.go file generated by sqlc sets up the type Post struct and the resulting json formatting that comes with it.  The json is configured to use lowercase representations of the rows.  The User struct that is generated enables user authentication functionality to be implemented, using UUIDs as identification strings for each user.

*models.go*
```go
package db

import (
	"time"

	"github.com/google/uuid"
)

type Post struct {
	ID        int64     `json:"id"`
	Owner     string    `json:"owner"`
	Image     string    `json:"image"`
	Title     string    `json:"title"`
	Subtitle  string    `json:"subtitle"`
	Content   string    `json:"content"`
	CreatedAt time.Time `json:"created_at"`
}

type User struct {
	ID             uuid.UUID `json:"id"`
	UserName       string    `json:"user_name"`
	HashedPassword string    `json:"hashed_password"`
	FullName       string    `json:"full_name"`
	Email          string    `json:"email"`
	CreatedAt      time.Time `json:"created_at"`
}
```

The post.sql.go file that interfaces with the PostgreSQL DB is generated along with the previously shown files.  This go code is both type-safe and idiomatic, allowing the programmer to write his/her own custom API application code.  The `CreatePost`, `GetPost`, `ListPosts`, `UpdatePosts`, and `DeletePost` generated functions are within this file.

*post.sql.go*
```go
// source: post.sql

package db

import (
	"context"
)

const createPost = `-- name: CreatePost :one
INSERT INTO posts (
  image,
  title,
  subtitle,
  content
) VALUES (
  $1, $2, $3, $4
)
RETURNING id, image, title, subtitle, content, created_at
`

type CreatePostParams struct {
	Image    string `json:"image"`
	Title    string `json:"title"`
	Subtitle string `json:"subtitle"`
	Content  string `json:"content"`
}

func (q *Queries) CreatePost(ctx context.Context, arg CreatePostParams) (Post, error) {
	row := q.db.QueryRowContext(ctx, createPost,
		arg.Image,
		arg.Title,
		arg.Subtitle,
		arg.Content,
	)
	var i Post
	err := row.Scan(
		&i.ID,
		&i.Image,
		&i.Title,
		&i.Subtitle,
		&i.Content,
		&i.CreatedAt,
	)
	return i, err
}

const deletePost = `-- name: DeletePost :exec
DELETE FROM posts
WHERE id = $1
`

func (q *Queries) DeletePost(ctx context.Context, id int64) error {
	_, err := q.db.ExecContext(ctx, deletePost, id)
	return err
}

const getPost = `-- name: GetPost :one
SELECT id, image, title, subtitle, content, created_at FROM posts
WHERE id = $1 LIMIT 1
`

func (q *Queries) GetPost(ctx context.Context, id int64) (Post, error) {
	row := q.db.QueryRowContext(ctx, getPost, id)
	var i Post
	err := row.Scan(
		&i.ID,
		&i.Image,
		&i.Title,
		&i.Subtitle,
		&i.Content,
		&i.CreatedAt,
	)
	return i, err
}

const listPosts = `-- name: ListPosts :many
SELECT id, image, title, subtitle, content, created_at FROM posts
ORDER BY id
LIMIT $1
OFFSET $2
`

type ListPostsParams struct {
	Limit  int32 `json:"limit"`
	Offset int32 `json:"offset"`
}

func (q *Queries) ListPosts(ctx context.Context, arg ListPostsParams) ([]Post, error) {
	rows, err := q.db.QueryContext(ctx, listPosts, arg.Limit, arg.Offset)
	if err != nil {
		return nil, err
	}
	defer rows.Close()
	items := []Post{}
	for rows.Next() {
		var i Post
		if err := rows.Scan(
			&i.ID,
			&i.Image,
			&i.Title,
			&i.Subtitle,
			&i.Content,
			&i.CreatedAt,
		); err != nil {
			return nil, err
		}
		items = append(items, i)
	}
	if err := rows.Close(); err != nil {
		return nil, err
	}
	if err := rows.Err(); err != nil {
		return nil, err
	}
	return items, nil
}

const updatePost = `-- name: UpdatePost :one
UPDATE posts
SET content = $2
WHERE id = $1
RETURNING id, image, title, subtitle, content, created_at
`

type UpdatePostParams struct {
	ID      int64  `json:"id"`
	Content string `json:"content"`
}

func (q *Queries) UpdatePost(ctx context.Context, arg UpdatePostParams) (Post, error) {
	row := q.db.QueryRowContext(ctx, updatePost, arg.ID, arg.Content)
	var i Post
	err := row.Scan(
		&i.ID,
		&i.Image,
		&i.Title,
		&i.Subtitle,
		&i.Content,
		&i.CreatedAt,
	)
	return i, err
}
```

The SQL query along with the type struct for each function precedes the generated Go code for that respective operation.  `DeletePost` and `GetPost` do not have types defined for them, as they are the only exceptions.

*user.sql.go*
```go
package db

import (
	"context"
)

const createUser = `-- name: CreateUser :one
INSERT INTO users (
  user_name,
  hashed_password,
  full_name,
  email
) VALUES (
  $1, $2, $3, $4
)
RETURNING id, user_name, hashed_password, full_name, email, created_at
`

type CreateUserParams struct {
	UserName       string `json:"user_name"`
	HashedPassword string `json:"hashed_password"`
	FullName       string `json:"full_name"`
	Email          string `json:"email"`
}

func (q *Queries) CreateUser(ctx context.Context, arg CreateUserParams) (User, error) {
	row := q.db.QueryRowContext(ctx, createUser,
		arg.UserName,
		arg.HashedPassword,
		arg.FullName,
		arg.Email,
	)
	var i User
	err := row.Scan(
		&i.ID,
		&i.UserName,
		&i.HashedPassword,
		&i.FullName,
		&i.Email,
		&i.CreatedAt,
	)
	return i, err
}

const getUser = `-- name: GetUser :one
SELECT id, user_name, hashed_password, full_name, email, created_at FROM users
WHERE user_name = $1 LIMIT 1
`

func (q *Queries) GetUser(ctx context.Context, userName string) (User, error) {
	row := q.db.QueryRowContext(ctx, getUser, userName)
	var i User
	err := row.Scan(
		&i.ID,
		&i.UserName,
		&i.HashedPassword,
		&i.FullName,
		&i.Email,
		&i.CreatedAt,
	)
	return i, err
}

const listUsers = `-- name: ListUsers :many
SELECT id, user_name, hashed_password, full_name, email, created_at FROM users
ORDER BY id
LIMIT $1
OFFSET $2
`

type ListUsersParams struct {
	Limit  int32 `json:"limit"`
	Offset int32 `json:"offset"`
}

func (q *Queries) ListUsers(ctx context.Context, arg ListUsersParams) ([]User, error) {
	rows, err := q.db.QueryContext(ctx, listUsers, arg.Limit, arg.Offset)
	if err != nil {
		return nil, err
	}
	defer rows.Close()
	items := []User{}
	for rows.Next() {
		var i User
		if err := rows.Scan(
			&i.ID,
			&i.UserName,
			&i.HashedPassword,
			&i.FullName,
			&i.Email,
			&i.CreatedAt,
		); err != nil {
			return nil, err
		}
		items = append(items, i)
	}
	if err := rows.Close(); err != nil {
		return nil, err
	}
	if err := rows.Err(); err != nil {
		return nil, err
	}
	return items, nil
}
```

---

### Custom CRUD Implementation

This custom application code allows the various CRUD operations to execute within a few miliseconds time at worst, and faster than one milisecond at best.  A slice of posts is used as the dynamic datastructure when listing the posts on the page, so we are dealing with pointers to the array.  Slices let us work with dynamically sized collections of posts, whilst abstracting the array itself and pointing to a contiguous section of the array in memory.  The slice of posts uses a for loop to seed each row in the post.  The columns are then copied in the current row into the values pointed at by the destination.

To see this code, check out the previously shown `post.sql.go` file.

The `post.go` file starts off with the various type structs that are needed by their respective functions.  Luckily, the Gin web framework allows us to perform struct/field data validation with the [validator](https://github.com/go-playground/validator) package.

*post.go*
```go {post.go}
package api

import (
	"database/sql"
	"net/http"

	db "github.com/CM-IV/mef-api/db/sqlc"
	"github.com/gin-gonic/gin"
)

type createPostRequest struct {
	Image    string `json:"image" binding:"required"`
	Title    string `json:"title" binding:"required"`
	Subtitle string `json:"subtitle" binding:"required"`
	Content  string `json:"content" binding:"required"`
}

type getPostRequest struct {
	ID int64 `uri:"id" binding:"required,min=1"`
}

type listPostRequest struct {
	PageID   int32 `form:"page_id" binding:"required,min=1"`
	PageSize int32 `form:"page_size" binding:"required,min=5,max=15"`
}

type updatePostRequest struct {
	Content string `json:"content" binding:"required"`
}
```
The `binding` tag gives conditions that need to be satisfied by the validator, and we use the `context` object to use the `ctx.ShouldBindJSON` function in order to pass input data from the request body.  With GIN, everything that is done inside of the handler has to do with the `context` object.  Checking the [validator](https://github.com/go-playground/validator) documentation, you can see more tags to use in order to validate your request parameters.  However, for this use case, only the `required` tag is needed.

The `ctx.ShouldBindJSON` function returns an error, where if it is not empty - then the client has passed invalid data.  The error handling here will serialize the response and give the client a `400 - Bad Request`.

*post.go*
```go
func (server *Server) createPost(ctx *gin.Context) {

	var req createPostRequest
	if err := ctx.ShouldBindJSON(&req); err != nil {

		ctx.JSON(http.StatusBadRequest, errorResponse(err))
		return

	}

	arg := db.CreatePostParams{

		Image:    req.Image,
		Title:    req.Title,
		Subtitle: req.Subtitle,
		Content:  req.Content,
	}

	post, err := server.store.CreatePost(ctx, arg)

	if err != nil {

		ctx.JSON(http.StatusInternalServerError, errorResponse(err))
		return

	}

	ctx.JSON(http.StatusOK, post)

}
```

The `errorResponse()` function here points to the implementation inside the `server.go` file.  This is a `gin.H` object, which is basically a `map[string]interface{}`.  This allows us to store however many key value pairs we want.

*server.go*
```go
func errorResponse(err error) gin.H {

	return gin.H{"error": err.Error()}

}
```


--- 

### Minimum Required Ports

Running the docker compose setup correctly has a few minimum port requirements. 

- 5432
- 8080

A common error you may see is a failed PostgreSQL driver due to a port not being open.  

```
Error response from daemon: driver failed programming external connectivity on endpoint mef-api-master-postgres-1: Bind for 0.0.0.0:5432 failed: port is already allocated
```

