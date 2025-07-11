# OpenTelemetry Go Setup Examples <!-- omit from toc -->

This repository offers practical examples for instrumenting Go HTTP applications with OpenTelemetry (OTel). It demonstrates manual instrumentation for popular Go HTTP frameworks, showing how to collect and export traces, metrics, and logs using OTLP exporters.

- [📦 Dependencies](#-dependencies)
- [🔧 Configuration Overview](#-configuration-overview)
- [🧪 net/http Standard Library Example](#-nethttp-standard-library-example)
  - [Key Components](#key-components)
- [⚡ Gin Framework Example](#-gin-framework-example)
  - [Key Components](#key-components-1)
- [🚀 Echo Framework Example](#-echo-framework-example)
  - [Key Components](#key-components-2)
- [📈 Exporting Telemetry Data](#-exporting-telemetry-data)
- [🧪 Example Usage](#-example-usage)
  - [net/http Application](#nethttp-application)
  - [Gin Application](#gin-application)
  - [Echo Application](#echo-application)
- [📚 References](#-references)


## 📦 Dependencies

Ensure the following packages are installed:

```bash
go mod init your-app-name
go get go.opentelemetry.io/otel \
  go.opentelemetry.io/otel/sdk \
  go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracegrpc \
  go.opentelemetry.io/otel/exporters/otlp/otlpmetric/otlpmetricgrpc \
  go.opentelemetry.io/otel/exporters/otlp/otlplog/otlploggrpc \
  go.opentelemetry.io/contrib/bridges/otelslog \
  go.opentelemetry.io/otel/sdk/log \
  go.opentelemetry.io/otel/sdk/metric \
  go.opentelemetry.io/otel/sdk/trace \
  go.opentelemetry.io/otel/semconv/v1.21.0
```

For framework-specific instrumentation, additional packages are required:

```bash
# For Gin
go get github.com/gin-gonic/gin \
  go.opentelemetry.io/contrib/instrumentation/github.com/gin-gonic/gin/otelgin

# For Echo
go get github.com/labstack/echo/v4 \
  go.opentelemetry.io/contrib/instrumentation/github.com/labstack/echo/otelecho

# For net/http
go get go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp
```

## 🔧 Configuration Overview

The examples utilize the OTLP gRPC exporter by default, with the endpoint configurable via the `OTEL_EXPORTER_OTLP_ENDPOINT` environment variable. If not set, it defaults to `localhost:4317`.

## 🧪 net/http Standard Library Example

The [net_http_setup.go](net_http_setup.go) file demonstrates how to set up OpenTelemetry in a Go application using the standard library's `net/http` package. It includes configurations for tracing, metrics, and logging, along with instrumentation for HTTP handlers.

### Key Components
- **Tracing**: Configured using TracerProvider and OTLPTraceExporter.
- **Metrics**: Set up with MeterProvider and OTLPMetricExporter.
- **Logging**: Implemented via LoggerProvider and OTLPLogExporter with structured logging using slog.
- **Instrumentation**: Applied to HTTP handlers using otelhttp middleware.

The setup functions are modular, allowing for reuse and clarity across different applications.

## ⚡ Gin Framework Example

The [gin_setup.go](gin_setup.go) file illustrates the OpenTelemetry setup for a Gin web application. The configuration is similar to the net/http example, with Gin-specific middleware integration.

### Key Components
- **Tracing**: Utilizes TracerProvider and OTLPTraceExporter.
- **Metrics**: Configured with MeterProvider and OTLPMetricExporter.
- **Logging**: Set up using LoggerProvider and OTLPLogExporter with structured logging.
- **Instrumentation**: Applied to Gin using otelgin middleware.

The setup functions mirror those in the net/http example, ensuring consistency across different frameworks.

## 🚀 Echo Framework Example

The [echo_setup.go](echo_setup.go) file demonstrates OpenTelemetry setup for an Echo web application, following the same patterns as the other examples.

### Key Components
- **Tracing**: Configured using TracerProvider and OTLPTraceExporter.
- **Metrics**: Set up with MeterProvider and OTLPMetricExporter.
- **Logging**: Implemented via LoggerProvider and OTLPLogExporter.
- **Instrumentation**: Applied to Echo using otelecho middleware.

## 📈 Exporting Telemetry Data

All examples are configured to export telemetry data using the OTLP gRPC protocol. Ensure that your OpenTelemetry Collector or backend is set up to receive data at the specified endpoint (`localhost:4317` by default).

## 🧪 Example Usage

Set the OTLP Endpoint (if different from default):
```bash
export OTEL_EXPORTER_OTLP_ENDPOINT="localhost:4317"
```

### net/http Application
```go
package main

import (
    "context"
    "fmt"
    "net/http"
    "os"
    "os/signal"
    "syscall"
    "time"
)

func main() {
    // Initialize OpenTelemetry
    shutdown := setupInstrumentation()
    defer shutdown()

    // Create HTTP server with instrumentation
    mux := http.NewServeMux()
    mux.Handle("/", otelhttp.NewHandler(http.HandlerFunc(helloHandler), "hello"))
    
    server := &http.Server{
        Addr:    ":8090",
        Handler: mux,
    }

    // Start server
    go func() {
        fmt.Println("Server starting on :8090")
        if err := server.ListenAndServe(); err != nil && err != http.ErrServerClosed {
            appLogger.Error("Server failed to start", "error", err)
        }
    }()

    // Wait for interrupt signal
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
    <-quit

    // Graceful shutdown
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()
    if err := server.Shutdown(ctx); err != nil {
        appLogger.Error("Server forced to shutdown", "error", err)
    }
}
```

```bash
go run net_http_setup.go
```

### Gin Application
```go
package main

import (
    "github.com/gin-gonic/gin"
    "go.opentelemetry.io/contrib/instrumentation/github.com/gin-gonic/gin/otelgin"
)

func main() {
    // Initialize OpenTelemetry
    shutdown := setupInstrumentation()
    defer shutdown()

    // Create Gin router with instrumentation
    r := gin.Default()
    r.Use(otelgin.Middleware("gin-service"))
    
    r.GET("/", helloHandler)
    
    appLogger.Info("Server starting on :8090")
    r.Run(":8090")
}
```

```bash
go run gin_setup.go
```

### Echo Application
```go
package main

import (
    "github.com/labstack/echo/v4"
    "go.opentelemetry.io/contrib/instrumentation/github.com/labstack/echo/otelecho"
)

func main() {
    // Initialize OpenTelemetry
    shutdown := setupInstrumentation()
    defer shutdown()

    // Create Echo instance with instrumentation
    e := echo.New()
    e.Use(otelecho.Middleware("echo-service"))
    
    e.GET("/", helloHandler)
    
    appLogger.Info("Server starting on :8090")
    e.Start(":8090")
}
```

```bash
go run echo_setup.go
```

## 📚 References

- [OpenTelemetry Go Documentation](https://opentelemetry.io/docs/instrumentation/go/)
- [OpenTelemetry Go SDK](https://github.com/open-telemetry/opentelemetry-go)
- [OpenTelemetry Go Contrib](https://github.com/open-telemetry/opentelemetry-go-contrib)
- [Gin OpenTelemetry Instrumentation](https://github.com/open-telemetry/opentelemetry-go-contrib/tree/main/instrumentation/github.com/gin-gonic/gin/otelgin)
- [Echo OpenTelemetry Instrumentation](https://github.com/open-telemetry/opentelemetry-go-contrib/tree/main/instrumentation/github.com/labstack/echo/otelecho)
- [net/http OpenTelemetry Instrumentation](https://github.com/open-telemetry/opentelemetry-go-contrib/tree/main/instrumentation/net/http/otelhttp)