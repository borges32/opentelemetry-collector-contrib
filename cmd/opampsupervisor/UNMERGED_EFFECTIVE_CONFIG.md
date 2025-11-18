# Unmerged Effective Config Capability

## Overview

This feature adds a new capability to the OpAMP Supervisor called `reports_unmerged_effective_config` that allows reporting the original agent configuration to the OpAMP server without supervisor-injected modifications.

## Problem Statement

By default, the OpAMP Supervisor injects several additional configurations into the collector's configuration file:
- OpAMP extension configuration for communication with the supervisor
- Own telemetry configuration
- Extra telemetry settings

When the supervisor sends the `effectiveConfig` to the OpAMP server, it sends the complete configuration with all these injections. This can be problematic when:
1. The OpAMP server needs to see the original configuration sent, not the modified version
2. It's necessary to audit or debug the configuration without supervisor modifications
3. The configuration persisted to disk should match what's reported to OpAMP

## How It Works

### 1. New Capability: `reports_unmerged_effective_config`

When enabled, this capability changes the supervisor's behavior:

**Before** (default behavior):
```
Local Config + Remote Config → Merge → + OpAMP Extension + Own Telemetry → effectiveConfig (sent to OpAMP)
```

**After** (with capability enabled):
```
Local Config + Remote Config → unmergedConfig (sent to OpAMP)
                             → Merge → + OpAMP Extension + Own Telemetry → effectiveConfig (used by collector)
```

### 2. Integration with Persist Remote Config

When combined with `persist_remote_config_path`, the configuration persisted to disk also updates the `unmergedConfig`:

```yaml
capabilities:
  reports_unmerged_effective_config: true

agent:
  persist_remote_config_path: /var/lib/otelcol/remote_config.yaml
```

Flow:
1. OpAMP Server sends new configuration
2. Supervisor persists the configuration to `/var/lib/otelcol/remote_config.yaml`
3. Supervisor updates `unmergedConfig` with the persisted configuration
4. Supervisor reports `unmergedConfig` in the `effectiveConfig` field to OpAMP
5. Supervisor creates the complete configuration (with injections) for the collector

## Configuration

### Basic Example

```yaml
server:
  endpoint: wss://opamp-server.example.com/v1/opamp

capabilities:
  reports_effective_config: true
  reports_unmerged_effective_config: true  # New capability
  accepts_remote_config: true

agent:
  executable: /usr/local/bin/otelcontribcol
```

### Complete Example with Persistence

```yaml
server:
  endpoint: wss://opamp-server.example.com/v1/opamp

capabilities:
  reports_effective_config: true
  reports_unmerged_effective_config: true
  accepts_remote_config: true

agent:
  executable: /usr/local/bin/otelcontribcol
  # Configuration received from OpAMP will be persisted here
  persist_remote_config_path: /var/lib/otelcol/remote_config.yaml

storage:
  directory: /var/lib/otelcol/supervisor
```

## Use Cases

### 1. Configuration Auditing
When you need to audit exactly what the OpAMP server sent, without supervisor modifications:

```yaml
capabilities:
  reports_unmerged_effective_config: true
  accepts_remote_config: true
```

The OpAMP server will receive the original configuration in the `effectiveConfig` field.

### 2. Configuration Debugging
To debug configuration issues, it's useful to see the original configuration without supervisor injections:

```yaml
capabilities:
  reports_unmerged_effective_config: true

agent:
  persist_remote_config_path: /tmp/debug_config.yaml
```

You can inspect `/tmp/debug_config.yaml` to see exactly what was sent by OpAMP.

### 3. Backup and Recovery
Keep a clean copy of the configuration received from OpAMP:

```yaml
capabilities:
  reports_unmerged_effective_config: true

agent:
  persist_remote_config_path: /backup/otelcol_config_backup.yaml
```

## Technical Implementation

### Code Changes

1. **config/config.go**
   - Added `ReportsUnmergedEffectiveConfig` field to `Capabilities` struct

2. **supervisor.go**
   - Added `unmergedConfig *atomic.Value` field to `Supervisor` struct
   - Modified `composeMergedConfig()` to store config before merge
   - Modified `createEffectiveConfigMsg()` to use `unmergedConfig` when capability is enabled
   - Modified `persistRemoteConfigToDisk()` to update `unmergedConfig` when capability is enabled

3. **supervisor_test.go**
   - Added tests for the new functionality

### Data Flow

```
1. OpAMP Server sends RemoteConfig
   ↓
2. processRemoteConfigMessage()
   ├─→ saveLastReceivedConfig()
   ├─→ persistRemoteConfigToDisk() 
   │   └─→ [if reports_unmerged_effective_config] unmergedConfig.Store()
   └─→ composeMergedConfig()
       ├─→ composeAgentConfigFiles() → agentConfigBytes
       ├─→ [if reports_unmerged_effective_config] unmergedConfig.Store(agentConfigBytes)
       └─→ + OpAMP Extension + Own Telemetry → mergedConfig

3. createEffectiveConfigMsg()
   └─→ [if reports_unmerged_effective_config] return unmergedConfig
       [else] return effectiveConfig or mergedConfig
```

## Compatibility

- **Backward Compatible**: If the capability is not enabled, the behavior is identical to the previous version
- **Minimum Version**: Requires OpenTelemetry Collector Contrib with OpAMP Supervisor support
- **Dependencies**: No additional dependencies

## Limitations

1. If `reports_unmerged_effective_config` is enabled but `unmergedConfig` is not available (e.g., at startup before receiving a remote config), the supervisor will fall back to the default behavior.

2. The configuration reported as "unmerged" still includes the merge of multiple local configuration files specified in `agent.config_files`.

## Configuration Examples

See example files at:
- `examples/supervisor_with_unmerged_config.yaml`
- `examples/supervisor_with_persist_config.yaml`

## References

- [OpAMP Specification](https://github.com/open-telemetry/opamp-spec)
- [OpenTelemetry Collector Contrib](https://github.com/open-telemetry/opentelemetry-collector-contrib)
