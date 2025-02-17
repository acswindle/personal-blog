---
date: "2025-02-16"
draft: false
title: "Setting Up Go Authentication"
tags: ["HTTP", "Go", "Security","Authentication","JWT"]
categories: ["Backend"]
summary: "Setting up Go Authentication using Bcrypt and JWT."
weight: 1
cover:
  image: "go-security-cover.jpg"
  relative: true
  hidden: false
  hiddenInSingle: false
---

## Introduction

In our [last article]({{< relref "/posts/go-backend-setup/" >}}),
we saw how to set up a simple HTTP server using
Go's built in net/http package.
The ultimate goal of this project will be to satisfy the requirements
set out in this [roadmap.sh project](https://roadmap.sh/projects/expense-tracker-api).
After we complete the requirements I will also go through how
to set up a simple build and deployment pipeline to have our
application hosted on AWS.

In this article, we will see how to set up Go Authentication using Bcrypt and JWT.

## Refactors

First let us refactor up the project a bit.
I would like to move all the authentication / security related code
into a separate package.

Lets create `internal/secuirty.go`

```go
package internal

import (
	"context"
	"fmt"
	"net/http"

	"github.com/acswindle/task-manager/database"
)

func SecurityRoutes(ctx context.Context, queries *database.Queries) {
	// GET /users
	http.HandleFunc("GET /users", func(w http.ResponseWriter, r *http.Request) {
		users, err := queries.GetUsers(ctx)
		if err != nil {
			http.Error(w, err.Error(), http.StatusInternalServerError)
			return
		}
		fmt.Fprint(w, users)
	})
	// POST /user
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
}
```

And then update our main go code to reference our new SecurityRoutes function.

```go {linenos=table,hl_lines=[13,48]}
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
	"github.com/acswindle/task-manager/internal"
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
	internal.SecurityRoutes(ctx, queries)

	fmt.Printf("Listening on port %s\n", port)
	http.ListenAndServe(fmt.Sprintf(":%s", port), nil)
}
```

## Adding Salt and Hashing Password

When our users sign up, we do not want to directly store their password 
in the database. Instead, we want to store a salted hash of their password.
In addition, we want to add a salt to our password to make it more secure
in the event that the password is weak or easy to guess for hackers in 
the event that our database is leaked.

Since all the users we have added so far have been for testing purposes,
we do note need to keep them in the database. So lets rollback our database.

```bash
~/Projects/task-manager>dbmate rollback
Rolling back: 20250215170014_create-users.sql
Rolled back: 20250215170014_create-users.sql in 3.548604ms
Writing: ./db/schema.sql
```

Now we should have an empty database again. Lets modify our migration.

```sql {linenos=table,hl_lines=["5-6"]}
-- migrate:up
create table users (
  id integer primary key,
  name text not null unique,
  salt blob not null,
  hashpassword blob not null
);

-- migrate:down
drop table users;
```

And then lets add a query to get the user's salt and hash password,
and modify our insert user query to store the salt and hash password.


```sql {linenos=table,hl_lines=["2-3","9-11"]}
-- name: InsertUser :one
insert into users (name, salt, hashpassword)
values (?, ?, ?)
returning id;

-- name: GetUsers :many
select * from users;

-- name: GetCredentials :one
select salt, hashpassword from users
where name = ?;
```

Then we migrate our database and generate our query code.

```bash
~/Projects/task-manager>dbmate migrate
Applying: 20250215170014_create-users.sql
Applied: 20250215170014_create-users.sql in 5.125985ms
Writing: ./db/schema.sql
~/Projects/task-manager>sqlc generate
```

Now we can add the salt and hash password our our insert user http request.
In order to secure our password, we will use Bcrypt package which must
be installed with `go get golang.org/x/crypto/bcrypt`.
After that we modify `internal/security.go` file as such.

```go {linenos=table,hl_lines=[5,9,"14-20","39-59"]}
package internal

import (
	"context"
	"crypto/rand"
	"fmt"
	"net/http"

	"golang.org/x/crypto/bcrypt"

	"github.com/acswindle/task-manager/database"
)

func generateSalt() ([]byte, error) {
	salt := make([]byte, 16)
	if _, err := rand.Read(salt); err != nil {
		return nil, err
	}
	return salt, nil
}

func SecurityRoutes(ctx context.Context, queries *database.Queries) {
	// GET /users
	http.HandleFunc("GET /users", func(w http.ResponseWriter, r *http.Request) {
		users, err := queries.GetUsers(ctx)
		if err != nil {
			http.Error(w, err.Error(), http.StatusInternalServerError)
			return
		}
		fmt.Fprint(w, users)
	})
	// POST /user
	http.HandleFunc("POST /user", func(w http.ResponseWriter, r *http.Request) {
		username := r.URL.Query().Get("username")
		if username == "" {
			http.Error(w, "username not set", http.StatusBadRequest)
			return
		}
		rawPassword := r.URL.Query().Get("password")
		if rawPassword == "" {
			http.Error(w, "password not set", http.StatusBadRequest)
			return
		}
		password := []byte(rawPassword)
		salt, err := generateSalt()
		if err != nil {
			http.Error(w, err.Error(), http.StatusInternalServerError)
			return
		}
		password, err = bcrypt.GenerateFromPassword(append(password, salt...), bcrypt.DefaultCost)
		if err != nil {
			http.Error(w, err.Error(), http.StatusInternalServerError)
			return
		}
		id, err := queries.InsertUser(ctx, database.InsertUserParams{
			Name:         username,
			Hashpassword: password,
			Salt:         salt,
		})
		if err != nil {
			http.Error(w, err.Error(), http.StatusInternalServerError)
			return
		}
		fmt.Fprintf(w, "Inserted user with id %d\n", id)
	})
}
```

The generateSalt function generates a random 16 byte salt using the crypto/rand package.
Then we use the bcrypt.GenerateFromPassword function to generate a hash of the password and salt.
Finally, we insert the user into the database using the newly defined InsertUser function generated by sqlc.

Then let us test out our new SecurityRoutes function.

```bash
~>curl -s -X 'POST' 'http://localhost:8080/user?username=another&password=secret'
Inserted user with id 1
~>curl -s -X 'GET' 'http://localhost:8080/users'
[{1 another [154 118 11 64 84 196 246 56 92 131 130 221 219 133 83 85] [36 50 97 36 49 48 36 66 75 70 119 111 117 100 105 108 118 97 101 117 106 69 77 75 48 65 53 78 79 52 88 106 117 104 99 106 115 65 84 90 102 86 122 115 74 115 65 90 87 122 117 118 50 121 98 78 70 69 89 113]}]
```

## OAuth2 Password Grant Authentication

Now that we have a safe way to sign our users in, we can implement JWT Authentication when they sign in.

First lets create a new route called oauth2/token to our `internal/security.go` file.

```go {linenos=table,hl_lines=["66-91"]}

package internal

import (
	"context"
	"crypto/rand"
	"fmt"
	"net/http"

	"golang.org/x/crypto/bcrypt"

	"github.com/acswindle/task-manager/database"
)

func generateSalt() ([]byte, error) {
	salt := make([]byte, 16)
	if _, err := rand.Read(salt); err != nil {
		return nil, err
	}
	return salt, nil
}

func SecurityRoutes(ctx context.Context, queries *database.Queries) {
	// GET /users
	http.HandleFunc("GET /users", func(w http.ResponseWriter, r *http.Request) {
		users, err := queries.GetUsers(ctx)
		if err != nil {
			http.Error(w, err.Error(), http.StatusInternalServerError)
			return
		}
		fmt.Fprint(w, users)
	})
	// POST /user
	http.HandleFunc("POST /user", func(w http.ResponseWriter, r *http.Request) {
		username := r.URL.Query().Get("username")
		if username == "" {
			http.Error(w, "username not set", http.StatusBadRequest)
			return
		}
		rawPassword := r.URL.Query().Get("password")
		if rawPassword == "" {
			http.Error(w, "password not set", http.StatusBadRequest)
			return
		}
		password := []byte(rawPassword)
		salt, err := generateSalt()
		if err != nil {
			http.Error(w, err.Error(), http.StatusInternalServerError)
			return
		}
		password, err = bcrypt.GenerateFromPassword(append(password, salt...), bcrypt.DefaultCost)
		if err != nil {
			http.Error(w, err.Error(), http.StatusInternalServerError)
			return
		}
		id, err := queries.InsertUser(ctx, database.InsertUserParams{
			Name:         username,
			Hashpassword: password,
			Salt:         salt,
		})
		if err != nil {
			http.Error(w, err.Error(), http.StatusInternalServerError)
			return
		}
		fmt.Fprintf(w, "Inserted user with id %d\n", id)
	})
	http.HandleFunc("/oauth2/token", func(w http.ResponseWriter, r *http.Request) {
		if err := r.ParseForm(); err != nil {
			http.Error(w, err.Error(), http.StatusBadRequest)
			return
		}
		if r.Form.Get("grant_type") != "password" {
			http.Error(w, "grant_type must be password", http.StatusBadRequest)
			return
		}
		username := r.Form.Get("username")
		password := r.Form.Get("password")
		if username == "" || password == "" {
			http.Error(w, "username or password not set", http.StatusBadRequest)
			return
		}
		user, err := queries.GetCredentials(ctx, username)
		if err != nil {
			http.Error(w, err.Error(), http.StatusInternalServerError)
			return
		}
		if err := bcrypt.CompareHashAndPassword(user.Hashpassword, append([]byte(password), user.Salt...)); err != nil {
			http.Error(w, err.Error(), http.StatusUnauthorized)
			return
		}
		fmt.Fprint(w, "Successfully authenticated\n")
	})
}
```

Here we parse the username and password as form data instead of query parameters. 
This is part of the [OAuth2 Password Grant specification](https://www.oauth.com/oauth2-servers/access-tokens/password-grant).
Note that this is not the most secure way to authenticate users, but it is easy and simple
to set up compared to other authentication methods.

Then let us test out our new SecurityRoutes function.

```bash
~>curl -s -X 'POST' --data \
'username=another&password=secret&grant_type=password'\
-H 'Content-Type:application/x-www-form-urlencoded' \
'http://localhost:8080/oauth2/token'
Successfully authenticated
```

And then trying with an incorrect password.

```bash
~>curl -s -X 'POST' --data \
'username=another&password=wrongpassword&grant_type=password'\
-H 'Content-Type:application/x-www-form-urlencoded'\
'http://localhost:8080/oauth2/token'
crypto/bcrypt: hashedPassword is not the hash of the given password
```

## JWT Token

Finally, now that our user has signed in, we can generate a JWT token for them to use
to access protected routes in our API.

Lets first add a new module to our project by running `go get github.com/golang-jwt/jwt/v5`.
In order to sign our JWT tokens, we will need a secret key. Lets update our .env file with
`JWT_SECRET=secretkey`. 

```go {linenos=table,hl_lines=[6,"9-11","27-63","133-146"]}
package internal

import (
	"context"
	"crypto/rand"
	"encoding/json"
	"fmt"
	"net/http"
	"os"
	"strconv"
	"time"

	"golang.org/x/crypto/bcrypt"

	"github.com/acswindle/task-manager/database"
	"github.com/golang-jwt/jwt/v5"
)

func generateSalt() ([]byte, error) {
	salt := make([]byte, 16)
	if _, err := rand.Read(salt); err != nil {
		return nil, err
	}
	return salt, nil
}

type ResponseToken struct {
	Token   string `json:"access_token"`
	Type    string `json:"token_type"`
	Expires int    `json:"expires_in"`
}

func genrateJWT(username string) (ResponseToken, error) {
	jwtSecret, secretSet := os.LookupEnv("JWT_SECRET")
	if !secretSet {
		return ResponseToken{}, fmt.Errorf("JWT_SECRET not set")
	}
	jwtExpireTime, setExp := os.LookupEnv("JWT_EXPIRE_TIME")
	if !setExp {
		return ResponseToken{}, fmt.Errorf("JWT_EXPIRE_TIME not set")
	}
	expireTime, err := strconv.Atoi(jwtExpireTime)
	if err != nil {
		return ResponseToken{}, fmt.Errorf("JWT_EXPIRE_TIME must be an integer")
	}
	expireTime = expireTime * 3600
	claims := jwt.MapClaims{
		"username":   username,
		"exp":        time.Now().Add(time.Second * time.Duration(expireTime)).Unix(),
		"iat":        time.Now().Unix(),
		"authorized": true,
	}
	token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
	tokenString, err := token.SignedString([]byte(jwtSecret))
	if err != nil {
		return ResponseToken{}, fmt.Errorf("failed to generate JWT token: %v", err)
	}
	return ResponseToken{
		Token:   tokenString,
		Type:    "Bearer",
		Expires: expireTime,
	}, nil
}

func SecurityRoutes(ctx context.Context, queries *database.Queries) {
	// GET /users
	http.HandleFunc("GET /users", func(w http.ResponseWriter, r *http.Request) {
		users, err := queries.GetUsers(ctx)
		if err != nil {
			http.Error(w, err.Error(), http.StatusInternalServerError)
			return
		}
		fmt.Fprint(w, users)
	})
	// POST /user
	http.HandleFunc("POST /user", func(w http.ResponseWriter, r *http.Request) {
		username := r.URL.Query().Get("username")
		if username == "" {
			http.Error(w, "username not set", http.StatusBadRequest)
			return
		}
		rawPassword := r.URL.Query().Get("password")
		if rawPassword == "" {
			http.Error(w, "password not set", http.StatusBadRequest)
			return
		}
		password := []byte(rawPassword)
		salt, err := generateSalt()
		if err != nil {
			http.Error(w, err.Error(), http.StatusInternalServerError)
			return
		}
		password, err = bcrypt.GenerateFromPassword(append(password, salt...), bcrypt.DefaultCost)
		if err != nil {
			http.Error(w, err.Error(), http.StatusInternalServerError)
			return
		}
		id, err := queries.InsertUser(ctx, database.InsertUserParams{
			Name:         username,
			Hashpassword: password,
			Salt:         salt,
		})
		if err != nil {
			http.Error(w, err.Error(), http.StatusInternalServerError)
			return
		}
		fmt.Fprintf(w, "Inserted user with id %d\n", id)
	})
	http.HandleFunc("/oauth2/token", func(w http.ResponseWriter, r *http.Request) {
		if err := r.ParseForm(); err != nil {
			http.Error(w, err.Error(), http.StatusBadRequest)
			return
		}
		if r.Form.Get("grant_type") != "password" {
			http.Error(w, "grant_type must be password", http.StatusBadRequest)
			return
		}
		username := r.Form.Get("username")
		password := r.Form.Get("password")
		if username == "" || password == "" {
			http.Error(w, "username or password not set", http.StatusBadRequest)
			return
		}
		user, err := queries.GetCredentials(ctx, username)
		if err != nil {
			http.Error(w, err.Error(), http.StatusInternalServerError)
			return
		}
		if err := bcrypt.CompareHashAndPassword(user.Hashpassword, append([]byte(password), user.Salt...)); err != nil {
			http.Error(w, err.Error(), http.StatusUnauthorized)
			return
		}
		token, err := genrateJWT(username)
		if err != nil {
			http.Error(w, err.Error(), http.StatusInternalServerError)
			return
		}
		response, err := json.Marshal(token)
		if err != nil {
			http.Error(w, err.Error(), http.StatusInternalServerError)
			return
		}
		w.Header().Set("Content-Type", "application/json")
		w.Header().Set("Cache-Control", "no-store")
		w.Write(response)
	})
}
```

Our response token struct reflects what we respond back with on a successful login attempt.
Details of the spec can be found [here](https://www.oauth.com/oauth2-servers/access-tokens/access-token-response/).

The generateJWT function is responsible for generating the token and setting the expiration time.
We first read in our JWT_SECRET and JWT_EXPIRE_TIME environment variables.
Then we generate our claims, which we can later decifer when the client makes a request.
We then generate the token using the [HS256 algorithm](https://www.loginradius.com/blog/engineering/jwt-signing-algorithms/).
Finally, we sign the token with our jwt secret and return back the token.
Back in our handle function, we get the jwt Token, parse it into a json string,
and return it back to the user after setting the appropriate response headers.

## Validating Client Token

Now that we are able to authenticate and send our users a token, we need to validate any 
tokens that they may send back. 

Lets add validate token function to validate jwt tokens from our clients.

```go {linenos=table,hl_lines=["66-84","168-189"]}
package internal

import (
	"context"
	"crypto/rand"
	"encoding/json"
	"fmt"
	"net/http"
	"os"
	"strconv"
	"strings"
	"time"

	"golang.org/x/crypto/bcrypt"

	"github.com/acswindle/task-manager/database"
	"github.com/golang-jwt/jwt/v5"
)

func generateSalt() ([]byte, error) {
	salt := make([]byte, 16)
	if _, err := rand.Read(salt); err != nil {
		return nil, err
	}
	return salt, nil
}

type ResponseToken struct {
	Token   string `json:"access_token"`
	Type    string `json:"token_type"`
	Expires int    `json:"expires_in"`
}

func genrateJWT(username string) (ResponseToken, error) {
	jwtSecret, secretSet := os.LookupEnv("JWT_SECRET")
	if !secretSet {
		return ResponseToken{}, fmt.Errorf("JWT_SECRET not set")
	}
	jwtExpireTime, setExp := os.LookupEnv("JWT_EXPIRE_TIME")
	if !setExp {
		return ResponseToken{}, fmt.Errorf("JWT_EXPIRE_TIME not set")
	}
	expireTime, err := strconv.Atoi(jwtExpireTime)
	if err != nil {
		return ResponseToken{}, fmt.Errorf("JWT_EXPIRE_TIME must be an integer")
	}
	expireTime = expireTime * 3600
	claims := jwt.MapClaims{
		"username":   username,
		"exp":        time.Now().Add(time.Second * time.Duration(expireTime)).Unix(),
		"iat":        time.Now().Unix(),
		"authorized": true,
	}
	token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
	tokenString, err := token.SignedString([]byte(jwtSecret))
	if err != nil {
		return ResponseToken{}, fmt.Errorf("failed to generate JWT token: %v", err)
	}
	return ResponseToken{
		Token:   tokenString,
		Type:    "Bearer",
		Expires: expireTime,
	}, nil
}

func validateToken(token string) (string, error) {
	jwtSecret, secretSet := os.LookupEnv("JWT_SECRET")
	if !secretSet {
		return "", fmt.Errorf("JWT_SECRET not set")
	}
	tokenClaims, err := jwt.Parse(token, func(token *jwt.Token) (interface{}, error) {
		if _, ok := token.Method.(*jwt.SigningMethodHMAC); !ok {
			return nil, fmt.Errorf("unexpected signing method: %v", token.Header["alg"])
		}
		return []byte(jwtSecret), nil
	})
	if err != nil {
		return "", err
	}
	if claims, ok := tokenClaims.Claims.(jwt.MapClaims); ok && tokenClaims.Valid {
		return claims["username"].(string), nil
	}
	return "", fmt.Errorf("invalid token")
}

func SecurityRoutes(ctx context.Context, queries *database.Queries) {
	// GET /users
	http.HandleFunc("GET /users", func(w http.ResponseWriter, r *http.Request) {
		users, err := queries.GetUsers(ctx)
		if err != nil {
			http.Error(w, err.Error(), http.StatusInternalServerError)
			return
		}
		fmt.Fprint(w, users)
	})
	// POST /user
	http.HandleFunc("POST /user", func(w http.ResponseWriter, r *http.Request) {
		username := r.URL.Query().Get("username")
		if username == "" {
			http.Error(w, "username not set", http.StatusBadRequest)
			return
		}
		rawPassword := r.URL.Query().Get("password")
		if rawPassword == "" {
			http.Error(w, "password not set", http.StatusBadRequest)
			return
		}
		password := []byte(rawPassword)
		salt, err := generateSalt()
		if err != nil {
			http.Error(w, err.Error(), http.StatusInternalServerError)
			return
		}
		password, err = bcrypt.GenerateFromPassword(append(password, salt...), bcrypt.DefaultCost)
		if err != nil {
			http.Error(w, err.Error(), http.StatusInternalServerError)
			return
		}
		id, err := queries.InsertUser(ctx, database.InsertUserParams{
			Name:         username,
			Hashpassword: password,
			Salt:         salt,
		})
		if err != nil {
			http.Error(w, err.Error(), http.StatusInternalServerError)
			return
		}
		fmt.Fprintf(w, "Inserted user with id %d\n", id)
	})
	http.HandleFunc("/oauth2/token", func(w http.ResponseWriter, r *http.Request) {
		if err := r.ParseForm(); err != nil {
			http.Error(w, err.Error(), http.StatusBadRequest)
			return
		}
		if r.Form.Get("grant_type") != "password" {
			http.Error(w, "grant_type must be password", http.StatusBadRequest)
			return
		}
		username := r.Form.Get("username")
		password := r.Form.Get("password")
		if username == "" || password == "" {
			http.Error(w, "username or password not set", http.StatusBadRequest)
			return
		}
		user, err := queries.GetCredentials(ctx, username)
		if err != nil {
			http.Error(w, err.Error(), http.StatusInternalServerError)
			return
		}
		if err := bcrypt.CompareHashAndPassword(user.Hashpassword, append([]byte(password), user.Salt...)); err != nil {
			http.Error(w, err.Error(), http.StatusUnauthorized)
			return
		}
		token, err := genrateJWT(username)
		if err != nil {
			http.Error(w, err.Error(), http.StatusInternalServerError)
			return
		}
		response, err := json.Marshal(token)
		if err != nil {
			http.Error(w, err.Error(), http.StatusInternalServerError)
			return
		}
		w.Header().Set("Content-Type", "application/json")
		w.Header().Set("Cache-Control", "no-store")
		w.Write(response)
	})
	http.HandleFunc("GET /validate", func(w http.ResponseWriter, r *http.Request) {
		auth := r.Header.Get("Authorization")
		if auth == "" {
			http.Error(w, "token not set", http.StatusUnauthorized)
			return
		}
		token := strings.Split(auth, " ")
		if token[0] != "Bearer" {
			http.Error(w, "token not set", http.StatusUnauthorized)
			return
		}
		if len(token) != 2 {
			http.Error(w, "token not set", http.StatusUnauthorized)
			return
		}
		username, err := validateToken(token[1])
		if err != nil {
			http.Error(w, err.Error(), http.StatusUnauthorized)
			return
		}
		fmt.Fprint(w, username)
	})
}
```

The validateToken function is nearly identical to the [example](https://pkg.go.dev/github.com/golang-jwt/jwt/v5#example-Parse-Hmac)
given in the official package documentation. 
We create a new route called /validate to test out our token validation.
We expect the client to send the token in the Authorization header.
It should be of the form `Bearer <token>`.
So we split the token from the header and check that it is of the correct format.
Then we validate the token and return the username back to the client.

Lets test it out.

```bash
~>curl -s -X 'POST' --data 'username=testuser&password=secret&grant_type=password' -H 'Content-Type:application/x-www-form-urlencoded' 'http://localhost:8080/oauth2/token'
{"access_token":"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJhdXRob3JpemVkIjp0cnVlLCJleHAiOjE3Mzk3NzM1NTksImlhdCI6MTczOTc2NjM1OSwidXNlcm5hbWUiOiJ0ZXN0dXNlciJ9.f5NHCJdKb_jdlsDJyYL0DsyrzOt9evHTzAL4fFk6BgM","token_type":"Bearer","expires_in":7200}
~>curl -s -X 'GET' -H 'Authorization:Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJhdXRob3JpemVkIjp0cnVlLCJleHAiOjE3Mzk3NzM0ODIsImlhdCI6MTczOTc2NjI4MiwidXNlcm5hbWUiOiJ0ZXN0dXNlciJ9.Mxp0hOEx6jNMPZ29xZa9vSXrgFYEIv5Q5dEIbWA75Mo' 'http://localhost:8080/validate'
testuser
```

And it appears that we are authenticated.

## Conclusion

Here we implemented a simple Oauth2 Password Grant flow. In the next article 
we will go through and implement some of the remaining CRUD operations listed
in the requirements of the [project](https://roadmap.sh/projects/expense-tracker-api).
See you next time! 🍻
