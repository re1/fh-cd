# Go Microservice

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
