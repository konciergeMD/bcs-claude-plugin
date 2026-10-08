# ACP Boot — `acp-boot-starter-eventbus` reference

Auto-configured messaging over AWS SQS / SNS / EventBridge, driven by
`acp.eventbus.*` properties. All beans are `@ConditionalOnMissingBean`, so any
piece can be replaced by declaring your own bean.

## What gets auto-configured

| Auto-config class | Provides |
|---|---|
| `DefaultAwsConfiguration` | `AwsCredentialsProvider` (default chain, or named `profile`), SQS `ProviderConfiguration` |
| `DefaultPayloadAwsConfiguration` | Plain `SqsClient`, `SnsClient`, `EventBridgeClient` (runs last in the chain) |
| `ExtendedPayloadAwsConfiguration` | Large-payload (S3-backed) SQS/SNS clients — only when `acp.eventbus.large-payload.enabled=true` |
| `PublisherAutoConfiguration` | `AcpEventBus` bean wired with SNS/SQS/EventBridge publishers from `acp.eventbus.publishers` |
| `ListenerAutoConfiguration` | JMS listener container (`JmsController`), message handler, error/exception handlers; routes messages to your `EventHandler` beans |
| `HealthCheckAutoConfiguration` | `HealthCheckService` over SNS + SQS |

## Listeners (consuming)

Only **SQS** is supported as a listener source. Two moving parts:

1. **Handler beans** — Spring beans implementing `EventHandler<T>`. ACP Boot
   injects `List<EventHandler<...>>` and keys them by lowercase simple class name
   (CGLIB-proxy safe, so `@Instrument`/AOP-advised handlers still resolve).
2. **`acp.eventbus.handlers`** — maps each event `type` to a handler class name.
3. **`acp.eventbus.subscriptions`** — maps each SQS queue to the listener container.

### Handler base classes (from `com.accolade.acp-common-eventbus:framework`)

```java
// interface every handler implements
interface EventHandler<T extends AbstractAcpEvent> {
    boolean parseMessage(JsonNode node, Tracer tracer) throws IOException; // usually inherited
    boolean handleEvent(T event);                                         // you implement this
}
```

- `AcpEventHandler extends AbstractEventHandler<AcpEvent>` — standard event with
  `patientId`, `customerId`, `resource` (JsonNode), `metadata` (JsonNode).
  Simplest starting point; `parseMessage` is inherited.
- `TypedAcpEventHandler<T> extends AbstractTypedAcpEventHandler<TypedAcpEvent<T>, T>`
  — payload deserialized to a typed POJO `T`. Constructor requires a
  `TypeReference<TypedAcpEvent<T>>`.
- `AbstractEventHandler<T extends AbstractAcpEvent>` — base for a fully custom
  event type; construct with `Class<T>` (and optionally a custom `ObjectMapper`).

`handleEvent` returns `boolean`: `true` = handled (message acknowledged),
`false`/exception = routed to the configured `ErrorHandler` (default logs at
error level; override the `ErrorHandler` bean to change behavior).

### Typed handler example

```java
@Component
public class ClaimEventHandler extends TypedAcpEventHandler<Claim> {
    public ClaimEventHandler() {
        super(new TypeReference<TypedAcpEvent<Claim>>() {});
    }

    @Override
    public boolean handleEvent(TypedAcpEvent<Claim> event) {
        Claim claim = event.getData(); // typed payload
        // ...
        return true;
    }
}
```

## Publishers (producing)

Inject the auto-configured `AcpEventBus` and call `publish(...)`. It selects the
publisher by the event key you configured.

```java
@RequiredArgsConstructor
@Service
public class ContractService {
    private final AcpEventBus eventBus;

    public void onContractCreated(AcpEvent event) {
        eventBus.publish(event); // AbstractAcpEvent overload
    }
}
```

`AcpEventBus.publish` overloads: `AbstractAcpEvent`, `SnsPublishEvent`,
`AbstractSqsPublishEvent`, `EventBridgePublishEvent`.

```yaml
acp:
  eventbus:
    publishers:
      - event: ContractCreated                 # logical key used to pick the publisher
        target: arn:aws:sns:...:contract-topic  # ARN (SNS/EventBridge) or queue URL (SQS)
        type: SNS                               # SQS | SNS | EVENTBRIDGE
```

## Large payloads (S3 offload)

When enabled, messages over the SQS/SNS size limit are stored in S3 and a pointer
is sent. Requires the optional extended-client dependencies **and** the property.

```xml
<dependency>
    <groupId>software.amazon.sns</groupId>
    <artifactId>sns-extended-client</artifactId>
    <version>2.1.0</version>
</dependency>
<dependency>
    <groupId>com.amazonaws</groupId>
    <artifactId>amazon-sqs-java-extended-client-lib</artifactId>
    <version>2.1.3</version>
</dependency>
```

```yaml
acp:
  eventbus:
    large-payload:
      enabled: true
      global-s3-name: my-overflow-bucket
```

## AWS credentials & client overrides

- No `acp.eventbus.profile` → `DefaultCredentialsProvider` (instance role, env
  vars, etc.). Set `profile: <name>` for a named local profile
  (`ProfileCredentialsProvider`).
- To fully control a client, declare your own bean — it wins via
  `@ConditionalOnMissingBean`:

```java
@Bean
public SqsClient getAmazonSqs(AwsCredentialsProvider creds) {
    return SqsClient.builder().region(Region.US_EAST_1).credentialsProvider(creds).build();
}
```

The same override pattern applies to `SnsClient`, `EventBridgeClient`,
`AwsCredentialsProvider`, `ProviderConfiguration`, `ErrorHandler`,
`ExceptionListener`, `JmsMessageHandler`, and `JmsController`.

## Full `acp.eventbus.*` property reference

| Property | Default | Meaning |
|---|---|---|
| `acp.eventbus.profile` | (unset) | Named AWS profile; unset = default provider chain |
| `acp.eventbus.min-concurrent-consumers` | 3 | Global default min listener threads |
| `acp.eventbus.max-concurrent-consumers` | 10 | Global default max listener threads |
| `acp.eventbus.subscriptions[].source` | — | SQS queue URL |
| `acp.eventbus.subscriptions[].type` | SQS | Listener type (only `SQS`) |
| `acp.eventbus.subscriptions[].min-concurrent-consumers` | 0 (→ global) | Per-subscription min threads |
| `acp.eventbus.subscriptions[].max-concurrent-consumers` | 0 (→ global) | Per-subscription max threads |
| `acp.eventbus.handlers[].event-type` | — | Event `type` value on the message |
| `acp.eventbus.handlers[].event-handler` | — | Simple class name of the handler bean |
| `acp.eventbus.publishers[].event` | — | Logical key passed to `publish` selection |
| `acp.eventbus.publishers[].target` | — | ARN (SNS/EventBridge) or queue URL (SQS) |
| `acp.eventbus.publishers[].type` | — | `SQS` \| `SNS` \| `EVENTBRIDGE` |
| `acp.eventbus.large-payload.enabled` | (unset) | Enable S3-backed large payloads |
| `acp.eventbus.large-payload.global-s3-name` | (unset) | S3 bucket for overflow payloads |
