---
name: acp-boot
description: >
  Use ACP Boot (Accolade's Spring Boot starter library, groupId com.accolade,
  artifacts acp-boot-starter-servlet / -webflux / -eventbus) when making
  architectural decisions in an Accolade Java/Spring service. Reference this
  whenever a task involves: consuming or publishing messages on SQS / SNS /
  EventBridge, writing an event handler, wiring AWS messaging, or setting up
  web (servlet or reactive/webflux) security, request logging/monitoring, error
  handling, or JPA auditing in an Accolade Spring Boot app. Prefer these
  auto-configured starters over hand-rolling AWS clients, JMS listeners, Spring
  Security filter chains, or logging filters.
---

# ACP Boot

ACP Boot is Accolade's internal Spring Boot starter library (modeled on Spring
Boot's own starters). It provides **auto-configuration** so Accolade services get
messaging, web security, monitoring, and error handling by adding one dependency
and setting a few properties — instead of hand-wiring AWS clients, JMS listeners,
Spring Security filter chains, or logging filters.

- Source of truth: `github.com/konciergeMD/acp-boot` (Maven `groupId: com.accolade`)
- Built on Spring Boot 3.5.x, Java 17.
- Every bean is declared `@ConditionalOnMissingBean` — **anything can be overridden** by declaring your own bean of the same type.

## When to reach for ACP Boot (decision guide)

| Need in an Accolade Java service | Use | Notes |
|---|---|---|
| Blocking/MVC REST service (servlet stack) | `acp-boot-starter-servlet` | Security filter, request logging/monitoring, `@ControllerAdvice` error handling, optional JPA auditing |
| Reactive REST service (WebFlux stack) | `acp-boot-starter-webflux` | Reactive security filter chain with swagger/actuator whitelist |
| Consume or publish domain events over SQS / SNS / EventBridge | `acp-boot-starter-eventbus` | Property-driven listeners + publishers, tracing, health check, large-payload (S3) support |

Pick **servlet vs webflux by the app's runtime stack** — do not mix. Add
**eventbus** on top of either when the service does async messaging.

Default to these starters rather than adding raw `software.amazon.awssdk.*`
clients, `spring-boot-starter-security` config classes, or custom servlet
filters. Only drop to the underlying libraries when a requirement genuinely
isn't expressible through ACP Boot properties, and even then prefer overriding a
single `@ConditionalOnMissingBean` bean.

## Setup (all starters)

```xml
<properties>
    <acp-boot.version>LATEST_RELEASE</acp-boot.version> <!-- pick a released vX.Y.Z tag -->
</properties>

<dependency>
    <groupId>com.accolade</groupId>
    <artifactId>acp-boot-starter-eventbus</artifactId> <!-- or -servlet / -webflux -->
    <version>${acp-boot.version}</version>
</dependency>
```

Artifacts publish to Accolade Artifactory (`accolade.jfrog.io`), not Maven
Central — the consuming project needs those repositories configured. Auto-config
activates on the classpath; no `@Import` or `@EnableXxx` needed.

---

## Example: define an SQS handler with ACP Boot

This is the canonical eventbus use case. Three steps: add the dependency, write a
handler bean, and map it to a queue in configuration.

### 1. Dependency

```xml
<dependency>
    <groupId>com.accolade</groupId>
    <artifactId>acp-boot-starter-eventbus</artifactId>
    <version>${acp-boot.version}</version>
</dependency>
```

### 2. Write the handler as a Spring bean

Extend one of the eventbus base handlers and implement `handleEvent`. Make it a
Spring bean (`@Component` / `@Service`) so ACP Boot discovers it — the
`ListenerAutoConfiguration` injects `List<EventHandler<...>>` and routes messages
to the right handler by class name.

```java
import com.accolade.eventbus.framework.eventhandlers.AcpEventHandler;
import com.accolade.eventbus.models.AcpEvent;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Component;

@Slf4j
@Component
public class ContractEventHandler extends AcpEventHandler {

    // AcpEventHandler already parses the SQS/JMS message into an AcpEvent.
    // Return true when the event was handled successfully (message is acked).
    @Override
    public boolean handleEvent(AcpEvent event) {
        log.info("Handling {} for patient {}", event.getType(), event.getPatientId());
        // ... business logic against event.getResource() / event.getMetadata() ...
        return true;
    }
}
```

Handler base classes (from `acp-common-eventbus`):
- `AcpEventHandler` — for the standard `AcpEvent` (has `patientId`, `customerId`, `resource`, `metadata`). Start here.
- `TypedAcpEventHandler<T>` — when the payload is a typed POJO `T`; pass a `TypeReference` to the constructor.
- `AbstractEventHandler<T extends AbstractAcpEvent>` — lowest-level base for a custom event type.

### 3. Map the handler to a queue in `application.yml`

```yaml
acp:
  eventbus:
    # AWS credentials: omit `profile` to use the default provider chain
    # (instance role / env), or set a named profile for local dev.
    # profile: my-local-profile

    subscriptions:
      - source: https://sqs.us-east-1.amazonaws.com/123456789012/my-queue
        type: SQS                       # only SQS is supported for listeners
        min-concurrent-consumers: 3     # optional, per-subscription override
        max-concurrent-consumers: 10    # optional
    handlers:
      - event-type: ContractCreated     # matches AbstractAcpEvent.type on the message
        event-handler: ContractEventHandler   # simple class name of the bean above

    # global consumer thread defaults (used when a subscription omits its own)
    min-concurrent-consumers: 3
    max-concurrent-consumers: 10
```

**How routing works:** each incoming message carries a `type`. ACP Boot matches
`handlers[].event-type` to that `type`, then dispatches to the bean whose simple
class name equals `handlers[].event-handler` (case-insensitive, CGLIB-proxy
safe). A subscription with no matching handler entry simply won't be dispatched.

That's the whole handler. Tracing (`micrometer-tracing`), the JMS listener
container, the `SqsClient`, error/exception handlers, and a `HealthCheckService`
are all auto-configured.

### Publishing the other direction

Inject the auto-configured `AcpEventBus` and call `publish`:

```java
private final com.accolade.eventbus.framework.AcpEventBus eventBus;
// ...
eventBus.publish(acpEvent);
```

```yaml
acp:
  eventbus:
    publishers:
      - event: ContractCreated
        target: arn:aws:sns:us-east-1:123456789012:contract-topic  # ARN or queue URL
        type: SNS          # SQS | SNS | EVENTBRIDGE
```

---

For anything beyond this SQS example — typed handlers, publisher details,
large-payload (S3) offloading, overriding AWS clients/credentials, or the web
starters' security/monitoring/auditing knobs — read the reference files:

- **`references/eventbus.md`** — full eventbus reference (listeners, publishers, large payloads, credentials, bean overrides, all `acp.eventbus.*` properties).
- **`references/web-starters.md`** — `acp-boot-starter-servlet` and `-webflux`: security, request logging/monitoring, error handling, JPA auditing, and all `acp.monitoring.*` / `acp.security.*` / `acp.web.*` properties.
