---
title: "How Go helped for the rewrite of pg_back"
date: 2021-03-10T22:00:00+01:00
categories:
- Code
tags:
- go
- postgresql
- pg_back
---


When I started developping [pg_back](https://github.com/orgrim/pg_back), I
wanted to keep it a simple as possible, by limiting its features. It also
limited contributions, and at some point by accepting some useful
contributions, the script grew and became not so simple. We could go on with
the shell script and try to carefully control its growth, but it will sooner or
later reach some limits of shell scripting.

After some time without making it evolve, I got interested in the Go
programming language. It appeared that pg_back could be a good candidate projet
to let me learn Go. This is why pg_back version 2 is written in Go.

<!--more-->

Go can do better than shell scripting, like many languages, but one helpful
feature of Go is that it produces static binaries. This make it even easier to
install than the shell script, because `pg_back` no longer has any external
dependencies, apart from `pg_dump` and `pg_dumpall`. When one does not have
root privileges, it make it possible be deploy the binary file by dropping it
somewhere like one would do for a shell script.

About the rewrite, getting rid of dependencies on external command is
great. For example, the syntax of Go being close to C, it was quite easy to
port `pg_dumpacl` and integrate it directly in pg_back. It is now just two
function `dumpCreateDBAndACL()` and `makeACLCommands()` in
[sql.go](https://github.com/orgrim/pg_back/blob/master/sql.go).

Continuing on external commands, pg_back now connects to PostgreSQL without
using `psql`, with the native `database/sql` API. The standard library is
pretty rich, it allowed the use standard Go APIs for checksums, file locking,
logging, etc. Only 4 external modules are required, plus one for unit tests:

* `gopkg.in/ini.v1` for the configuration file
* `github.com/spf13/pflag` for POSIX command line options
* `github.com/lib/pq` to access PostgreSQL
* `github.com/anmitsu/go-shlex` to parse shell commands (mostly hooks)

The other module is `github.com/google/go-cmp`, with is useful to compare
structs in unit tests. Speaking of which, unit tests are native to Go, and it
was easy to add tests while developping. The difficult part of unit testing is
related to the replication control functions. Unit testing the replication
control fonctions is difficult because it requires a more complex setup. We
need a PostgreSQL instance with a replica.

This feature is interesting, because I could use some features of Go to
implement pausing the replication.

When dumping from a replica, we need to ensure pg_dump won't be stuck on an
exclusive lock, we must pause the replication replay to avoid pg_dump being
cancelled and there must not be any exclusive lock granted on an object when
the replication is paused. Otherwise the exclusive lock is only released when
replication replay is resumed, making the dump of the object impossible.

I used a go routine with a `time.Ticker` to try to pause the replication every
10 seconds:


``` go
func pauseReplicationWithTimeout(db *pg, timeOut int) error {
    // ...
    ticker := time.NewTicker(time.Duration(10) * time.Second)
    done := make(chan bool)
    stop := make(chan bool)
    fail := make(chan error)

    l.Infoln("pausing replication")

    // We want to retry pausing replication at a defined interval
    // but not forever. We cannot put the timeout in the same
    // select as the ticker since the ticker would always win
    go func() {
        var rerr *pgReplicaHasLocks
        defer ticker.Stop()

        for {
            if err := pauseReplication(db); err != nil {
                if errors.As(err, &rerr) {
                    l.Warnln(err)
                } else {
                    fail <- err
                    return
                }
            } else {
                done <- true
                return
            }

            select {
            case <-stop:
                return
            case <-ticker.C:
                break
            }
        }
    }()

    // ...
}
```

When there is an exclusive lock, a custom error is used to inform the goroutine
that it must retry:

``` go
type pgReplicaHasLocks struct{}

func (*pgReplicaHasLocks) Error() string {
    return "replication not paused because of AccessExclusiveLock"
}

func pauseReplication(db *pg) error {
    // ..
    if void == "failed" {
        return &pgReplicaHasLocks{}
    }
    return nil
}
```

The main goroutine just wait on some channel until there is a timeout using
`time.After()`, replication has been paused or an error occured:

``` go
    // ... (After lauching the goroutine)

    // Return as soon as the replication is paused or stop the
    // goroutine if we hit the timeout
    select {
    case <-done:
        l.Infoln("replication paused")
    case <-time.After(time.Duration(timeOut) * time.Second):
        stop <- true
        return fmt.Errorf("replication not paused after %v", time.Duration(timeOut)*time.Second)
    case err := <-fail:
        return fmt.Errorf("%s", err)
    }
    
    // ...
```

See the complete function in [sql.go#L509-L629](https://github.com/orgrim/pg_back/blob/master/sql.go#L509-L629).

Another use of goroutines was to run `pg_dump` commands concurrently, in order to dump many
database at the same time. It uses the worker example from
[gobyexample.com](https://gobyexample.com/worker-pools), a great ressource for
beginners, by the way!

The other problems with shell scripting were about parsing strings, like
`keyword=value` connection strings of PostgreSQL
([connstring.go](https://github.com/orgrim/pg_back/blob/master/connstring.go)). Shell
scripting is notoriously difficult when it comes to handling strings properly,
and Go helped a lot. It was happily surprised that the Go implementation of pg_back
could dump databases with weird characters in their names, like the equal sign,
spaces, quotes, etc. without a lot of extra work.

Finally, the embed feature of Go 1.16 is so easy to use that embeding the
default configuration file, including examples and comments, and adding a
command line option to print it took a just few lines of code:

``` go
import (
    // ...
    _ "embed"
    // ...
)

//go:embed pg_back.conf
var defaultCfg string

func parseCli(args []string) (options, []string, error) {
    // ...
    
    pflag.BoolVar(&pce.ShowConfig, "print-default-config", false, "print the default configuration\n")
    
    if pce.ShowConfig {
        fmt.Print(defaultCfg)
        // ...
    }
    
    // ...
}
```

This will make the feature easier to maintain in the future because it avoids
duplication, as I would have had to copy the file in a string in the source
code without `embed`.

In conclusion, rewriting pg_back in a proper programming language with a rich
standard library and ecosystem was great. I enjoyed doing it with Go and it
convinced me that Go is programming language worth learning.
