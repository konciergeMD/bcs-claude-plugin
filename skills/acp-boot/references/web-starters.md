# ACP Boot — web starters reference

Two mutually-exclusive web starters. Choose by the service's runtime stack.

- `acp-boot-starter-servlet` → Spring MVC / servlet (blocking) apps.
- `acp-boot-starter-webflux` → Spring WebFlux (reactive) apps.

Both are auto-configured; both pull in `spring-boot-starter-security` and a bean
validation implementation (`hibernate-validator`). All beans are
`@ConditionalOnMissingBean` — override by declaring your own.

---

## `acp-boot-starter-servlet`

| Auto-config class | Provides | Toggle |
|---|---|---|
| `AcpSecurityServletAutoConfiguration` | `SecurityFilterChain` with CSRF disabled and Accolade's `AcpAuthenticationFilter` (header/token parsing) before `BasicAuthenticationFilter`; whitelists ignore-auth matchers | active on servlet web apps |
| `AcpMonitoringAutoConfiguration` | Ordered request filters: timer logging, monitoring/request logging, request-id (MDC) enrichment, MDC cleanup; plus `TimerInstrumentLogger` for `@Instrument`/`@Timer` AOP | `acp.monitoring.*` |
| `AcpWebAutoConfiguration` | `CommonErrorHandler` (`@ControllerAdvice`-style consistent error responses) and a trailing-slash filter | `acp.web.*` |
| `AcpAuditorAwareAutoConfiguration` | JPA `AuditorAware<String>` + `@EnableJpaAuditing` for `@CreatedBy`/`@LastModifiedBy` | `acp.security.auditor-aware-enabled=true` |

### Servlet properties

| Property | Default | Meaning |
|---|---|---|
| `acp.monitoring.process-type` | (unset) | App/process name stamped on request-id logging |
| `acp.monitoring.body-enabled` | false | Log request body (truncated to 1000 chars) |
| `acp.monitoring.token-headers-enabled` | false | Include token headers in logs |
| `acp.monitoring.query-string-enabled` | false | Include query string in logs |
| `acp.web.controller-advice-enabled` | true | Register `CommonErrorHandler` |
| `acp.web.match-trailing-slash` | true | Register trailing-slash filter |
| `acp.security.auditor-aware-enabled` | false | Enable JPA auditing via `AcpAuditorAware` |

To customize security, declare your own `SecurityFilterChain` bean — it replaces
the default. Same for `CommonErrorHandler`, `AuditorAware`, etc.

---

## `acp-boot-starter-webflux`

| Auto-config class | Provides | Toggle |
|---|---|---|
| `AcpSecurityReactiveAutoConfiguration` | `SecurityWebFilterChain` with CSRF disabled and Accolade's `AcpAuthenticationWebFilter` at the authentication stage; requires auth on all paths **except** the whitelist below | active on reactive web apps |

### Reactive auth whitelist (no authentication required)

Swagger/OpenAPI and actuator-style paths are open by default:
`/api-docs`, `/v2/api-docs`, `/v3/api-docs`, `/swagger-resources`,
`/swagger-resources/**`, `/configuration/ui`, `/configuration/security`,
`/swagger-ui.html`, `/swagger-ui/**`, `/webjars/**`, `/info`, `/health`, `/env`,
`/status`.

To change the chain or whitelist, declare your own `SecurityWebFilterChain` bean.

---

## Choosing between them

- Do **not** add both — servlet and webflux stacks conflict.
- Match the starter to how the app serves HTTP: blocking Spring MVC controllers
  → servlet; reactive `WebFlux`/`Mono`/`Flux` handlers → webflux.
- Add `acp-boot-starter-eventbus` alongside either when the service also does
  async SQS/SNS/EventBridge messaging (see `eventbus.md`).
