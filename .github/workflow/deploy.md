name: CI/CD

on:
  push:
    branches:
      - main

jobs:
  lint:
    name: Lint Go Code
    runs-on: self-hosted

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Set up Go
        uses: actions/setup-go@v5
        with:
          go-version: '1.26'

      - name: Download dependencies
        run: go mod download

      - name: Check formatting
        run: test -z "$(gofmt -l .)"

      - name: Run go vet
        run: go vet ./...

  build:
    name: Build Go App
    runs-on: self-hosted
    needs: lint

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Set up Go
        uses: actions/setup-go@v5
        with:
          go-version: '1.24'

      - name: Download dependencies
        run: go mod download

      - name: Build application
        run: go build -o app .

  docker:
    name: Build Docker Image
    runs-on: self-hosted
    needs: build

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Build Docker image
        run: docker build -t simple-go-api .

  deploy:
    name: Deploy App
    runs-on: self-hosted
    needs: docker

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Stop old container
        run: docker rm -f simple-go-api || true

      - name: Run new container
        run: docker run -d --name simple-go-api --restart always -p 8080:8080 --env-file .env simple-go-api