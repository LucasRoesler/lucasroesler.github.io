---
title: "When sequential tests aren't"
summary: "A brief reminder that go will default to testing packages in parallel"
date: 2019-02-02T00:00:00+02:00
tags:
  - programming
  - golang
  - testing
coverImage: dominos.jpg
---

At work we have been expanding the testing coverage of our auth service and have discovered the perfect storm of edge cases around test caching and Go's test building. In short, we had tests that looked like they were passing locally but would fail sporadically in CI because Go will run the tests inside of a package sequentially but build and test each package in parallel (up to the number of CPU cores).

---

## Background

For various reasons we have a package called `server` that looks roughly like this

```
server
├── server.go
├── bootstrap.go
├── bootstrap_test.go
├── tenant
│   ├── db.go
│   ├── service.go
│   ├── service_test.go
│   ├── testutils.go
│   └── validation.go
└── user
    ├── db.go
    ├── password.go
    ├── service.go
    ├── service_test.go
    └── validation.go
```

Each of those test files wants to connect to the DB and will run some queries. To enable running individual tests, we also have a helper method for initializing the database

```go
var (
	testDatabase *sql.DB
	err          error
	once         sync.Once
)

// GetDatabase gets a test database
func GetDatabase(t *testing.T) *sql.DB {
	once.Do(func() {
		testDatabase, err = connectDB()
		if err != nil {
			return
		}
		ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
		defer cancel()

		err = db.SetupTables(ctx, testDatabase)
		if err != nil {
			return
		}
	})

	if err != nil {
		t.Fatalf("Can't initialize the database: %v", err)
		return nil
	}
	return testDatabase
}
```

## The bug

Using `sync.Once` ensures that the database setup method runs exactly once. This works but with a big caveat, when you run

```sh
go test ./...
```

Go builds and runs binaries for `tenant/service_test.go`, `user/service_test.go`, and `bootstrap_test.go` in parallel. When they run in parallel at the same time we would see a sql error

```sql
pq: duplicate key value violates unique constraint "pg_extension_name_index"
```

which is caused by running `CREATE EXTENSION IF NOT EXISTS foo` multiple times simultaneously.

At first, I couldn't determine why. I thought all of the tests were run sequentially. But, after digging through the documentation for `go test` and then `go build`, [I found][go-build-docs]

```
	-p n
		the number of programs, such as build commands or
		test binaries, that can be run in parallel.
		The default is the number of CPUs available.
```

This points to the quick fix, but I was still confused why it would mostly succeed when run locally but usually fail in CI. [CACHING!][go-1-10-notes]. In Go 1.10

> The go test command now caches test results

Due to random timing, sometimes one of those test packages would succeed, the result cached, and then on subsequent test runs, skipped. Which would reduce the number of calls to `GetDatabase` and allow all of the tests to pass. This can be disabled by passing `-count=1` to your test command.

## The fix(es)

Having read the docs and disable the cache, I was able to repoduce and then "fix" the issue

1. _The quick fix to get CI passing_: the most obvious way to resolve this issue is to not allow parallel test binaries, using

   ```sh
   go test -p=1 ./...
   ```

   Now each package and test is run exactly in the order that it is printed, nothing in parallel.

2. _A slightly better fix_: Create a new database per test package. In the initial `GetDatabase` method the same database connection is used over and over. In particular, the same database itself is used. If we modify the method to allow us to pass a new db name, then we can instantiate a fresh database per package and allows us to drop the `-p=1` flag.

   ```go
   // GetDatabase gets a test database
   func GetDatabase(t *testing.T, name string) *sql.DB {
       once.Do(func() {
           testDatabase, err = connectDB(name)
           if err != nil {
               return
           }
           ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
           defer cancel()

           err = db.SetupTables(ctx, testDatabase)
           if err != nil {
               return
           }
       })

       if err != nil {
           t.Fatalf("Can't initialize the database: %v", err)
           return nil
       }
       return testDatabase
   }

   func connectDB(name string) (db *sql.DB, err error) {
       adminConnStr := fmt.Sprintf(
           "dbname=%s user=%s sslmode=disable host=0.0.0.0 password=%s",
           databaseName,
           databaseUser,
           databasePassword,
       )

       adminDB, err := sql.Open("postgres", adminConnStr)
       if err != nil {
           return nil, err
       }
       defer adminDB.Close()
       _, err = adminDB.Exec(fmt.Sprintf("CREATE DATABASE %s OWNER %s", name, databaseUser))
       if err != nil {
           return nil, err
       }

       connStr := fmt.Sprintf(
           "dbname=%s user=%s sslmode=disable host=0.0.0.0 password=%s",
           name,
           databaseUser,
           databasePassword,
       )

       return sql.Open("postgres", connStr)
   }
   ```

3. _The "correct" solution_: separate the datalayer into its own package. Any code that contains SQL or needs to actually access the DB should live in a single package and expose an interface that we can import into the server. If only one package needs to access the db, then there is only one call to setup the db! This also provides various other great benefits. For various, not very good, reasons we didn't do this immediately from the start. This requires a bit more effort to resolve. (Although some other very smart decisions we made will make this refactor much more simple than it seems.)

## Wrapping up

I love Go's test caching, but it definitely bit me this week. Fortunately, it only required a small tweak to get tests up and running consistently.

[go-1-10-notes]: https://golang.org/doc/go1.10#test
[go-build-docs]: https://golang.org/cmd/go/#hdr-Compile_packages_and_dependencies
