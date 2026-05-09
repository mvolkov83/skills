---
name: grpc-python-best-practices
description: Python gRPC best practices for async-first services using `grpc.aio` in Kubernetes — covers project structure (`.proto` files, committed generated stubs), protobuf message design and field-number stability, async server bootstrap with graceful shutdown that flips Health to NOT_SERVING before `server.stop()`, async client setup with a single shared per-process channel, all four streaming patterns, the canonical decorator pattern (`@grpc_logger` + `@grpc_error_handler`) for cross-cutting concerns and `ServerInterceptor` when you need wire-level access, channel and server configuration with `max_connection_age_*` for HPA rebalancing and matching keepalive on both ends, client-side load balancing via `dns:///` URI scheme + `lb_policy_name=round_robin` + service config JSON over a Kubernetes headless Service (no service mesh), rich error model using an exception-to-(status, business_code, message) map + `google.rpc.Status` + a per-domain detail proto packed via `rpc_status.to_status()`, client-side translation from `grpc.RpcError` into a per-service-area exception hierarchy (`*Unavailable / *InvalidArgument / *Conflict / *Unknown`) so orchestrators never see `RpcError`, deadlines, UNAVAILABLE-only retries, and the `experimental_non_blocking=True` Health service. Use this skill whenever the user is writing, reviewing, debugging, or designing Python gRPC code — including any work involving `grpc.aio.server`, `grpc.aio.insecure_channel`, `grpc.aio.secure_channel`, `ServerInterceptor`, `UnaryUnaryClientInterceptor` and the other client interceptor base classes, `.proto` files compiled with `grpc_tools.protoc`, channel options, service config JSON, LB policy selection, `max_connection_age_ms`, headless Services for gRPC, mTLS setup, async streaming RPCs, status code error handling, or rich error details. Trigger on any review of files containing `grpc.aio.*` imports, generated `*_pb2.py` / `*_pb2_grpc.py` files, `.proto` files in a Python project, or gRPC servicer / channel-factory / client-wrapper implementations. Do NOT use for grpc-go, grpc-java, grpc-web in browsers, or generic RPC questions unrelated to gRPC.
---

# gRPC Python Best Practices (Async-First, Kubernetes-Native)

This skill captures production-tested patterns for Python gRPC services built on `grpc.aio` and deployed in Kubernetes without a service mesh. Sync `grpcio` is legacy here — when reviewing existing sync code, recommend migration to `grpc.aio` and explain the path.

User-emphasized depth areas: **interceptors / decorators, configuration, load balancing**. Other sections are tighter — expand when a specific question lands there.

For full HPA / rollout / verification scenarios for the load balancing setup, the project should keep a per-service `docs/grpc_load_balancing.md` doc capturing the deployment narrative. This skill captures the patterns; that doc captures the rollout.

## Stack

- `grpcio` (async API: `grpc.aio.*`)
- `grpcio-tools` for `python -m grpc_tools.protoc` to generate `*_pb2.py` (messages) and `*_pb2_grpc.py` (stubs) — **commit the generated stubs** (recommended convention)
- `grpcio-health-checking` for `grpc.health.v1.Health` — required for k8s liveness probes via `grpc-health-probe` and for the L7 LB drain signal
- `grpcio-status` (`grpc_status.rpc_status`) for the rich error model
- `googleapis-common-protos` for `google.rpc.Status` and `google.rpc.Code`
- `structlog` for structured logging
- `opentelemetry-instrumentation-grpc` for distributed tracing
- A per-domain `*_proto` package holding the compiled `.proto` modules — single source of truth shared by producer and consumers

## Project Structure & Proto Organization

```
my-service/
├── docs/
│   └── grpc_load_balancing.md       # canonical LB+k8s narrative for this service
├── my_service/
│   ├── apps/
│   │   ├── <server_app>/            # one app = one gRPC server process
│   │   │   ├── server.py            # _compose_and_serve() bootstrap
│   │   │   ├── service.py           # MyServiceServicer
│   │   │   └── <other>_service.py
│   │   └── <other_server_app>/      # parallel layout for separate processes
│   ├── shared/
│   │   ├── grpc_tools/
│   │   │   ├── grpc_error_handler.py
│   │   │   ├── grpc_logger.py
│   │   │   ├── log_extractors.py
│   │   │   └── health.py
│   │   ├── clients/                 # one subdir per upstream gRPC dependency
│   │   │   └── <upstream_service>/
│   │   │       ├── channel.py       # build_<upstream>_channel()
│   │   │       ├── <area>_client.py # thin async wrapper around generated stub
│   │   │       └── errors.py        # domain exception hierarchy for this client
│   │   └── exceptions/              # domain exception classes
│   └── telemetry.py
└── tests/
```

1. **One app per gRPC server process.** Multi-server deployments compose by separate `apps/<server>/` subdirectories, each with its own `_compose_and_serve()`. Don't multiplex unrelated services on one port.
2. **Commit generated stubs.** Build-time generation works for greenfield, but committing makes the repo clone-and-run, eliminates `protoc` from CI, and surfaces proto-stub diffs in code review.
3. **Versioned proto packages**: `myservice.v1.MyService` rather than bare `myservice.MyService`. Greenfield services start versioned; breaking changes go to v2 alongside v1, not by mutating v1.
4. **A separate `*_proto` package** holds the compiled `.proto` modules. Both producer and consumers depend on this package — single source of truth for the wire schema.

## Protobuf Design

5. **Field numbers are forever.** Never reuse a removed field's number — old clients with cached stubs will deserialize garbage. Mark removed fields with `reserved`:
   ```proto
   message Account {
     reserved 3, 5;
     reserved "old_email";
     string id = 1;
     string account_number = 2;
     // field 3 was 'old_email' — removed
     string display_name = 4;
   }
   ```
6. **Numbers 1-15 use 1 byte; 16-2047 use 2 bytes.** Put high-cardinality fields (present on most messages) in 1-15.
7. **Optional vs default**: in proto3, scalar fields default to zero values, indistinguishable from "absent". Use `optional` keyword (proto3.15+) when you need to detect "field not set".
8. **Avoid required.** Removed in proto3 for a reason — it's forwards-incompatible.
9. **Use semantic types**: `google.protobuf.Timestamp` for time, `google.protobuf.Duration` for durations, `google.protobuf.FieldMask` for partial updates. Don't reinvent with int64-of-millis.
10. **Pack rich error details into a per-domain detail message** (e.g. `MyServiceErrorDetail { string code; string message; map<string, string> metadata; }`) and use the same detail type across all services in a domain. A single error contract simplifies the client-side translation table.

## Async Server Bootstrap

11. The canonical bootstrap pattern:

    ```python
    import asyncio
    import signal

    from grpc import aio
    from grpc_health.v1.health import HealthServicer

    _SHUTDOWN_GRACE_SECONDS = 35  # match max_connection_age_grace_ms + slack


    async def _compose_and_serve() -> None:
        options = [
            *DEFAULT_SERVER_OPTIONS,                              # keepalive, msg sizes
            ("grpc.max_connection_age_ms", 5 * 60 * 1000),       # GOAWAY every 5 min
            ("grpc.max_connection_age_grace_ms", 30 * 1000),     # in-flight stream grace
            ("grpc.max_connection_idle_ms", 60 * 1000),          # idle channel reaper
        ]
        server = aio.server(options=options)
        myservice_pb2_grpc.add_MyServiceServicer_to_server(MyServicer(), server)
        myservice_pb2_grpc.add_OtherServiceServicer_to_server(OtherServicer(), server)

        health_servicer = register_health(
            server,
            service_names=[
                "myservice.v1.MyService",
                "myservice.v1.OtherService",
            ],
        )
        server.add_insecure_port(f"[::]:{settings.port}")
        await _serve_with_graceful_shutdown(server, settings.port, health_servicer)


    async def _serve_with_graceful_shutdown(
        server: aio.Server, port: int, health_servicer: HealthServicer,
    ) -> None:
        stop_event = asyncio.Event()
        loop = asyncio.get_running_loop()
        for sig in (signal.SIGTERM, signal.SIGINT):
            loop.add_signal_handler(sig, stop_event.set)

        await server.start()
        logger.info("server_started", extra={"port": port})
        await stop_event.wait()

        # Flip Health to NOT_SERVING BEFORE server.stop() so L7 LBs and
        # Watch-subscribed clients see the drain signal immediately.
        health_servicer.enter_graceful_shutdown()
        logger.info("health_draining")
        await server.stop(grace=_SHUTDOWN_GRACE_SECONDS)
        logger.info("graceful_shutdown_done")


    def run() -> None:
        setup_telemetry(component="server-app")
        asyncio.run(_compose_and_serve())
    ```

12. **Multiple servicers on one server, one `[::]:port`.** Compose all related services on a single `aio.server` so they share connection / multiplexing / config. Different processes (e.g. unary RPCs vs streaming, public vs admin) get separate ports and separate `apps/` subdirectories.
13. **`loop.add_signal_handler(SIGTERM, stop_event.set)`** is the asyncio-native way to react to k8s pod termination. The default `server.wait_for_termination()` does NOT react to SIGTERM — wrap it.
14. **Health flip BEFORE `server.stop()` is non-negotiable.** L7 load balancers and `Watch`-subscribed clients see the drain signal via `grpc.health.v1.Health` immediately; without it, in-flight RPCs continue but new traffic keeps arriving until DNS/endpoint propagation catches up — seconds-to-minutes too late.
15. **`server.stop(grace=N)` cooperates with in-flight RPCs.** New streams are refused; existing ones get N seconds to finish. Match `_SHUTDOWN_GRACE_SECONDS ≥ max_connection_age_grace_ms + slack` so streams that the server itself is draining via GOAWAY have time to complete.

## Async Client: Shared Channel Factory

16. **One `grpc.aio.Channel` per process, shared by every stub against that target.** Channels are connection pools — cheap to call against, expensive to construct. Constructing per-call defeats HTTP/2 multiplexing and triggers TLS handshake storms.
17. The canonical channel factory:

    ```python
    import json
    import grpc

    SERVICE_CONFIG: str = json.dumps({
        "methodConfig": [{
            "name": [
                {"service": "myservice.v1.MyService", "method": "DoSomething"},
                {"service": "myservice.v1.MyService", "method": "GetThing"},
                {"service": "myservice.v1.OtherService", "method": "ListThings"},
            ],
            "retryPolicy": {
                "maxAttempts": 5,
                "initialBackoff": "0.1s",
                "maxBackoff": "2s",
                "backoffMultiplier": 2,
                "retryableStatusCodes": ["UNAVAILABLE"],
            },
        }],
    })


    def build_myservice_channel(
        address: str,
        *,
        extra_options: list[tuple[str, object]] | None = None,
    ) -> grpc.aio.Channel:
        options: list[tuple[str, object]] = [
            ("grpc.service_config", SERVICE_CONFIG),
            ("grpc.lb_policy_name", "round_robin"),
            ("grpc.enable_retries", 1),
            ("grpc.dns_min_time_between_resolutions_ms", 30_000),
        ]
        if extra_options:
            options.extend(extra_options)
        return grpc.aio.insecure_channel(address, options=options)
    ```

18. **`extra_options` parameter.** The factory module should not import settings — pass keepalive / message-size knobs from the caller (FastAPI lifespan or `serve()`). Keeps the module reusable and free of config dependencies.
19. **Always set a per-call `timeout=`.** Without one, network hangs propagate into your handler and exhaust your task pool. Pass `timeout: float` as a keyword-only required argument on every client wrapper method.
20. **Channel ownership belongs to the bootstrap layer**, not the client wrapper. FastAPI lifespan or `serve()` opens it; client wrapper classes take it as a constructor argument and never close it.

## Streaming Patterns

21. **Server streaming** — servicer yields, client iterates:
    ```python
    async def ListItems(self, request, context):
        async for item in self._fetch_items(request.user_id):
            yield item

    # Client
    async for item in stub.ListItems(request, timeout=30.0):
        process(item)
    ```
22. **Client streaming** — servicer takes async iterator, returns single response. Useful for upload-style RPCs.
23. **Bidirectional** — both sides are async iterators. Use cases: chat, real-time sync, long-poll replacement.
24. **Backpressure**: gRPC handles flow control over HTTP/2 automatically. If a slow consumer holds the producer, the producer's `yield` will eventually block (await). Don't build unbounded in-memory queues parallel to the stream — let the stream itself be the queue.
25. **Streaming RPCs are NOT auto-retried.** The `retryPolicy` in #17 applies only to unary RPCs (the `name` array is unary-only by design). For bidirectional streams, a `GOAWAY` mid-stream surfaces as `grpc.StatusCode.UNAVAILABLE` on the next read/write — application code must catch and decide: fail the operation, or reopen and retry from a known-safe point (with a fresh idempotency key if the operation could have partially committed).

## Configuration (Channel & Server Options)

26. Define channel and server options as constants in a config module. Keep them aligned across both sides — keepalive that the client expects must match what the server permits.

    ```python
    DEFAULT_SERVER_OPTIONS = [
        ("grpc.max_send_message_length", 16 * 1024 * 1024),
        ("grpc.max_receive_message_length", 16 * 1024 * 1024),
        ("grpc.keepalive_time_ms", 10_000),
        ("grpc.keepalive_timeout_ms", 5_000),
        ("grpc.keepalive_permit_without_calls", 1),
        ("grpc.http2.max_ping_strikes", 0),  # tolerate aggressive client pings
    ]

    DEFAULT_CHANNEL_OPTIONS = [
        ("grpc.max_send_message_length", 16 * 1024 * 1024),
        ("grpc.max_receive_message_length", 16 * 1024 * 1024),
        ("grpc.keepalive_time_ms", 10_000),
        ("grpc.keepalive_timeout_ms", 5_000),
        ("grpc.keepalive_permit_without_calls", 1),
        ("grpc.http2.max_pings_without_data", 0),
        ("grpc.http2.min_time_between_pings_ms", 10_000),
    ]
    ```

27. **The HPA-rebalance triad on the server** (`max_connection_age_*`) is what makes client-side LB actually rebalance after k8s scale-up. Without it, existing client subchannels never close, so newly-scaled-up pods receive zero traffic until something restarts.

    | Option | Value | What it does |
    |---|---|---|
    | `grpc.max_connection_age_ms` | `5 * 60 * 1000` | Server sends `GOAWAY` after 5 min; client opens fresh channel, re-resolves DNS, picks up new pods. |
    | `grpc.max_connection_age_grace_ms` | `30 * 1000` | After GOAWAY, in-flight streams have 30 s to finish. Must be ≥ longest expected stream duration. |
    | `grpc.max_connection_idle_ms` | `60 * 1000` | Channels with zero in-flight RPCs for 60 s close. Hygiene against zombie TCPs. |

28. **Default max message size is 4 MiB.** Larger payloads fail with `RESOURCE_EXHAUSTED`. Set explicit limits matching your domain — don't blindly raise to unlimited (`-1`).
29. **Compression**: `grpc.Compression.Gzip` for typical JSON-like payloads. Per-call override: `await stub.Method(req, compression=grpc.Compression.NoCompression)` — useful for already-compressed payloads.

## Cross-Cutting Concerns: Decorators (Canonical) vs Interceptors

The recommended pattern for logging, error mapping, and per-method context extraction is **decorators on servicer methods**, not `ServerInterceptor`. Both work; they trade off differently.

### Decorator pattern (canonical for cross-cutting + proto-aware)

30. Decorator stack on every unary RPC method, ordered intentionally:
    ```python
    @grpc_logger(extract_context=do_something_context)
    @grpc_error_handler(ERROR_MAP)
    async def DoSomething(self, request, context):
        ...
    ```
    - **`@grpc_logger` outermost** so it sees the gRPC status code AFTER `grpc_error_handler` has aborted the context. Logs business errors as `warning` (status code set), unhandled exceptions as `error`.
    - **`@grpc_error_handler` innermost** so it catches application exceptions and maps them to gRPC status + rich error before they propagate.

31. **Per-method `extract_context` callable** — each method has its own extractor that knows its proto fields:
    ```python
    def do_something_context(request) -> dict:
        return {
            "request_id": request.request_id,
            "user_id": request.user_id,
            "resource_id": request.resource_id,
            "amount": request.amount,
        }

    def get_thing_context(request) -> dict:
        return {
            "tenant_id": request.tenant_id,
            "thing_id": request.thing_id,
        }
    ```
    The logger binds these as structlog keys and as OTel span attributes (e.g. `myservice.{key}`). Single source of truth for "what fields matter for this method".

32. **Why decorators over `ServerInterceptor`** for these concerns:
    - The error decorator needs the project's domain exception types and `ERROR_MAP` — it's tied to the service's domain model, not generic.
    - The logger needs to know which proto fields are Tier-1 per method — a generic interceptor sees only the wire-level message.
    - Decorators stack visibly in code, making it obvious which method gets which behavior. `ServerInterceptor` chains are configured at server construction and easy to forget.

### `ServerInterceptor` (when wire-level access is needed)

33. Use `grpc.aio.ServerInterceptor.intercept_service()` when you need to inspect or mutate the request before it reaches the servicer — auth tokens in metadata, request-id propagation, before-handler tracing spans:
    ```python
    class AuthInterceptor(grpc.aio.ServerInterceptor):
        async def intercept_service(self, continuation, handler_call_details):
            metadata = dict(handler_call_details.invocation_metadata)
            token = metadata.get("authorization")
            if not token or not _verify_token(token):
                return _abort_handler(grpc.StatusCode.UNAUTHENTICATED, "Invalid token")
            return await continuation(handler_call_details)
    ```
    - **Always `await continuation`** — it's async; forgetting `await` returns a coroutine where a handler is expected.

34. **Use `contextvars` to pass data from interceptor to handler.** Setting attributes on `context` works but is fragile across multi-interceptor chains. `ContextVar` is asyncio-aware and propagates correctly across `await` boundaries within a single RPC. This is the canonical pattern from the official gRPC examples for context propagation.

### Client-side interceptors

35. Client interceptors come in **four base classes — one per RPC type** (`UnaryUnaryClientInterceptor`, `UnaryStreamClientInterceptor`, `StreamUnaryClientInterceptor`, `StreamStreamClientInterceptor`). Cross-cutting concerns (auth, tracing) usually subclass all four — even if the body is identical, the signatures differ. Pass them via `interceptors=[...]` to the channel constructor.

## Load Balancing

### The problem — why plain ClusterIP breaks gRPC LB

36. gRPC uses HTTP/2 and **multiplexes many streams over one long-lived TCP connection**. A k8s `ClusterIP` Service load-balances at L4 — picks a backend pod **at TCP connect time** and sticks for the connection's lifetime.

    Outcome with naive wiring (`ClusterIP` + single channel):
    - Client opens one TCP+HTTP/2 connection. kube-proxy picks one pod. All subsequent streams land on that pod forever.
    - HPA scales 2 → 6 pods. The four new pods receive zero traffic from existing clients — they only pick up new clients that reconnect.
    - Result: uneven load, broken HPA, scale-up that "works" only on rolling restart.

    This is HTTP/2 + L4 interaction, not a bug. Two remedies, applied **together**:
    - **Client-side LB** with `dns:///` + `round_robin` — client knows all backend IPs and rotates streams across them. Requires a **headless** k8s Service so DNS returns all pod IPs.
    - **Server-side connection recycling** via `max_connection_age_ms` (#27) — server forces clients to reconnect periodically, picking up newly-scaled-up pods.

### Solution

37. **Headless Service** alongside (not replacing) the existing ClusterIP — gives DNS-level access to all pod IPs:
    ```yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: myservice-headless              # alongside the regular ClusterIP service
      namespace: my-namespace
    spec:
      clusterIP: None                       # KEY DIFFERENCE — no VIP, DNS returns all pod IPs
      selector:
        app: myservice
      ports:
      - name: grpc
        port: 50051
        targetPort: 50051
      publishNotReadyAddresses: false
    ```
    Verify with `nslookup myservice-headless.my-namespace.svc.cluster.local` from any pod in the cluster — should return multiple A-records, one per Ready pod.

38. **Client target URI must use `dns:///` scheme.** Without it, gRPC treats the host as a single endpoint and client-side LB does nothing:
    ```python
    address = "dns:///myservice-headless.my-namespace.svc.cluster.local:50051"
    channel = build_myservice_channel(address, extra_options=DEFAULT_CHANNEL_OPTIONS)
    ```
    Expose via env var, e.g. `MYSERVICE_GRPC_TARGET=dns:///myservice-headless.my-namespace.svc.cluster.local:50051`.

39. **`lb_policy_name=round_robin` is not optional** — the gRPC default is `pick_first`, which would re-create the pinning problem even on a headless Service.

40. **`enable_retries=1` + `retryPolicy` on UNAVAILABLE** transparently re-routes RPCs racing with `GOAWAY` close windows to live subchannels. Without retries, those windows produce visible `UNAVAILABLE` errors on the caller every 5 minutes per pod.

41. **Available built-in policies**:
    - `pick_first` — connect sequentially, stick to first that works (default; pinning).
    - `round_robin` — distribute RPCs across all healthy addresses (k8s + headless + HPA standard).
    - `weighted_round_robin` — backend-reported weights (xDS).
    - `ring_hash` — consistent hashing for session affinity.
    - `xds` — full xDS protocol (Envoy/Istio control-plane integration).
    - `rls` — Route Lookup Service (Google-internal pattern for very large fleets).

42. **No service mesh by design.** The headless + `round_robin` + `max_connection_age` triad is the standard "no-mesh" answer. Adopting Istio/Linkerd just for this would be disproportionate (sidecar CPU/memory per pod, operator tax, mTLS surface area). If a mesh is adopted later, none of this breaks — headless becomes irrelevant, `lb_policy_name` becomes a no-op, `max_connection_age` becomes unnecessary, but the code keeps working.

43. **Don't try to do load balancing in your application code.** No round-robin loops over a list of stubs, no manual failover. The library handles connectivity state, subchannel management, and retry — circumventing it usually breaks deadlines and creates connection storms under failure.

## Kubernetes Deployment Patterns

44. **`terminationGracePeriodSeconds` budget** = `preStop sleep` + drain window + slack:
    ```yaml
    spec:
      template:
        spec:
          terminationGracePeriodSeconds: 90
          containers:
          - name: server
            lifecycle:
              preStop:
                exec:
                  command: ["/bin/sh", "-c", "sleep 15"]
    ```
    The `preStop sleep 15` lets DNS caches in clients clear before the process starts refusing streams. Then SIGTERM hits the process; `_serve_with_graceful_shutdown` (#11) flips Health to NOT_SERVING, then `server.stop(grace=35)` drains in-flight streams. Total: 15 + 35 = 50 s, comfortably under the 90 s grace period.

45. **`HealthServicer(experimental_non_blocking=True)`** — without this flag, Health calls run on the same thread pool as the rest of the server. Under heavy load this can starve health checks and cause flapping. The flag has been stable for years despite the name.

46. **Register full proto service names** (`package.Service`) — mesh sidecars and `grpc-health-probe` query by exact `.proto`-defined names, not module paths.

47. **HPA scale-up timing**: `dns_min_time_between_resolutions_ms=30000` + `max_connection_age_ms=300000` means new pods start receiving traffic within at most 30 s (DNS cycle) of being Ready. Faster on most failure events.

## Error Model

The pattern is two-sided: server packs domain errors into gRPC status + rich detail proto; client wrappers translate `grpc.RpcError` into a domain exception hierarchy so callers never see `RpcError`.

### Server side: error map + decorator

48. **Single error contract per domain** — one detail proto for all services in the domain:
    ```proto
    message MyServiceErrorDetail {
      string code = 1;          // business code: "INSUFFICIENT_FUNDS", "DUPLICATE_REQUEST", ...
      string message = 2;
      map<string, string> metadata = 3;
    }
    ```

49. **`ERROR_MAP: dict[type[Exception], tuple[int, str, str]]`** — exception class → (gRPC code, business code, default message):
    ```python
    from google.rpc import code_pb2

    ERROR_MAP: dict[type[Exception], tuple[int, str, str]] = {
        ConcurrencyError: (
            code_pb2.ABORTED, "RETRY_REQUIRED",
            "Stale anchor — re-read state and retry",
        ),
        BusinessRuleError: (
            code_pb2.FAILED_PRECONDITION, "BUSINESS_RULE_VIOLATED",
            "Operation violates a business rule",
        ),
        IdempotencyConflictError: (
            code_pb2.ALREADY_EXISTS, "DUPLICATE_REQUEST",
            "Request id reused for a different operation",
        ),
        ResourceNotFoundError: (
            code_pb2.NOT_FOUND, "NOT_FOUND",
            "Resource not found",
        ),
        InvalidArgumentError: (
            code_pb2.INVALID_ARGUMENT, "INVALID_ARGUMENT",
            "Invalid request",
        ),
    }
    ```

50. **Status code semantics**:

    | gRPC code | Use for |
    |---|---|
    | `INVALID_ARGUMENT` | Caller built a bad request (operator-actionable bug) |
    | `NOT_FOUND` | Missing resource |
    | `ALREADY_EXISTS` | Idempotency conflict (request id reused with different operation) |
    | `FAILED_PRECONDITION` | Business rule violated (insufficient balance, limit exceeded, configuration mismatch) |
    | `ABORTED` | Optimistic-concurrency / retry-required (e.g. stale anchor) |
    | `UNAUTHENTICATED` / `PERMISSION_DENIED` | AuthN / AuthZ failures |
    | `UNAVAILABLE` | Transient transport — the **only** auto-retryable status |
    | `INTERNAL` | Unexpected server bug; never used for bad client input |
    | `DEADLINE_EXCEEDED` | Server didn't finish in time; treated by client as transport failure |

51. **Pack rich detail via `rpc_status.to_status()`** + `context.abort_with_status()`:
    ```python
    from google.protobuf import any_pb2
    from google.rpc import code_pb2, status_pb2
    from grpc_status import rpc_status

    async def _abort(context, grpc_code, business_code, message):
        detail = any_pb2.Any()
        detail.Pack(MyServiceErrorDetail(code=business_code, message=message))
        rich = status_pb2.Status(code=grpc_code, message=message, details=[detail])
        await context.abort_with_status(rpc_status.to_status(rich))
    ```

52. **Don't return error info in success responses.** A success response with `error_code: 5` field is an anti-pattern — gRPC status carries this, and clients shouldn't have to inspect both.

### Client side: translation hierarchy

53. **Every `grpc.RpcError` gets translated** at the client wrapper boundary. Orchestrators (FastAPI handlers, business services) never see `RpcError` — that's the invariant.

54. **Per-domain exception hierarchy** — base class + transport / argument / conflict / unknown subclasses:
    ```python
    class MyServiceError(Exception):
        def __init__(self, message: str = "", metadata: dict[str, str] | None = None) -> None:
            super().__init__(message)
            self.message = message
            self.metadata: dict[str, str] = dict(metadata or {})

    class MyServiceUnavailableError(MyServiceError):
        """UNAVAILABLE / DEADLINE_EXCEEDED — transport failure after retries exhausted."""

    class MyServiceInvalidArgumentError(MyServiceError):
        """INVALID_ARGUMENT — caller built a bad request; operator-actionable bug."""

    class MyServiceConflictError(MyServiceError):
        """ALREADY_EXISTS / DUPLICATE_REQUEST — idempotency conflict."""

    class MyServiceUnknownError(MyServiceError):
        """Unexpected status (INTERNAL / CANCELLED / etc.)."""
    ```

55. **Translation function pulls the rich detail via `rpc_status.from_call()`**:
    ```python
    def _extract_detail(rpc_error: grpc.RpcError) -> MyServiceErrorDetail:
        rich: status_pb2.Status | None = rpc_status.from_call(rpc_error)
        if rich is None:
            return MyServiceErrorDetail(code="UNKNOWN", message=rpc_error.details() or "")
        for detail in rich.details:
            if detail.Is(MyServiceErrorDetail.DESCRIPTOR):
                wd = MyServiceErrorDetail()
                detail.Unpack(wd)
                return wd
        return MyServiceErrorDetail(code="UNKNOWN", message=rpc_error.details() or "")


    def _translate_error(rpc_error: grpc.RpcError) -> MyServiceError:
        code = rpc_error.code()
        if code in (grpc.StatusCode.UNAVAILABLE, grpc.StatusCode.DEADLINE_EXCEEDED):
            return MyServiceUnavailableError(rpc_error.details() or "transport failure")
        if code == grpc.StatusCode.INVALID_ARGUMENT:
            detail = _extract_detail(rpc_error)
            return MyServiceInvalidArgumentError(detail.message, metadata=dict(detail.metadata))
        if code == grpc.StatusCode.ALREADY_EXISTS:
            detail = _extract_detail(rpc_error)
            if detail.code == "DUPLICATE_REQUEST":
                return MyServiceConflictError(detail.message, metadata=dict(detail.metadata))
        detail = _extract_detail(rpc_error)
        return MyServiceUnknownError(
            f"unhandled error: grpc_code={code.name}, business_code={detail.code}",
            metadata=dict(detail.metadata),
        )
    ```

56. **Thin client wrapper class** owns the stub and translation:
    ```python
    class MyServiceClient:
        def __init__(self, channel: grpc.aio.Channel) -> None:
            self._stub = myservice_pb2_grpc.MyServiceStub(channel)

        async def do_something(
            self,
            request: myservice_pb2.DoSomethingRequest,
            *,
            timeout: float,
        ) -> myservice_pb2.DoSomethingResponse:
            try:
                return await self._stub.DoSomething(request, timeout=timeout)
            except grpc.RpcError as exc:
                raise _translate_error(exc) from exc
    ```

57. **UNAVAILABLE-only retry principle.** Design APIs so that **transport bounce is the only safe auto-retry case**; business errors must reach the caller. This is why the service config `retryableStatusCodes` lists only `["UNAVAILABLE"]` — never `INVALID_ARGUMENT` (won't change), `FAILED_PRECONDITION` (caller must decide), or `ALREADY_EXISTS` (idempotency conflict, caller dispatches).

## Logging

58. **Two log lines per RPC** (request on entry, response on exit), via the `@grpc_logger` decorator:
    ```python
    bound = log.bind(**ctx)            # ctx from extract_context callable
    bound.info(f"{method}.request | {text_format.MessageToString(request, as_one_line=True)}")
    # ... handler ...
    bound.info(f"{method}.response | {text_format.MessageToString(result, as_one_line=True)}",
               status="ok", duration_ms=duration_ms)
    ```
    Proto payload via `text_format.MessageToString(msg, as_one_line=True)` for human-readable output.

59. **Log level by error class** in the response line:
    - `info` for ok responses
    - `warning` for known business errors (gRPC code is set on context after `grpc_error_handler` aborted)
    - `error` for unhandled exceptions (no gRPC code; `INTERNAL_ERROR`)

60. **Tier-1 fields go as structlog `extra` and OTel span attributes** — not in the message text. The `extract_context(request)` callable returns the dict; the logger binds it both to log events and to the active span (e.g. `span.set_attribute(f"myservice.{key}", str(value))`).

## Deadlines, Keepalive, Retries

61. **Always set a deadline** on outbound calls (`timeout=5.0`). Without one, network hangs propagate into your handler.
62. **Propagate deadlines across calls.** When `serviceA.Foo` calls `serviceB.Bar`, pass the remaining deadline so B doesn't keep working after A's caller gave up:
    ```python
    deadline = context.time_remaining()
    await b_stub.Bar(req, timeout=deadline)
    ```
63. **Retries via service config**, not custom code. The `retryPolicy` in #17 covers transient errors. Don't retry on `INVALID_ARGUMENT` or `PERMISSION_DENIED` — those won't change. Don't retry streaming RPCs — the policy applies only to unary, by design.
64. **Cumulative retry backoff eats into the deadline**, doesn't extend it. With `maxAttempts=5, initialBackoff=0.1s, backoffMultiplier=2, maxBackoff=2s` the worst case wait is 0.1+0.2+0.4+0.8+1.6 = ~3.1 s of pure backoff; if the per-call deadline is 5 s and each attempt itself takes time, you may run out of deadline before all retries fire. Tune `maxAttempts` accordingly.

## Authentication & TLS

65. **Internal cluster traffic without mesh: insecure channels are acceptable** if the network boundary is trusted (k8s NetworkPolicies, single VPC). Use `grpc.aio.insecure_channel()` for service-to-service inside the cluster; TLS is added at the ingress/edge.
66. **External-facing services need TLS.** `grpc.ssl_channel_credentials()` (client) / `grpc.ssl_server_credentials([(key, cert)])` (server). For mTLS, add `root_certificates=` and `require_client_auth=True` on the server.
67. **Token auth via metadata in a client interceptor** (`UnaryUnaryClientInterceptor`). Keep tokens out of `.proto` schemas — they're transport-layer, not application-layer.

## Observability

68. **Health service** is required (#11, #45-46). k8s liveness probes use `grpc-health-probe`; L7 LBs and mesh sidecars subscribe to `Watch`.
69. **Reflection** (`grpc_reflection.v1alpha.reflection.enable_server_reflection`) — useful in dev/staging for `grpcurl` introspection. Disable in production.
70. **OpenTelemetry** via `opentelemetry-instrumentation-grpc` — provides `GrpcAioInstrumentorClient` and `GrpcAioInstrumentorServer`, registers at `setup_telemetry(component=...)` startup. Spans propagate via metadata automatically; the `@grpc_logger` decorator additionally attaches Tier-1 fields as `<service>.{key}` span attributes.
71. **Verification of LB after rollout**:
    - `nslookup` returns multiple A-records.
    - `kubectl top pods` CPU within ~20% across pods after warm-up.
    - OTel duration metric p50 should match across pods (mismatch = broken balancing).
    - HPA scale-up test: bump replicas, wait 60 s, new pods should show non-zero CPU.

## Testing

72. **Real server in tests**, not mocks. Spin up an actual `grpc.aio.server()` bound to a random port (`server.add_insecure_port("[::]:0")` returns the assigned port); clients connect to it. Catches wire-format issues, decorator ordering bugs, and codec failures that mock-stubs hide.
73. For the testing workflow (pytest-asyncio fixtures, `db_session` patterns, factory_boy), defer to the `pytest-best-practices` skill.
74. **Test the decorator chain end-to-end.** Construct the production decorator order on a real servicer method, invoke via real RPC, assert both the response and the log lines. Unit-testing each decorator in isolation usually misses interaction with `continuation` / `context.abort_with_status()`.

## When applying these rules

- **Be opinionated about footguns** (#13 SIGTERM signal handler, #14 health flip BEFORE server.stop, #16 channel reuse, #19 always set timeout, #25 streams not retried, #27 max_connection_age triad, #28 default 4 MiB limit, #38 dns:/// scheme is mandatory, #39 round_robin not optional, #43 don't DIY load balancing, #53 orchestrators never see RpcError, #57 UNAVAILABLE-only retries) — these cause fleet-wide silent failures, hangs, broken HPA, or business errors swallowed as transport bugs.
- **Be flexible about layout** (#1-4 multi-app structure, #26 specific channel/server option values) — match the existing project's convention. The values shown are tested defaults; tune to traffic profile and payload shape.
- **Read existing code first.** If the project has `shared/grpc_tools/` with `grpc_error_handler.py`, `grpc_logger.py`, `health.py` — conform to them. New services adopt the existing decorator and channel-factory patterns, not reinvent.
- **Cross-skill awareness:**
  - `pytest-best-practices` for test fixtures (`pytest-asyncio`, real DB, factory_boy).
  - `fastapi-best-practices` for services that expose both gRPC and REST surfaces — channel ownership in FastAPI lifespan (`app.state`), translation from domain exceptions to HTTP responses in middleware.
  - `sqlalchemy-best-practices` when servicer methods touch the DB — `expire_on_commit=False`, async session per request, `selectinload` for relationships.
