---
title: "Building a REST API in Go from Scratch"
date: 2025-01-15
tags: ["golang", "backend", "api"]
categories: ["Engineering"]
description: "A step-by-step guide to building a REST API using Go's net/http package."
showToc: true
draft: false
cover:
  image: /images/covers/go-api.png
  alt: "Go API diagram"
  caption: "Go REST API architecture"
---

Go's standard library is one of its greatest strengths. For many REST API use cases you don't need a framework at all — `net/http` combined with the new `ServeMux` introduced in Go 1.22 gives you everything you need. In this post, we'll build a small but production-ready API from scratch.

## Project Structure

A clean project layout makes the codebase easier to navigate as it grows. Here's the layout we'll use:

```
myapi/
├── cmd/
│   └── server/
│       └── main.go
├── internal/
│   ├── handler/
│   │   └── user.go
│   └── store/
│       └── user.go
├── go.mod
└── go.sum
```

The `cmd/` directory holds the entry point. Business logic lives under `internal/`, which is not importable by external packages — a nice Go convention.

## Defining the Router

Go 1.22 enhanced `net/http`'s `ServeMux` to support method-scoped and path-parameterised routes natively:

```go
package main

import (
    "log"
    "net/http"

    "myapi/internal/handler"
)

func main() {
    mux := http.NewServeMux()

    userHandler := handler.NewUserHandler()

    mux.HandleFunc("GET /users", userHandler.List)
    mux.HandleFunc("POST /users", userHandler.Create)
    mux.HandleFunc("GET /users/{id}", userHandler.Get)
    mux.HandleFunc("PUT /users/{id}", userHandler.Update)
    mux.HandleFunc("DELETE /users/{id}", userHandler.Delete)

    log.Println("Server listening on :8080")
    log.Fatal(http.ListenAndServe(":8080", mux))
}
```

## Writing a Handler

Each handler receives a `ResponseWriter` and a `*Request`. We decode JSON from the request body and encode our response:

```go
package handler

import (
    "encoding/json"
    "net/http"
)

type UserHandler struct{}

func NewUserHandler() *UserHandler {
    return &UserHandler{}
}

type User struct {
    ID    int    `json:"id"`
    Name  string `json:"name"`
    Email string `json:"email"`
}

func (h *UserHandler) Create(w http.ResponseWriter, r *http.Request) {
    var u User
    if err := json.NewDecoder(r.Body).Decode(&u); err != nil {
        http.Error(w, "invalid request body", http.StatusBadRequest)
        return
    }

    // TODO: persist user to store
    u.ID = 1 // placeholder

    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusCreated)
    json.NewEncoder(w).Encode(u)
}
```

## Middleware for Logging

Middleware in Go is simply a function that wraps an `http.Handler`. Here's a request logger:

```go
func LoggingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        log.Printf("%s %s", r.Method, r.URL.Path)
        next.ServeHTTP(w, r)
    })
}
```

Wrap your mux when starting the server:

```go
log.Fatal(http.ListenAndServe(":8080", LoggingMiddleware(mux)))
```

## Error Handling Strategy

A consistent error response format makes life easier for API consumers. Define a small helper:

```go
type APIError struct {
    Code    int    `json:"code"`
    Message string `json:"message"`
}

func writeError(w http.ResponseWriter, code int, msg string) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(code)
    json.NewEncoder(w).Encode(APIError{Code: code, Message: msg})
}
```

## Testing with curl

With the server running (`go run ./cmd/server`), test your endpoints:

```bash
# Create a user
curl -s -X POST http://localhost:8080/users \
  -H "Content-Type: application/json" \
  -d '{"name":"Alice","email":"alice@example.com"}' | jq

# List users
curl -s http://localhost:8080/users | jq
```

## Next Steps

This covers the skeleton. In follow-up posts I'll add PostgreSQL persistence using `pgx`, JWT authentication middleware, graceful shutdown, and a Docker setup for local development. Stay tuned!
