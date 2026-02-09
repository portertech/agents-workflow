---
name: otel-mdatagen
description: Use the OpenTelemetry metadata generator (mdatagen) to create and maintain component metadata, documentation, and generated code.
---

# OpenTelemetry Metadata Generator (mdatagen)

Generate standardized metadata, documentation, and Go code for OpenTelemetry Collector components using `mdatagen`.

## Overview

Every collector component (receiver, processor, exporter, connector, extension) uses `metadata.yaml` to define:
- Component type and class
- Stability levels per signal (logs, metrics, traces)
- Metrics emitted (for receivers, scrapers, connectors)
- Attributes and resource attributes
- Feature gates
- Telemetry settings

Running `mdatagen` generates:
- `internal/metadata/generated_status.go` - Component status and stability
- `internal/metadata/generated_metrics.go` - Metric builders and types
- `internal/metadata/generated_metrics_test.go` - Metric tests
- `internal/metadata/generated_config.go` - Metrics config struct
- `internal/metadata/generated_config_test.go` - Config tests
- `internal/metadata/generated_telemetry.go` - Telemetry builder
- `internal/metadata/generated_resource.go` - Resource attribute builders
- `documentation.md` - Human-readable documentation

## metadata.yaml Schema

### Minimal Example

```yaml
type: myreceiver
status:
  class: receiver
  stability:
    beta: [metrics, traces]
    alpha: [logs]
```

### Complete Example

```yaml
type: myreceiver

status:
  class: receiver
  stability:
    stable: [metrics]
    beta: [traces]
    alpha: [logs]
  distributions:
    - contrib
  codeowners:
    active:
      - githubuser

# Optional: custom package name (default: metadata)
# generated_package_name: customname

resource_attributes:
  my.resource.attr:
    description: Description of the resource attribute.
    enabled: true
    type: string

attributes:
  my.attr:
    description: Description of the attribute.
    type: string
    enum:
      - value1
      - value2

metrics:
  my.metric.name:
    description: Description of the metric.
    unit: "1"
    enabled: true
    sum:
      value_type: int
      monotonic: true
      aggregation_temporality: cumulative
    attributes:
      - my.attr

  my.gauge.metric:
    description: A gauge metric example.
    unit: By
    enabled: true
    gauge:
      value_type: double
    attributes:
      - my.attr

telemetry:
  metrics:
    my_component_items_processed:
      description: Number of items processed.
      unit: "{items}"
      enabled: true
      sum:
        value_type: int
        monotonic: true

feature_gates:
  - id: myreceiver.newFeature
    description: Enables new feature functionality.
    stage: alpha
    from_version: v0.100.0
    reference_url: https://github.com/open-telemetry/opentelemetry-collector/issues/12345

tests:
  config:
  skip_lifecycle: false
  skip_shutdown: false
```

### Status Classes

| Class | Description |
|-------|-------------|
| `receiver` | Receives telemetry data |
| `processor` | Processes telemetry data |
| `exporter` | Exports telemetry data |
| `connector` | Connects pipelines |
| `extension` | Provides extensions |

### Stability Levels

| Level | Description |
|-------|-------------|
| `development` | Not recommended for production |
| `alpha` | May change without notice |
| `beta` | API mostly stable |
| `stable` | Production ready |
| `deprecated` | Will be removed |

### Metric Types

**Sum (Counter)**
```yaml
metrics:
  my.counter:
    description: A counter metric.
    unit: "1"
    sum:
      value_type: int          # int or double
      monotonic: true          # true for counters, false for updown
      aggregation_temporality: cumulative  # cumulative or delta
```

**Gauge**
```yaml
metrics:
  my.gauge:
    description: A gauge metric.
    unit: By
    gauge:
      value_type: double       # int or double
```

**Histogram**
```yaml
metrics:
  my.histogram:
    description: A histogram metric.
    unit: ms
    histogram:
      value_type: double
```

### Attribute Types

| Type | Description |
|------|-------------|
| `string` | String value |
| `int` | Integer value |
| `double` | Floating point value |
| `bool` | Boolean value |
| `slice` | Slice of values |
| `map` | Map of values |

### Attribute with Enum

```yaml
attributes:
  state:
    description: The state of the connection.
    type: string
    enum:
      - active
      - idle
      - closed
```

## File Structure

Standard component layout in opentelemetry-collector-contrib:

```
receiver/myreceiver/
├── metadata.yaml          # Metadata definition
├── doc.go                 # go:generate directive
├── factory.go             # Component factory
├── config.go              # Configuration struct
├── receiver.go            # Implementation
├── documentation.md       # Generated documentation
└── internal/
    └── metadata/          # Generated code
        ├── generated_status.go
        ├── generated_metrics.go
        ├── generated_metrics_test.go
        ├── generated_config.go
        ├── generated_config_test.go
        ├── generated_resource.go
        └── generated_telemetry.go
```

## doc.go Setup

Create `doc.go` in the component package:

```go
// Copyright The OpenTelemetry Authors
// SPDX-License-Identifier: Apache-2.0

//go:generate mdatagen metadata.yaml

// Package myreceiver implements a receiver that...
package myreceiver
```

## Running mdatagen

### Generate for Single Component (Recommended)

From the **opentelemetry-collector-contrib project root**, use make with the `-C` flag to target a specific component:

```bash
# Pattern: make -C <component-path> generate
make -C ./processor/spanpruningprocessor generate
make -C ./receiver/hostmetricsreceiver generate
make -C ./exporter/prometheusexporter generate
```

This is the most reliable method and handles all dependencies correctly.

### Alternative: go generate

```bash
cd processor/spanpruningprocessor
go generate ./...
```

### Generate for All Components

```bash
# From repository root
make generate
```

### Install mdatagen Locally

```bash
cd cmd/mdatagen && go install .
mdatagen metadata.yaml
```

## Using Generated Code

### Import the Metadata Package

```go
import "github.com/open-telemetry/opentelemetry-collector-contrib/receiver/myreceiver/internal/metadata"
```

### Create Metrics Builder

```go
func newScraper(cfg *Config, settings receiver.Settings) *myScraper {
    return &myScraper{
        mb: metadata.NewMetricsBuilder(cfg.MetricsBuilderConfig, settings),
    }
}
```

### Record Metrics

```go
func (s *myScraper) scrape(ctx context.Context) (pmetric.Metrics, error) {
    now := pcommon.NewTimestampFromTime(time.Now())
    
    // Record a metric with attributes
    s.mb.RecordMyMetricNameDataPoint(now, value, metadata.AttributeMyAttrValue1)
    
    // Emit all recorded metrics
    return s.mb.Emit(), nil
}
```

### Use Resource Attributes

```go
rb := metadata.NewResourceBuilder(cfg.ResourceAttributes)
rb.SetMyResourceAttr("value")
resource := rb.Emit()
```

### Access Component Metadata

```go
// Get component type
componentType := metadata.Type

// Get stability for a signal
stability := metadata.MetricsStability
```

## Common Patterns in Contrib

### Scraper-based Receiver

Receivers with internal scrapers each have their own metadata:

```
receiver/hostmetricsreceiver/
├── metadata.yaml                    # Main receiver metadata
├── doc.go
└── internal/
    └── scraper/
        └── cpuscraper/
            ├── metadata.yaml        # Scraper-specific metrics
            ├── doc.go
            └── internal/metadata/   # Generated for scraper
```

### Enabling/Disabling Metrics

```yaml
metrics:
  my.metric:
    description: An optional metric.
    enabled: false    # Disabled by default
    # ...
```

Users enable in config:
```yaml
receivers:
  myreceiver:
    metrics:
      my.metric:
        enabled: true
```

### Warnings for Metrics

```yaml
metrics:
  my.deprecated.metric:
    description: This metric is deprecated.
    enabled: true
    warnings:
      if_enabled: "This metric is deprecated and will be removed in v0.100.0"
```

### Feature Gates

```yaml
feature_gates:
  - id: receiver.myreceiver.emitMetricsV2
    description: Emit metrics using the new schema.
    stage: alpha
    from_version: v0.95.0
    reference_url: https://github.com/open-telemetry/opentelemetry-collector-contrib/issues/12345
```

## Validation

mdatagen validates:
- Required fields are present
- Stability levels are valid
- Metric types are properly defined
- Attribute references exist
- Unit formats follow conventions

## Useful Commands

| Command | Purpose |
|---------|---------|
| `make -C ./path/to/component generate` | Generate metadata for specific component (recommended) |
| `go generate ./...` | Generate metadata for current component |
| `make generate` | Generate all components in repository |
| `make mdatagen-test` | Test mdatagen itself |
| `go install ./cmd/mdatagen` | Install mdatagen binary |

## Reference Examples

- **Extensive metadata**: `receiver/elasticsearchreceiver/metadata.yaml`
- **Scraper with metrics**: `receiver/hostmetricsreceiver/internal/scraper/cpuscraper/metadata.yaml`
- **Processor**: `processor/attributesprocessor/metadata.yaml`
- **Exporter**: `exporter/prometheusexporter/metadata.yaml`
- **Extension**: `extension/healthcheckextension/metadata.yaml`

## Tips

- **Use `make -C ./path/to/component generate` from the project root** - this is the most reliable way to regenerate a single component
- Run `make generate` after any metadata.yaml changes
- Check generated `documentation.md` for accuracy
- Use `enabled: false` for optional/expensive metrics
- Follow semantic conventions for metric and attribute names
- Keep descriptions clear and concise
- Include units for all metrics (use "1" for dimensionless counts)
