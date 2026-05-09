# Desktop Configuration

Vipr Desktop uses a centralized configuration system to manage worker, process, IPC, and security settings.

## Configuration File

All configuration is defined in `clients/desktop/src/shared/config.ts`. The configuration system provides type-safe settings using Zod validation, supports environment-specific configs (development vs production), and allows runtime overrides via environment variables.

## Configuration Categories

### Worker Configuration

| Setting               | Development     | Production       | Environment Variable          | Description                    |
| --------------------- | --------------- | ---------------- | ----------------------------- | ------------------------------ |
| `timeout`             | 30000ms         | 45000ms          | `VIPR_WORKER_TIMEOUT`         | Analysis timeout per file      |
| `cacheTTL`            | 300000ms (5min) | 600000ms (10min) | `VIPR_WORKER_CACHE_TTL`       | Analysis result cache duration |
| `startupTimeout`      | 10000ms         | 15000ms          | `VIPR_WORKER_STARTUP_TIMEOUT` | Worker startup timeout         |
| `shutdownGracePeriod` | 5000ms          | 10000ms          | `VIPR_WORKER_SHUTDOWN_GRACE`  | Graceful shutdown wait time    |

### Process Configuration

| Setting                                 | Development | Production | Environment Variable     | Description                 |
| --------------------------------------- | ----------- | ---------- | ------------------------ | --------------------------- |
| `errorHandling.showDialogOnCrash`       | true        | true       | `VIPR_SHOW_ERROR_DIALOG` | Show error dialog on crash  |
| `errorHandling.exitOnUncaughtException` | true        | true       | `VIPR_EXIT_ON_ERROR`     | Exit on uncaught exceptions |
| `errorHandling.exitDelayMs`             | 1000ms      | 2000ms     | `VIPR_EXIT_DELAY`        | Delay before exit           |
| `logging.level`                         | debug       | info       | `VIPR_LOG_LEVEL`         | Log verbosity level         |

### IPC Configuration

| Setting          | Development | Production | Environment Variable      | Description                   |
| ---------------- | ----------- | ---------- | ------------------------- | ----------------------------- |
| `defaultTimeout` | 30000ms     | 45000ms    | `VIPR_IPC_TIMEOUT`        | Default IPC request timeout   |
| `retryAttempts`  | 0           | 0          | `VIPR_IPC_RETRY_ATTEMPTS` | Retry count                   |
| `batchSize`      | 10          | 10         | `VIPR_IPC_BATCH_SIZE`     | Batch size for data transfers |

### Coordinator Configuration

| Setting         | Development | Production | Environment Variable              | Description                |
| --------------- | ----------- | ---------- | --------------------------------- | -------------------------- |
| `maxConcurrent` | 4           | 2          | `VIPR_COORDINATOR_MAX_CONCURRENT` | Max parallel file analyses |
| `debounceMs`    | 500ms       | 1000ms     | `VIPR_COORDINATOR_DEBOUNCE`       | File change debounce delay |
| `batchSize`     | 10          | 10         | `VIPR_COORDINATOR_BATCH_SIZE`     | Initial scan batch size    |
| `timeoutMs`     | 60000ms     | 90000ms    | `VIPR_COORDINATOR_TIMEOUT`        | Per-file analysis timeout  |

### Security Configuration

#### CSP Directives

Production and development use the same CSP for consistency.

#### Sanitization Limits

| Setting            | Default       | Environment Variable   | Description              |
| ------------------ | ------------- | ---------------------- | ------------------------ |
| `maxPathLength`    | 4096          | `VIPR_MAX_PATH_LENGTH` | Maximum file path length |
| `maxUrlLength`     | 2048          | `VIPR_MAX_URL_LENGTH`  | Maximum URL length       |
| `allowedProtocols` | http:, https: | N/A                    | Allowed URL protocols    |

## Environment Variable Overrides

Override configuration values using environment variables:

```bash
# Increase worker timeout
VIPR_WORKER_TIMEOUT=60000 npm run dev

# Enable debug logging
VIPR_LOG_LEVEL=debug npm start

# Reduce concurrency
VIPR_COORDINATOR_MAX_CONCURRENT=2 npm run dev
```

## Performance Tuning

### High-End Machines

```bash
VIPR_COORDINATOR_MAX_CONCURRENT=8 VIPR_WORKER_CACHE_TTL=600000
```

### Low-End Machines

```bash
VIPR_COORDINATOR_MAX_CONCURRENT=1 VIPR_WORKER_TIMEOUT=120000
```

### Large Codebases

```bash
VIPR_WORKER_TIMEOUT=120000 VIPR_COORDINATOR_TIMEOUT=300000
```

## Configuration in Code

```typescript
import { config, buildCSPHeader } from '../shared/config';

// Worker timeout
const timeout = config.worker.timeout;

// CSP header
mainWindow.webContents.session.webRequest.onHeadersReceived((details, callback) => {
  callback({
    responseHeaders: {
      ...details.responseHeaders,
      'Content-Security-Policy': buildCSPHeader(),
    },
  });
});
```

## Troubleshooting

### Timeout Errors

Increase relevant timeouts via environment variables.

### High Memory Usage

Reduce concurrency and cache TTL.

### Slow Performance

Increase concurrency if hardware allows.
