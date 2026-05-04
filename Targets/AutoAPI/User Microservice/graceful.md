# Graceful Degradation и Shutdown для `account-vault`

## Graceful Shutdown

`account-vault` — gRPC-сервис с PostgreSQL/Prisma и Outbox. Shutdown сложнее, чем кажется, потому что есть три независимых слоя, которые нужно остановить в правильном порядке.

### 1. Порядок остановки (обратный порядку запуска)

```
SIGTERM получен
    │
    ├─► Readiness probe → NOT READY (k8s перестаёт слать трафик)
    │
    ├─► gRPC server.tryShutdown() — перестать принимать новые запросы
    │       └─ in-flight запросы — дать им завершиться (timeout ~15s)
    │
    ├─► Outbox Processor — остановить polling, дождаться текущего batch
    │
    ├─► Prisma — $disconnect()
    │
    └─► Process exit(0)
```

### 2. Signal handling в `main.ts`

```typescript
// main.ts
async function bootstrap() {
  const app = await NestFactory.createMicroservice<MicroserviceOptions>(
    AppModule,
    { transport: Transport.GRPC, options: { ... } }
  );

  // Readiness flag — health controller читает его
  const shutdownSignal = app.get(ShutdownSignal);

  const signals = ['SIGTERM', 'SIGINT'];
  for (const signal of signals) {
    process.on(signal, async () => {
      shutdownSignal.markNotReady(); // ← readiness probe начнёт отдавать 503
      await sleep(5_000);            // ← grace period, k8s успевает убрать pod из Endpoints
      await app.close();             // ← NestJS lifecycle hooks
      process.exit(0);
    });
  }

  await app.listen();
}
```

### 3. Lifecycle hooks по слоям

```typescript
// outbox.processor.ts
@Injectable()
export class OutboxProcessor implements OnModuleDestroy {
  private running = false;
  private currentBatch: Promise<void> | null = null;

  async onModuleDestroy() {
    this.running = false;
    // Дожидаемся текущего batch — не бросаем на полпути
    if (this.currentBatch) {
      await Promise.race([
        this.currentBatch,
        sleep(10_000), // hard timeout
      ]);
    }
  }
}

// prisma.service.ts
@Injectable()
export class PrismaService extends PrismaClient implements OnModuleDestroy {
  async onModuleDestroy() {
    await this.$disconnect();
  }
}
```

### 4. In-flight request tracking

```typescript
// interceptors/shutdown-guard.interceptor.ts
@Injectable()
export class ShutdownGuardInterceptor implements NestInterceptor {
  constructor(private readonly shutdownSignal: ShutdownSignal) {}

  intercept(ctx: ExecutionContext, next: CallHandler): Observable<any> {
    if (this.shutdownSignal.isShuttingDown()) {
      // Новые запросы отклоняем с UNAVAILABLE — клиент retry на другой pod
      throw new RpcException({
        code: status.UNAVAILABLE,
        message: 'Service is shutting down',
      });
    }
    return next.handle();
  }
}
```

---

## Graceful Degradation

`account-vault` — критический сервис: упал он — упал весь auth. Поэтому деградация важнее, чем для большинства сервисов.

### 1. Read-Only режим при деградации БД

Когда PostgreSQL недоступен для записи (failover, network partition), часть операций всё ещё может работать:

```typescript
// accounts.service.ts
async createAccount(dto: CreateAccountDto): Promise<Account> {
  if (await this.dbHealthCheck.isReadOnly()) {
    throw new RpcException({
      code: status.UNAVAILABLE,
      message: 'Service temporarily in read-only mode',
    });
  }
  // ...
}

async getAccount(id: string): Promise<Account> {
  // Reads работают даже при частичной деградации
  return this.accountRepository.findById(id);
}
```

**Что работает в read-only:**

- `GetAccount`, `GetAccountByEmail`
- `VerifyPassword` (read + bcrypt)
- `ValidateApiKey`

**Что не работает:**

- `CreateAccount`, `UpdatePassword`, `DisableAccount`, `BanAccount`

### 2. Outbox как буфер при недоступности downstream

Outbox уже есть в структуре — это и есть ключевой механизм деградации:

```
Создаём аккаунт → пишем в БД + outbox в одной транзакции
                       │
              notification-service недоступен?
                       │
              outbox хранит событие, retry позже
                       │
              аккаунт создан успешно, письмо придёт с задержкой
```

```typescript
// accounts.service.ts
async createAccount(dto: CreateAccountDto): Promise<Account> {
  return this.prisma.$transaction(async (tx) => {
    const account = await this.accountRepository.create(dto, tx);
    
    // Событие в outbox — atomically с созданием аккаунта
    await this.outbox.publish(
      new AccountCreatedEvent(account.id, account.email),
      tx,
    );
    
    return account;
  });
  // Если notification-service лежит — аккаунт всё равно создан
}
```

### 3. Circuit Breaker для внешних зависимостей

Если `account-vault` сам обращается к внешнему сервису (например, к `notification-service` напрямую вне outbox — антипаттерн, но бывает), нужен circuit breaker. В данной архитектуре это скорее защита на стороне Prisma connection pool:

```typescript
// shared/prisma/prisma.service.ts
@Injectable()
export class PrismaService extends PrismaClient {
  private consecutiveErrors = 0;
  private readonly ERROR_THRESHOLD = 5;

  async executeWithCircuitBreaker<T>(
    operation: () => Promise<T>,
  ): Promise<T> {
    if (this.consecutiveErrors >= this.ERROR_THRESHOLD) {
      throw new RpcException({
        code: status.UNAVAILABLE,
        message: 'Database circuit breaker open',
      });
    }

    try {
      const result = await operation();
      this.consecutiveErrors = 0;
      return result;
    } catch (err) {
      if (this.isTransientError(err)) {
        this.consecutiveErrors++;
      }
      throw err;
    }
  }
}
```

### 4. Connection Pool Exhaustion

```typescript
// config/config.schema.ts — настройки пула
datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
  // connection_limit в URL: ?connection_limit=20&pool_timeout=5
}
```

При исчерпании пула — быстрый fail с правильным gRPC кодом, а не зависание:

```typescript
// grpc-exception.filter.ts
catch (error) {
  if (error.message.includes('Timed out fetching a new connection')) {
    return { code: status.RESOURCE_EXHAUSTED, message: 'Too many requests' };
  }
}
```

### 5. Health checks как управляющий механизм деградации

```typescript
// health/health.controller.ts
@Controller('health')
export class HealthController {
  
  // Liveness — "process жив?"
  // Если упал — k8s рестартует pod
  @Get('live')
  liveness() {
    return { status: 'ok' };
  }

  // Readiness — "принимать трафик?"
  // Если БД недоступна — k8s убирает pod из балансировки
  @Get('ready')
  async readiness() {
    if (this.shutdownSignal.isShuttingDown()) {
      throw new ServiceUnavailableException('Shutting down');
    }

    const dbOk = await this.checkDb();
    if (!dbOk) {
      throw new ServiceUnavailableException('Database unavailable');
    }

    return { status: 'ok' };
  }

  // Startup — "первый запуск завершён?"
  // Даёт время на миграции при старте
  @Get('startup')
  async startup() {
    const migrationsOk = await this.checkMigrations();
    if (!migrationsOk) {
      throw new ServiceUnavailableException('Migrations pending');
    }
    return { status: 'ok' };
  }
}
```

---

## Итоговая карта стратегий

|Сценарий|Стратегия|Поведение|
|---|---|---|
|SIGTERM получен|Shutdown sequence|Readiness→503, drain in-flight, stop outbox, disconnect Prisma|
|PostgreSQL недоступен|Read-only mode + readiness|Reads работают, writes → UNAVAILABLE, pod уходит из балансировки|
|notification-service лежит|Outbox buffer|Аккаунт создан, событие отложено, retry автоматически|
|Connection pool исчерпан|Fast fail|RESOURCE_EXHAUSTED, клиент retry позже|
|Много transient ошибок БД|Circuit breaker|UNAVAILABLE после N ошибок подряд, защита БД от шторма|
|Новый запрос во время shutdown|ShutdownGuardInterceptor|UNAVAILABLE, gRPC клиент retry на другой pod|
|Долгий startup (миграции)|Startup probe|Pod не получает трафик, пока миграции не завершены|

Ключевая идея: `account-vault` должен уметь **сказать «нет» быстро и явно**, а не зависать — это позволяет `auth-orchestrator` выше по стеку принимать решения о retry/fallback самостоятельно.