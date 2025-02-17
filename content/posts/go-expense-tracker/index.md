---
date: "2025-02-17"
draft: false
title: "CRUD Operations with Go"
tags: ["HTTP", "Go", "CRUD","REST"]
categories: ["Backend"]
summary: "Writing a simple REST API using Go."
weight: 1
cover:
  image: "api-cover.jpg"
  relative: true
  hidden: false
  hiddenInSingle: false
---

## Introduction

In our [last article]({{< relref "/posts/go-security/" >}}),
we saw how to set up a simple OAuth2 Password Grant flow 
for our application.
In this article, we will go through and implement some of the remaining 
CRUD operations listed in the [roadmap.sh project](https://roadmap.sh/projects/expense-tracker-api).

## Refactors

Before we get started, let's refactor the `ValidateToken` code from last week.
Let's rewrite it to write an HTTP status code anytime we run into validation
issues instead of our handlers dealing with the errors.

```go
func ValidateToken(w http.ResponseWriter, r *http.Request) string {
	jwtSecret, secretSet := os.LookupEnv("JWT_SECRET")
	if !secretSet {
		http.Error(w, "JWT_SECRET not set", http.StatusInternalServerError)
		return ""
	}
	auth := r.Header.Get("Authorization")
	if auth == "" {
		http.Error(w, "token not set", http.StatusUnauthorized)
		return ""
	}
	token := strings.Split(auth, " ")
	if token[0] != "Bearer" {
		http.Error(w, "token not set", http.StatusUnauthorized)
		return ""
	}
	if len(token) != 2 {
		http.Error(w, "token not set", http.StatusUnauthorized)
		return ""
	}
```

And then we can rewrite our validate `GET /validate` handler inside the `SecurityRoutes` function
of `security.go` as follows

```go
	http.HandleFunc("GET /validate", func(w http.ResponseWriter, r *http.Request) {
		if username := ValidateToken(w, r); username != "" {
			fmt.Fprint(w, username)
		}
	})
```

## Design

The project calls out that we should be able to peform the following CRUD operations:

- [x] Create a new user
- [x] Generate and validate JWTS
- [ ] Create an expense
- [ ] Delete an expense
- [ ] Update an expense
- [ ] List and filter expenses

In addition, we need to categorize the expense. I will choose to 
use the following

- Housing
- Utilities
- Insurance
- Financing 
- Groceries
- Medical
- Entertainment
- Travel
- Other

## Defining Our Expense Schema And Queries

We will need to create a new table for our expenses. 
Since we will be using foriegn keys and sqlite does 
not enforce foreign keys by default, we will need to
modify the `setup` function in our `main.go` as follows

```go {linenos=table,hl_lines=[9]}
func setup() (context.Context, *database.Queries, error) {
	ctx := context.Background()
	dbfilename, envset := os.LookupEnv("DATABASE_FILE")
	if !envset {
		return nil, &database.Queries{},
			errors.New("DATABASE_FILE env variable not set")
	}
	db, err := sql.Open("sqlite3", dbfilename)
	db.ExecContext(ctx, "PRAGMA foreign_keys = ON;", nil)
	if err != nil {
		return nil, &database.Queries{}, err
	}
	queries := database.New(db)
	return ctx, queries, nil
}
```

The `PRAGMA` command is used to enable foreign keys.

Then we will create two new tables, one for the expenses and another for the categories.

First let's rollback our database from scratch, again since we only have test users that
do not need to be saved. 

```bash
~>dbmate rollback
Rolling back: 20250217170014_create-users.sql
Rolled back: 20250217170014_create-users.sql in 3.548604ms
```

Now let's modify our `create-users.sql` file as follows

```sql {linenos=table,hl_lines=[3]}
-- migrate:up
create table users (
  name text primary key,
  salt blob not null,
  hashpassword blob not null
);

-- migrate:down
drop table users;
```

Here I removed the id as primary key and made the name the primary key instead. 
Since our JWT is based on the username, we will use the username as the primary key.

Now let's create our `create-expense.sql` file as follows

```sql 
-- migrate:up
create table categories (
  category text primary key
);

insert into categories (category) values
('Housing'),
('Utilities'),
('Insurance'),
('Financing '),
('Groceries'),
('Medical'),
('Entertainment'),
('Travel'),
('Other');

 create table expenses (
   id integer primary key,
   user text references users(name) not null,
   description text not null,
   category text references categories(category) not null,
   amount decimal not null,
   created_date date not null default current_date
 );

-- migrate:down
drop table expenses;
drop table categories;
```

Here we created a new table for the categories and a new table for the expenses.
Let's run the migrations

```bash
~/Projects/task-manager>dbmate up
Applying: 20250215170014_create-users.sql
Applied: 20250215170014_create-users.sql in 20.391793ms
Applying: 20250217135647_create-expenses.sql
Applied: 20250217135647_create-expenses.sql in 4.538702ms
Writing: ./db/schema.sql
```

Now let's write add the following queries 
to our `db/queries.sql` file to interface with our new expense table.

```sql {linenos=table,hl_lines=["13-41"]}
-- name: InsertUser :one
insert into users (name, salt, hashpassword)
values (?, ?, ?)
returning name;

-- name: GetUsers :many
select * from users;

-- name: GetCredentials :one
select salt, hashpassword from users
where name = ?;

-- name: InsertExpense :one
insert into expenses (user, category, amount, description)
values (?, ?, ?, ?)
returning id;

-- name: GetExpenses :many
select * from expenses
where user = ?;

-- name: GetExpensesByCategory :many
select * from expenses
where category = ? and user = ?;

-- name: GetExpensesByDate :many
select * from expenses
where created_date >= ? and user = ?;

-- name: GetExpensesByDateAndCategory :many
select * from expenses
where created_date >= ? and category = ? and user = ?;

-- name: DeleteExpense :exec
delete from expenses
where id = ?;

-- name: UpdateExpense :exec
update expenses
set category = ?, amount = ?, description = ?
where id = ?;
```

Then we generate our database code by running

```bash
~/Projects/task-manager>sqlc generate
```

And now we can start creating some routes for our expense tracker.

## Inserting an Expense

Let's start by creating the `ExpenseRoutes` function inside a new file called `internal/expense.go`

```go
package internal

import (
	"context"
	"database/sql"
	"fmt"
	"net/http"
	"strconv"

	"github.com/acswindle/task-manager/database"
)

func ExpenseRoutes(ctx context.Context, queries *database.Queries) {
	// GET /users
	http.HandleFunc("POST /api/expense", func(w http.ResponseWriter, r *http.Request) {
		err := r.ParseForm()
		if err != nil {
			http.Error(w, err.Error(), http.StatusBadRequest)
			return
		}
		amount := r.FormValue("amount")
		description := r.FormValue("description")
		category := r.FormValue("category")
		if amount == "" || description == "" || category == "" {
			http.Error(w, "amount, description, and category are required", http.StatusBadRequest)
			return
		}
		amount_f, err := strconv.ParseFloat(amount, 64)
		if err != nil {
			http.Error(w, "invalid amount", http.StatusBadRequest)
			return
		}
		username := ValidateToken(w, r)
		if username == "" {
			return
		}
		id, err := queries.InsertExpense(ctx, database.InsertExpenseParams{
			User:        username,
			Amount:      amount_f,
			Description: description,
			Category:    category,
		})
		if err != nil {
			http.Error(w, err.Error(), http.StatusInternalServerError)
			return
		}
		w.Header().Set("Content-Type", "application/json")
		w.Write([]byte("{\"id\": " + strconv.Itoa(int(id)) + "}"))
	})
}
```

There isn't much new here that hasn't been seen before.
We do some basic validation on the input form data and then 
we utilize our `ValidateToken` function
to ensure that the user is authenticated before they can make a request.
The sqlc generated `InsertExpense` function makes it then simple to insert the expense into the database.
Finally, we send a response back to the client with the id of the new expense.

## Getting Expenses

Getting by the user's expenses is a bit more complex, due to the various
ways that we can filter the expenses. Let's start by just getting all of the
expenses for the user. Let's create a new route in our `internal/expense.go` file.

```go
	http.HandleFunc("GET /api/expenses", func(w http.ResponseWriter, r *http.Request) {
		username := ValidateToken(w, r)
		if username == "" {
			return
		}
		expenses, err := queries.GetExpenses(ctx, username)
		if err != nil {
			http.Error(w, err.Error(), http.StatusInternalServerError)
			return
		}
		payload, err := json.Marshal(expenses)
		if err != nil {
			http.Error(w, err.Error(), http.StatusInternalServerError)
			return
		}
		w.Header().Set("Content-Type", "application/json")
		w.Write(payload)
	})
```

So this is fairly simple. We validate the token, get the username, and then
use the `GetExpenses` sqlc generated function to get the expenses for the user.

To add a category filtering capability, we could expand it to include some query parameters 
like so.

```go {linenos=table,hl_lines=["6-18"]}
	http.HandleFunc("GET /api/expenses", func(w http.ResponseWriter, r *http.Request) {
		username := ValidateToken(w, r)
		if username == "" {
			return
		}
		// TODO: Add Category Check
		category := r.URL.Query().Get("category")
		var payload []byte
		var expenses []database.Expense
		var err error
		if category != "" {
			expenses, err = queries.GetExpensesByCategory(ctx, database.GetExpensesByCategoryParams{
				User:     username,
				Category: category,
			})
		} else {
			expenses, err = queries.GetExpenses(ctx, username)
		}
		if err != nil {
			http.Error(w, err.Error(), http.StatusInternalServerError)
			return
		}
		payload, err = json.Marshal(expenses)
		if err != nil {
			http.Error(w, err.Error(), http.StatusInternalServerError)
			return
		}
		w.Header().Set("Content-Type", "application/json")
		w.Write(payload)
	})
```

Here we check the case if the category query parameter is set. Notice the todo. 
If a user does not send a correct category, they will get no expenses. To better 
help our client, we should in the future check if the category they sent is a valid category first 
and return an error if it is not.

Next we add in the `GetExpensesByDate` and `GetExpensesByDateAndCategory` functions.

```go {linenos=table,hl_lines=["9-29"]}
	http.HandleFunc("GET /api/expenses", func(w http.ResponseWriter, r *http.Request) {
		username := ValidateToken(w, r)
		if username == "" {
			return
		}
		// TODO: Add Category Check
		category := r.URL.Query().Get("category")
		date := r.URL.Query().Get("date")
		var filterDate time.Time
		var err error
		if date != "" {
			filterDate, err = time.Parse("2006-01-02", date)
			if err != nil {
				http.Error(w, "invalid date, must be YYYY-MM-DD format", http.StatusBadRequest)
				return
			}
		}
		var expenses []database.Expense
		if category != "" && date != "" {
			expenses, err = queries.GetExpensesByDateAndCategory(ctx, database.GetExpensesByDateAndCategoryParams{
				User:        username,
				Category:    category,
				CreatedDate: filterDate,
			})
		} else if date != "" {
			expenses, err = queries.GetExpensesByDate(ctx, database.GetExpensesByDateParams{
				User:        username,
				CreatedDate: filterDate,
			})
		} else if category != "" {
			expenses, err = queries.GetExpensesByCategory(ctx, database.GetExpensesByCategoryParams{
				User:     username,
				Category: category,
			})
		} else {
			expenses, err = queries.GetExpenses(ctx, username)
		}
		if err != nil {
			http.Error(w, err.Error(), http.StatusInternalServerError)
			return
		}
		var payload []byte
		payload, err = json.Marshal(expenses)
		if err != nil {
			http.Error(w, err.Error(), http.StatusInternalServerError)
			return
		}
		w.Header().Set("Content-Type", "application/json")
		w.Write(payload)
	})
```

And we now have a way to filter the expenses by date and category.
The project leaves several other ways to filter the expenses. I will leave those to you to 
implement if you wish.

## Deleting Expenses

Now lets quickly implement the `DELETE /api/expense/:id` route inside our `internal/expense.go` file.

```go 
	http.HandleFunc("DELETE /api/expense/{id}", func(w http.ResponseWriter, r *http.Request) {
		idStr := r.PathValue("id")
		id, err := strconv.ParseInt(idStr, 10, 64)
		if err != nil {
			http.Error(w, "invalid id, must be an integer", http.StatusBadRequest)
			return
		}
		username := ValidateToken(w, r)
		if username == "" {
			return
		}
		err = queries.DeleteExpense(ctx, id)
		if err != nil {
			http.Error(w, err.Error(), http.StatusInternalServerError)
			return
		}
		w.WriteHeader(http.StatusOK)
	})
```

This method is again pretty simple. The only new concept we have is the `PathValue` function.
This is a helper function that allows us to get the value of a specific path parameter.
It was implement introduced to the `net/http` package in Go 1.22. Before then you 
had to import external packages such as [gorilla](https://github.com/gorilla/mux).

## Updating Expenses

Now lets implement the `PUT /api/expense/:id` route inside our `internal/expense.go` file.

```go
	http.HandleFunc("PATCH /api/expense/{id}", func(w http.ResponseWriter, r *http.Request) {
		idStr := r.PathValue("id")
		id, err := strconv.ParseInt(idStr, 10, 64)
		if err != nil {
			http.Error(w, "invalid id, must be an integer", http.StatusBadRequest)
			return
		}
		username := ValidateToken(w, r)
		if username == "" {
			return
		}
		if err := r.ParseForm(); err != nil {
			http.Error(w, err.Error(), http.StatusBadRequest)
			return
		}
		amount := r.FormValue("amount")
		description := r.FormValue("description")
		category := r.FormValue("category")
		if amount == "" || description == "" || category == "" {
			http.Error(w, "amount, description, and category are required", http.StatusBadRequest)
			return
		}
		amount_f, err := strconv.ParseFloat(amount, 64)
		if err != nil {
			http.Error(w, "invalid amount", http.StatusBadRequest)
			return
		}
		err = queries.UpdateExpense(ctx, database.UpdateExpenseParams{
			ID:          id,
			Amount:      amount_f,
			Description: description,
			Category:    category,
		})
		if err != nil {
			http.Error(w, err.Error(), http.StatusInternalServerError)
			return
		}
		w.WriteHeader(http.StatusOK)
	})
```

This method is again pretty simple, and no new concepts are introduced.

## Conclusion

Now our expense tracker is complete! We have implemented all of the routes that we need to
perform the CRUD operations for expenses. Of course the functionality can still be improved, 
such as adding more filtering options to the `GET /api/expense` route. We also still need to
implement the `GET /api/expense/:id` route to get a single expense. In additional, there are 
times that we are relying on the sqlite database engine to check the validity of the data.
We should instead check the validity of the data before we trying save it to the database.
This will allow us to provide the client with more useful error messages. If you would try to 
implement some of these, I encourage you to do so, and let me know if you have any questions.

The code for this project up to this point can be found [here](https://github.com/acswindle/task-manager/tree/expense-crud). 

Next time I will show you how to build our application into a docker container and how to 
upload it to a docker registry. This will enable us to easily deploy our application to 
AWS.

Until next time! 🍻
