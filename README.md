file-rotatelogs
==================

Provide an `io.Writer` that periodically rotates log files from within the application. Port of [File::RotateLogs](https://metacpan.org/release/File-RotateLogs) from Perl to Go.

[![Build Status](https://travis-ci.org/lestrrat-go/file-rotatelogs.png?branch=master)](https://travis-ci.org/lestrrat-go/file-rotatelogs)

[![GoDoc](https://godoc.org/github.com/lestrrat-go/file-rotatelogs?status.svg)](https://godoc.org/github.com/lestrrat-go/file-rotatelogs)


# SYNOPSIS

```go
import (
  "log"
  "net/http"

  apachelog "github.com/lestrrat-go/apache-logformat"
  rotatelogs "github.com/lestrrat-go/file-rotatelogs"
)

func main() {
  mux := http.NewServeMux()
  mux.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) { ... })

  logf, err := rotatelogs.New(
    "/path/to/access_log.%Y%m%d%H%M",
    rotatelogs.WithLinkName("/path/to/access_log"),
    rotatelogs.WithMaxAge(24 * time.Hour),
    rotatelogs.WithRotationTime(time.Hour),
  )
  if err != nil {
    log.Printf("failed to create rotatelogs: %s", err)
    return
  }

  // Now you must write to logf. apache-logformat library can create
  // a http.Handler that only writes the approriate logs for the request
  // to the given handle
  http.ListenAndServe(":8080", apachelog.Wrap(mux, logf))
}
```

# DESCRIPTION

When you integrate this to to you app, it automatically write to logs that
are rotated from within the app: No more disk-full alerts because you forgot
to setup logrotate!

To install, simply issue a `go get`:

```
go get github.com/lestrrat-go/file-rotatelogs
```

It's normally expected that this library is used with some other
logging service, such as the built-in `log` library, or loggers
such as `github.com/lestrrat-go/apache-logformat`.

```go
import(
  "log"
  "github.com/lestrrat-go/file-rotatelogs"
)
  
func main() {
  rl, _ := rotatelogs.New("/path/to/access_log.%Y%m%d%H%M")

  log.SetOutput(rl)

  /* elsewhere ... */
  log.Printf("Hello, World!")
}
```

OPTIONS
====

## Pattern (Required)

The pattern used to generate actual log file names. You should use patterns
using the strftime (3) format. For example:

```go
  rotatelogs.New("/var/log/myapp/log.%Y%m%d")
```

## Clock (default: rotatelogs.Local)

You may specify an object that implements the roatatelogs.Clock interface.
When this option is supplied, it's used to determine the current time to
base all of the calculations on. For example, if you want to base your
calculations in UTC, you may specify rotatelogs.UTC

```go
  rotatelogs.New(
    "/var/log/myapp/log.%Y%m%d",
    rotatelogs.WithClock(rotatelogs.UTC),
  )
```

## Location

This is an alternative to the `WithClock` option. Instead of providing an
explicit clock, you can provide a location for you times. We will create
a Clock object that produces times in your specified location, and configure
the rotatelog to respect it.

## LinkName (default: "")

Path where a symlink for the actual log file is placed. This allows you to 
always check at the same location for log files even if the logs were rotated

```go
  rotatelogs.New(
    "/var/log/myapp/log.%Y%m%d",
    rotatelogs.WithLinkName("/var/log/myapp/current"),
  )
```

```
  // Else where
  $ tail -f /var/log/myapp/current
```

If not provided, no link will be written.

## RotationTime (default: 86400 sec)

Interval between file rotation. By default logs are rotated every 86400 seconds.
Note: Remember to use time.Duration values.

```go
  // Rotate every hour
  rotatelogs.New(
    "/var/log/myapp/log.%Y%m%d",
    rotatelogs.WithRotationTime(time.Hour),
  )
```

Please be aware that the timestamps used in the file names are truncated according
to the rotation time value. For example if you rotate daily, please do not expect
to have your filenames to contain values for hours, minutes and so forth. i.e.
the following is doesn't do what you want:

```
  rotatelogs.New("/var/logs/myapp/log.%Y%m%d%H%M%S.log")
  // file names will NEVER contain %H%M%S values
```

This is because when we check if we should be rotating the files, we only check
if we can create a different file name compared to the previous generation.
So if you started logging on Aug 1 2019 23:59:59, you might be writing to the file
`20190801.log`, but then one minute later, you will be writing to `20190802.log`
because it's already Aug 2, 2019 00:00:00.

So in that regard, `RotationTime` is a misnomer. We don't actually _wait_ for
the specified amount of time before rotating. Instead, we keep the same file
name based on the current time normalized by the value of `RotationTime`

```
  // -> 2019080101.log, 2019080102.log, ... etc
  rotatelogs.New("%Y%m%d%H.log", rotatelogs.WithRotationTime(time.Hour))

  // -> 201908010101.log, 201908010102.log, ... etc
  rotatelogs.New("%Y%m%d%H%M.log", rotatelogs.WithRotationTime(time.minute))

  // -> 20190800.log 20190802.log, 20190804.log ...
  rotatelogs.New("%Y%m%d%H.log", rotatelogs.WithRotationTime(2*time.Hour))
```

## MaxAge (default: 7 days)

Time to wait until old logs are purged. By default no logs are purged, which
certainly isn't what you want.
Note: Remember to use time.Duration values.

```go
  // Purge logs older than 1 hour
  rotatelogs.New(
    "/var/log/myapp/log.%Y%m%d",
    rotatelogs.WithMaxAge(time.Hour),
  )
```

## RotationCount (default: -1)

The number of files should be kept. By default, this option is disabled.

Note: MaxAge should be disabled by specifing `WithMaxAge(-1)` explicitly.

```go
  // Purge logs except latest 7 files
  rotatelogs.New(
    "/var/log/myapp/log.%Y%m%d",
    rotatelogs.WithMaxAge(-1),
    rotatelogs.WithRotationCount(7),
  )
```

## Handler (default: nil)

Sets the event handler to receive event notifications from the RotateLogs
object. Currently only supported event type is FiledRotated

```go
  rotatelogs.New(
    "/var/log/myapp/log.%Y%m%d",
    rotatelogs.Handler(rotatelogs.HandlerFunc(func(e Event) {
      if e.Type() != rotatelogs.FileRotatedEventType {
        return
      }

      // Do what you want with the data. This is just an idea:
      storeLogFileToRemoteStorage(e.(*FileRotatedEvent).PreviousFile())
    })),
  )
```

# Rotating files forcefully

If you want to rotate files forcefully before the actual rotation time has reached,
you may use the `Rotate()` method. This method forcefully rotates the logs, but
if the generated file name clashes, then a numeric suffix is added so that
the new file will forcefully appear on disk.

For example, suppose you had a pattern of '%Y.log' with a rotation time of
`86400` so that it only gets rotated every year, but for whatever reason you
wanted to rotate the logs now, you could install a signal handler to
trigger this rotation:

```go
rl := rotatelogs.New(...)

signal.Notify(ch, syscall.SIGHUP)

go func(ch chan os.Signal) {
  <-ch
  rl.Rotate()
}()
```

And you will get a log file name in like `2018.log.1`, `2018.log.2`, etc.
