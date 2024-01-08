---
title: 'Dependency Injection with samber/do'
description: 'Automatically wire up your Go code without using reflection'
pubDate: 'Jan 08 2024'
heroImage: '/Go_Logo_LightBlue.png'
---

## Simple idea, fancy terminology

*Dependency injection* is a $50 term for a $5 concept.
In a nutshell, it means **having the caller of a function provide the function’s dependencies** instead of making the function fetch its dependencies on its own.

Rather than using cars and engines or espresso machines and coffee beans, let’s use a realistic example to demonstrate the idea:

```go
// CountNewlines returns the number of newline characters in the given file.
// It returns an error if the file cannot be opened or cannot be read.
func CountNewlines(filename string) (int, error) {
  file, err := os.Open(filename)
  if err != nil {
    slog.Error("unable to open file", "file", filename)
    return 0, err
  }
  defer file.Close()

  contents, err := io.ReadAll(file)
  if err != nil {
    return 0, err
  }

  return bytes.Count(contents, []byte("\n")), nil
}
```

Testing the above implementation is unwieldy because any test suite has to write fixtures to a specific file before passing the filename to `CountNewlines`.
Furthermore, some CI environments restrict what folders test code may write to and how much data can be written.

What if there were a way to test `CountNewlines` without dealing with folder permissions or doing any disk I/O?

```go
func CountNewlines(contents io.Reader) (int, error) {
  data, err := io.ReadAll(contents)
  if err != nil {
    return 0, err
  }

  return bytes.Count(data, []byte("\n")), nil
}

func main() {
  file, err := os.Open("data.csv")
  if err != nil {
    slog.Error("unable to open file", "error", err)
  }
  defer file.Close()

  n, err := CountNewlines(file)
  // ...
}
```

`CountNewlines` no longer has to muck around with file handles.
That responsibility has been moved to the caller.

We can now use an [in-memory data store](https://github.com/spf13/afero) that implements the `io.Reader` interface for unit tests and use actual files in production.

That’s it. That’s all dependency injection is.

## There’s always a framework

Well, what you saw above is the technical definition of dependency injection (DI).
In practice, the term implicitly includes the idea of a dependency injection framework.

**A dependency injection framework automatically connects functions with their dependencies**. If you have fifteen functions that require a logger and a database client as arguments, you can simply tell the framework about the functions and give it the logger and database client once. The framework will wire them up automatically.

[samber/do](https://github.com/samber/do) is one such DI framework. Some points in its favour:

- does not use reflection, which means it’s very fast
- supports generics
- easy to debug

Let’s build a simple application using *samber/do*.

## The magic box

At the core of any DI framework is the container. You can think of this container as a magic box in which you put every function your application needs. The container is smart enough to call functions in the right order and put returned values in the right places when calling other functions.

Here is a trivial application:

```go
package main

import "github.com/samber/do"

func main() {
  container := do.New()
}
```

Of course, this doesn’t do anything. Let’s add a function to the container:

```go
type Printer interface {
  Print()
}

type printer struct {}

func (p printer) Print() {
  fmt.Println("hello world")
}

func PrinterProvider(_ *do.Injector) (Printer, error) {
  return printer{}, nil
}

func main() {
  container := do.New()
  do.Provide(container, PrinterProvider)
}
```

*samber/do* can automatically wire up dependencies if functions have the signature `func [T] (*do.Injector) (T, error)`, where `T` is any interface or concrete type.
Such functions are called providers.
`do.Injector` is struct returned by `do.New`.

The example above doesn’t do anything either, since we are adding functions to the container without invoking anything.
Let’s make it actually print “hello world”:

```go
func main() {
  container := do.New()
  do.Provide(container, PrinterProvider)

  p, err := do.Invoke[Printer](container)
  if err != nil {
    slog.Error("Nothing in the DI container provides an implementation of the Printer interface")
    os.Exit(1)
  }

  p.Print()
}
```

Use `do.Invoke` to fetch the entrypoint function for your application from the DI container.
Typically, there is one entrypoint per application, and it ties together all the dependencies the application needs.

Note that we only had to provide the return type of the entrypoint as a type parameter in `do.Invoke[Printer](container)`.
Since the container includes only one provider that returns `Printer`, the framework calls that provider.

## Dependencies

The biggest selling point of dependency injection frameworks is that they make it very easy to connect dependencies together.

Most modern applications are comprised of components that can be [configured with environment variables](https://12factor.net/config).
These components also produce logs, which can be very useful when troubleshooting problems with the application.
It would be chaotic if each component were responsible for creating its own logger, as we would have no way to uniformly adjust the log level, add trace IDs, extend log entries with common data, etc.

Let’s create a service that accepts a user-provided logger, and accepts configuration via a struct:

```go
type Config struct {
  Port     string
  LogLevel string
}

type Service interface {
  Start()
}

type service struct {
  logger *slog.Logger
  config Config
}

func (s service) Start() {
  s.logger.Info("starting service", "port", s.config.Port)
}

func ConfigProvider(c *do.Injector) (Config, error) {
  port := os.Getenv("APP_PORT")
  if port == "" {
    return Config{}, errors.New("APP_PORT not provided")
  }
  level := os.Getenv("LOG_LEVEL")
  if level == "" {
    level = "info"
  }
  return Config{Port: port, LogLevel: level}, nil
}

func LogProvider(c *do.Injector) (*slog.Logger, error) {
  config, err := do.Invoke[Config](c)
  if err != nil {
    return nil, errors.New("Config not provided")
  }

  var level slog.Level
  switch config.LogLevel {
  case "debug":
    level = slog.LevelDebug
  case "info":
    level = slog.LevelInfo
    // you get the idea ...
  }

  logHandlerOptions := &slog.HandlerOptions{Level: level}
  handler := slog.NewJSONHandler(os.Stdout, logHandlerOptions)
  logger := slog.New(handler)
  return logger, nil
}

func ServiceProvider(c *do.Injector) (Service, error) {
  config := do.MustInvoke[Config](c)
  logger := do.MustInvoke[*slog.Logger](c)
  return service{logger: logger, config: config}, nil
}

func main() {
  container := do.New()
  do.Provide(container, LogProvider)
  do.Provide(container, ConfigProvider)
  do.Provide(container, ServiceProvider)
  service := do.MustInvoke[Service](container)
  service.Start()
}
```

Note that providers can be in any order.
The framework will construct a dependency tree and automatically figure out the order in which provider functions need to be called.

Thus far, we have had one provider per type in all the examples.
What if we wanted to use two or more values for a given type, like multiple loggers or multiple database clients?
Without hints, the framework would not be able to determine what value to use in what function.

We can resolve this ambiguity by **naming** dependencies, using `ProvideNamed` and `InvokeNamed`.

```go
func LogProvider(c *do.Injector) (*slog.Logger, error) {
  // implementation
}

func ServiceProvider(c *do.Injector) (Service, error) {
  logger, err := do.InvokeNamed[*slog.Logger](c, "with-traces")
  if err ! nil {
    return nil, errors.New("No *slog.Logger with name 'with-traces' provided")
  }
  // implementation
}

func main() {
  container := do.New()
  container.ProvideNamed(LogProvider, "with-traces")
  container.ProvideNamed(LogProvider, "sans-traces")
  container.Provide(ServiceProvider)
  service := do.MustInvoke[Service](container)
  service.Start()
}
```

## Debugging

Certain kinds of mistakes can happen when working with dependency injection frameworks:

- forgetting to provide a dependency
- using a provider that has an incorrect return type (e.g. it returns a struct instead of an interface the struct implements)
- using multiple providers with the same return type, without using `ProvideNamed` and `InvokeNamed`

The Go compiler cannot catch these errors because *samber/do* resolves dependencies at runtime.

Fortunately, the library provides two functions that can be useful when troubleshooting errors related to dependency injection.
Use the `ListProvidedServices` and `ListInvokedServices` methods on `Injector` to find the root cause of such errors.

## But why?

Dependency injection is hugely beneficial for non-trivial applications, but **dependency injection frameworks like *samber/do* turn compile-time errors into runtime errors**.
This is not intrinsic to all DI frameworks; other languages offer a selection of DI frameworks that resolve dependencies at compile-time.
Examples include [Micronaut](https://micronaut.io/) and [Dagger 2](https://dagger.dev/) for Java, and [Needle](https://github.com/uber/needle) for Swift.

Some teams find that DI frameworks make complex applications more readable and more easily extendable, enough to offset the loss of compile-time safety.
If you find this to be true, then give *samber/do* a try!