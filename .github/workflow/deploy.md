name: CI/CD

on:
  push:
    branches:
      - main

jobs:
  lint:
    runs-on: self-hosted
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version: '1.26'
      - run: go mod download
      - run: test -z "$(gofmt -l .)"
      - run: go vet ./...

  build:
    runs-on: self-hosted
    needs: lint
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version: '1.26'
      - run: go mod download
      - run: go build -o app .

  docker:
    runs-on: self-hosted
    needs: build
    steps:
      - uses: actions/checkout@v4
      - run: docker build -t go .

  deploy:
    runs-on: self-hosted
    needs: docker
    steps:
      - uses: actions/checkout@v4
      - run: docker rm -f go || true
      - run: |
          docker run -d --name go --restart always -p 8080:8080 \
            -e DB_HOST=${{ secrets.DB_HOST }} \
            -e DB_PORT=${{ secrets.DB_PORT }} \
            -e DB_USER=${{ secrets.DB_USER }} \
            -e DB_PASSWORD=${{ secrets.DB_PASSWORD }} \
            -e DB_NAME=${{ secrets.DB_NAME }} \
            -e DB_SSLMODE=${{ secrets.DB_SSLMODE }} \
            go