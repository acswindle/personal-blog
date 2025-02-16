---
date: "2025-02-16"
draft: false
title: "Setting Up A Go Backend"
tags: ["HTTP", "Go", "Server"]
categories: ["Backend"]
summary: "Getting started with a go backend."
weight: 1
cover:
  image: "go-backend-cover.jpg"
  relative: true
  hidden: false
  hiddenInSingle: false
---

## Introduction

In our [last article]({{< relref "/posts/database-project-setup/" >}}),
we saw how to set up a database and easily connect to it
using DBMate for migrations / schema definition and SQLC for
GO query code generation.
In this article, we will see how to set up a simple HTTP server using
Go's built in net/http package.
The ultimate goal of this project will be to satisfy the requirements
set out in this [roadmap.sh project](https://roadmap.sh/projects/expense-tracker-api).
After we complete the requirements I will also go through how
to set up a simple build and deployment pipeline to have our
application hosted on AWS.

## Refactors

Lets first start by refactoring our main.go file from last time a bit.

```go
package main

import (
 "context"
 "database/sql"
 "fmt"
 "github.com/acswindle/task-manager/database"
 _ "github.com/mattn/go-sqlite3"
 "log"
)

func setup() (context.Context, *database.Queries, error) {
 ctx := context.Background()
 db, err := sql.Open("sqlite3", "./db/task-manager.sqlite3")
 if err != nil {
  return nil, &database.Queries{}, err
 }
 queries := database.New(db)
 return ctx, queries, nil
}

func main() {
 fmt.Println("Running Task Manager...")
 _, _, err := setup()
 if err != nil {
  log.Fatal(err)
 }
}
```

Here we turned our run function instead to a database setup
function that returns the context, queries, and error. In
order to turn this into a [12 factor app](https://12factor.net/),
we need to remove our pull in the database file from an
environment variable. Lets pull in the
[godotenv](https://github.com/joho/godotenv) package.

```bash
go get github.com/joho/godotenv
```

And modify our main.go file.

```go {linenos=table,hl_lines=[6,9,12,"17-22",32]}
package main

import (
 "context"
 "database/sql"
 "errors"
 "fmt"
 "github.com/acswindle/task-manager/database"
 "github.com/joho/godotenv"
 _ "github.com/mattn/go-sqlite3"
 "log"
 "os"
)

func setup() (context.Context, *database.Queries, error) {
 ctx := context.Background()
 dbfilename, envset := os.LookupEnv("DATABASE_FILE")
 if !envset {
  return nil, &database.Queries{},
    errors.New("DATABASE_FILE env variable not set")
 }
 db, err := sql.Open("sqlite3", dbfilename)
 if err != nil {
  return nil, &database.Queries{}, err
 }
 queries := database.New(db)
 return ctx, queries, nil
}

func main() {
 fmt.Println("Running Task Manager...")
 godotenv.Load()
 _, _, err := setup()
 if err != nil {
  log.Fatal(err)
 }
}
```

We now are loading up the database file from an environment.

## Setting Up The Server

To set up the server, we will need to import the net/http package.

```go {linenos=table,hl_lines=[9,"35-38","44-49"]}
package main

import (
	"context"
	"database/sql"
	"errors"
	"fmt"
	"log"
	"net/http"
	"os"

	"github.com/acswindle/task-manager/database"
	"github.com/joho/godotenv"
	_ "github.com/mattn/go-sqlite3"
)

func setup() (context.Context, *database.Queries, error) {
	ctx := context.Background()
	dbfilename, envset := os.LookupEnv("DATABASE_FILE")
	if !envset {
		return nil, &database.Queries{},
			errors.New("DATABASE_FILE env variable not set")
	}
	db, err := sql.Open("sqlite3", dbfilename)
	if err != nil {
		return nil, &database.Queries{}, err
	}
	queries := database.New(db)
	return ctx, queries, nil
}

func main() {
	fmt.Println("Running Task Manager...")
	godotenv.Load()
	port, portset := os.LookupEnv("APP_PORT")
	if !portset {
		log.Fatal("APP_PORT env variable not set")
	}
	_, _, err := setup()
	if err != nil {
		log.Fatal(err)
	}

	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprint(w, "Writing from Go Server")
	})

	fmt.Printf("Listening on port %s\n", port)
	http.ListenAndServe(fmt.Sprintf(":%s", port), nil)
}
```
Here we imported net/http, which is part of the standard library.
We continue the pattern of keeping all our configuration information in 
environment variables by registering our port number as 
`APP_PORT=8080` in our .env file.
HandleFunc is a function that allows us to register
handlers with our server. A handler function must take a response writer
and a request. The request is a struct that contains information about the
client request, and we will use it later to access request parameters.
The response writer allows us to write back to the client, similar to 
as if we were writing to a file.

And running this we should see the following.

```bash
~/Projects/task-manager>go run .
Running Task Manager...
Listening on port 8080
```
And now making a request to our server.
```bash
~>curl -s -X 'GET' 'http://localhost:8080/'
Writing from Go Server
```

## Sending Back Users from Database

Let's now create a handler to pull in our users from the database.

```go {linenos=table,hl_lines=[39,"48-55"]}
package main

import (
	"context"
	"database/sql"
	"errors"
	"fmt"
	"log"
	"net/http"
	"os"

	"github.com/acswindle/task-manager/database"
	"github.com/joho/godotenv"
	_ "github.com/mattn/go-sqlite3"
)

func setup() (context.Context, *database.Queries, error) {
	ctx := context.Background()
	dbfilename, envset := os.LookupEnv("DATABASE_FILE")
	if !envset {
		return nil, &database.Queries{},
			errors.New("DATABASE_FILE env variable not set")
	}
	db, err := sql.Open("sqlite3", dbfilename)
	if err != nil {
		return nil, &database.Queries{}, err
	}
	queries := database.New(db)
	return ctx, queries, nil
}

func main() {
	fmt.Println("Running Task Manager...")
	godotenv.Load()
	port, portset := os.LookupEnv("APP_PORT")
	if !portset {
		log.Fatal("APP_PORT env variable not set")
	}
	ctx, queries, err := setup()
	if err != nil {
		log.Fatal(err)
	}

	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprint(w, "Writing from Go Server")
	})

	http.HandleFunc("GET /users", func(w http.ResponseWriter, r *http.Request) {
		users, err := queries.GetUsers(ctx)
		if err != nil {
			http.Error(w, err.Error(), http.StatusInternalServerError)
			return
		}
		fmt.Fprint(w, users)
	})

	fmt.Printf("Listening on port %s\n", port)
	http.ListenAndServe(fmt.Sprintf(":%s", port), nil)
}
```

Here we make use of the GetUsers function generated from our SQLC code,
and simply write the users back to the client.

```bash
~>curl -s -X 'GET' 'http://localhost:8080/users'
[{1 John Doe} {2 John Doe}]
```

And we see our users are returned.

Next lets create a user.

```go {linenos=table,hl_lines=["56-68"]}
package main

import (
	"context"
	"database/sql"
	"errors"
	"fmt"
	"log"
	"net/http"
	"os"

	"github.com/acswindle/task-manager/database"
	"github.com/joho/godotenv"
	_ "github.com/mattn/go-sqlite3"
)

func setup() (context.Context, *database.Queries, error) {
	ctx := context.Background()
	dbfilename, envset := os.LookupEnv("DATABASE_FILE")
	if !envset {
		return nil, &database.Queries{},
			errors.New("DATABASE_FILE env variable not set")
	}
	db, err := sql.Open("sqlite3", dbfilename)
	if err != nil {
		return nil, &database.Queries{}, err
	}
	queries := database.New(db)
	return ctx, queries, nil
}

func main() {
	fmt.Println("Running Task Manager...")
	godotenv.Load()
	port, portset := os.LookupEnv("APP_PORT")
	if !portset {
		log.Fatal("APP_PORT env variable not set")
	}
	ctx, queries, err := setup()
	if err != nil {
		log.Fatal(err)
	}

	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprint(w, "Writing from Go Server")
	})

	http.HandleFunc("GET /users", func(w http.ResponseWriter, r *http.Request) {
		users, err := queries.GetUsers(ctx)
		if err != nil {
			http.Error(w, err.Error(), http.StatusInternalServerError)
			return
		}
		fmt.Fprint(w, users)
	})
	http.HandleFunc("POST /user", func(w http.ResponseWriter, r *http.Request) {
		username := r.URL.Query().Get("username")
		if username == "" {
			http.Error(w, "username not set", http.StatusBadRequest)
			return
		}
		id, err := queries.InsertUser(ctx, username)
		if err != nil {
			http.Error(w, err.Error(), http.StatusInternalServerError)
			return
		}
		fmt.Fprintf(w, "Inserted user with id %d\n", id)
	})

	fmt.Printf("Listening on port %s\n", port)
	http.ListenAndServe(fmt.Sprintf(":%s", port), nil)
}
```

And testing it out.

```bash
~>curl -s -X 'POST' 'http://localhost:8080/user?username=testuser'
Inserted user with id 4
```

## Conclusion

In this article, we saw how to set up a simple HTTP server using
Go's built in net/http package. We also saw how to set up up 
a few simple routes using handlers and then processing client
request using handler functions. We also saw how to interface with
our generated SQLC code to allow clients to interact with our
database. Click [here](https://github.com/acswindle/task-manager/tree/net-http)
to see the full code for this project up to this point.

Next time we will add a secure sign up and login functionality 
to our project using JWTs.
Until next time 🍻
