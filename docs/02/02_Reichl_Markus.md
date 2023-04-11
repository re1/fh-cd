---
author: Markus Reichl
reproducible:
source-highlighter: rouge
rouge-style: github
---

# Microservice in Go

This application is based on the [tutorial](https://semaphoreci.com/community/tutorials/building-and-testing-a-rest-api-in-go-with-gorilla-mux-and-postgresql) found in the exercise description.

## Setup

The application requires a running PostgreSQL database. The easiest way to get one is to use Docker:

```bash
docker run -it -p 5432:5432 -d postgres
```

In order to connect to this database, the application needs to know the database name, username and password. These can be set using environment variables:

```bash
export APP_DB_USERNAME=postgres
export APP_DB_PASSWORD=
export APP_DB_NAME=postgres
```

## Overview

This Go application implements a RESTful API to interact with a PostgreSQL database. The main file is `app.go` which contains the implementation of the App struct with the main functions to run the server and to interact with the database. The file defines the HTTP endpoints that allow creating, reading, updating, and deleting products.

### `main.go`

This is where the application starts executing. The `main` function creates an instance of an App struct, initializes it with database credentials obtained from environment variables using `os.Getenv()`, and then starts the application on port 8010 using the `Run()` method of the App struct.

### `app.go`

The application uses the Gorilla Mux package to handle the routing of the HTTP requests. The code creates a new instance of a Router, initializes a database connection in the Initialize method, and sets up the routes in the initializeRoutes method. It also defines functions for responding to requests with error messages and with JSON data.

### `model.go`

The model provides functions to interact with the database. The functions are called by the HTTP handlers in `app.go`. The functions are implemented using the `database/sql` package.

### `main_test.go`

This file contains the tests for the application. The tests use the `httptest` package to create a test server and send HTTP requests to the endpoints.

## Additional features

### Health check

A simple feature to implement is to check if the server is up and running. I added a function for this returning `{"status": "ok"}`.

```go
func (a *App) healthCheck(w http.ResponseWriter, r *http.Request) {
    if err := a.DB.Ping(); err != nil {
        respondWithError(w, http.StatusInternalServerError, "Database connection error")
        return
    }

    respondWithJSON(w, http.StatusOK, map[string]string{"status": "ok"})
}
```

The feature was added as an endpoint in `initializeRoutes` like this:

```go
a.Router.HandleFunc("/health", a.healthCheck).Methods("GET")
```

The test creates a new HTTP request to the /health endpoint and checks that the response code is 200 (OK). It then unmarshals the response body, which should be a JSON object containing a single key-value pair, {"status": "ok"}, and checks that the value of the "status" key is "ok". If the value is anything else, the test fails.

```go
func TestHealthCheck(t *testing.T) {
    req, _ := http.NewRequest("GET", "/health", nil)
    response := executeRequest(req)

    checkResponseCode(t, http.StatusOK, response.Code)

    var m map[string]string
    json.Unmarshal(response.Body.Bytes(), &m)

    if m["status"] != "ok" {
        t.Errorf("Expected the 'status' key of the response to be set to 'ok'. Got '%s'", m["status"])
    }
}
```

### Products by name

This feature is an HTTP API endpoint that allows a client to retrieve a list of products from the server based on a substring of the product name. The function extracts the "name" parameter value from the request URL and passes it to the model function.

```go
func (a *App) getProductByNameSubstring(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    name := vars["name"]

    products, err := getProductsByNameSubstring(a.DB, name)
    if err != nil {
        respondWithError(w, http.StatusInternalServerError, err.Error())
        return
    }

    respondWithJSON(w, http.StatusOK, products)
}
```

Inside the model, the function uses the `LIKE` operator to find all products whose name contains the substring.

```go
func getProductsByNameSubstring(db *sql.DB, nameSubstring string) ([]product, error) {
    rows, err := db.Query("SELECT * FROM products WHERE name LIKE '%' || $1 || '%'", nameSubstring)
    if err != nil {
        return nil, err
    }
    defer rows.Close()

    var products []product
    for rows.Next() {
        var p product
        if err := rows.Scan(&p.ID, &p.Name, &p.Price); err != nil {
            return nil, err
        }
        products = append(products, p)
    }
    if err := rows.Err(); err != nil {
        return nil, err
    }
    return products, nil
}
```

The feature was added as an endpoint in `initializeRoutes` like this:

```go
a.Router.HandleFunc("/products/search", a.getProductByNameSubstring)
    .Methods("GET")
    .Queries("name", "{name}")
```

This test is testing the `getProductByNameSubstring` function which retrieves products by a name substring. The test first clears the table of products and adds 3 new products to the table. Then, it sends a GET request to the `/products/search` endpoint with the query parameter name set to "Product 1". The response is checked to ensure that the response code is http.StatusOK. The response body is then unmarshaled into a slice of maps, with each map representing a product. The test checks that only one product was returned and that the product has the name "Product 1".

```go
func TestGetProductByNameSubstring(t *testing.T) {
    clearTable()
    addProducts(3)

    req, _ := http.NewRequest("GET", "/products/search?name=Product+1", nil)
    response := executeRequest(req)

    checkResponseCode(t, http.StatusOK, response.Code)

    var m []map[string]interface{}
    json.Unmarshal(response.Body.Bytes(), &m)

    if len(m) != 1 {
        t.Errorf("Expected one product. Got %d", len(m))
    }

    if m[0]["name"] != "Product 1" {
        t.Errorf("Expected product with name 'Product 1'. Got '%v'", m[0]["name"])
    }
}
```
