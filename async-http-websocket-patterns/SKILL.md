---
name: async-http-websocket-patterns
description: Production async HTTP + WebSocket client patterns including rate limiting, exponential backoff, heartbeat/pong handling, and reconnection logic. Use when building any real-time API integration, when user says "websocket client", "API client", "rate limiting", or needs resilient async networking.
metadata:
  author: niyaznurbhasha
  version: 1.0.0
  category: infrastructure
  tags: [async, http, websocket, rate-limiting, retry, backoff, python]
---

# Async HTTP + WebSocket Patterns

Production-grade patterns for async HTTP clients with rate limiting and retry, plus WebSocket listeners with automatic reconnection and heartbeat handling.

## When to Use

- Integrating with any REST API that has rate limits
- Building a WebSocket client that must stay connected 24/7
- Need exponential backoff on transient failures (429, 5xx)
- Consuming real-time data feeds (market data, chat, notifications)

## Part 1: Async HTTP Client with Rate Limiting and Retry

### Step 1: Structure the client as an async context manager

```python
from typing import Any, Optional
import httpx
from aiolimiter import AsyncLimiter

class APIClient:
    """Async HTTP client with rate limiting and retry. Use as async context manager."""

    def __init__(
        self,
        base_url: str,
        rate_limit_rps: int = 20,
        auth: Optional[httpx.Auth] = None,
    ) -> None:
        self._base_url = base_url.rstrip("/")
        self._auth = auth
        self._limiter = AsyncLimiter(rate_limit_rps, 1.0)
        self._client: Optional[httpx.AsyncClient] = None

    async def __aenter__(self) -> "APIClient":
        self._client = httpx.AsyncClient(
            base_url=self._base_url,
            auth=self._auth,
            timeout=httpx.Timeout(10.0, connect=5.0),
            headers={"Content-Type": "application/json"},
        )
        return self

    async def __aexit__(self, *_: Any) -> None:
        if self._client:
            await self._client.aclose()
            self._client = None
```

### Step 2: Add retry with exponential backoff

Retry on transient errors (429, 5xx, transport errors). Use tenacity for clean retry logic.

```python
from tenacity import AsyncRetrying, retry_if_exception, stop_after_attempt, wait_exponential

async def _get(self, path: str, params: Optional[dict] = None) -> dict:
    assert self._client is not None, "Use as async context manager"

    def _is_retryable(exc: BaseException) -> bool:
        if isinstance(exc, httpx.TransportError):
            return True
        if isinstance(exc, httpx.HTTPStatusError):
            return exc.response.status_code in {429, 500, 502, 503, 504}
        return False

    async for attempt in AsyncRetrying(
        retry=retry_if_exception(_is_retryable),
        stop=stop_after_attempt(5),
        wait=wait_exponential(multiplier=1, min=1, max=30),
        reraise=True,
    ):
        with attempt:
            async with self._limiter:
                resp = await self._client.get(path, params=params)
            if resp.status_code in {429, 500, 502, 503, 504}:
                log.warning("api_retry", path=path, status=resp.status_code)
                resp.raise_for_status()
            if not resp.is_success:
                log.error("api_error", path=path, status=resp.status_code, body=resp.text[:200])
                resp.raise_for_status()
            return resp.json()

    raise RuntimeError(f"Exhausted retries for {path}")
```

### Step 3: Implement cursor-based pagination

```python
from typing import AsyncIterator

async def iter_items(self, path: str, limit: int = 200) -> AsyncIterator[dict]:
    """Paginate through all items at the given endpoint."""
    cursor: Optional[str] = None
    while True:
        params: dict[str, Any] = {"limit": limit}
        if cursor:
            params["cursor"] = cursor
        data = await self._get(path, params=params)
        items = data.get("items") or []
        for item in items:
            yield item
        cursor = data.get("cursor") or ""
        if not cursor:
            break
```

## Part 2: WebSocket Listener with Reconnection

### Step 4: Build the reconnecting WebSocket listener

The key pattern: a `run()` method with an outer reconnection loop and an inner message-processing loop.

```python
import asyncio
import json
from typing import Callable
from websockets.exceptions import ConnectionClosed

OnMessageCB = Callable[[dict], None]

class WSListener:
    """
    WebSocket listener with automatic reconnection.
    Reconnects with exponential backoff up to MAX_RECONNECT_DELAY.
    """

    _MIN_RECONNECT_DELAY = 1.0
    _MAX_RECONNECT_DELAY = 60.0

    def __init__(
        self,
        ws_url: str,
        channels: list[str],
        on_message: OnMessageCB,
        auth_headers: Optional[dict] = None,
    ) -> None:
        self._ws_url = ws_url
        self._channels = channels
        self._on_message = on_message
        self._auth_headers = auth_headers or {}

    async def run(self) -> None:
        """Main loop: connect, subscribe, process, reconnect on error."""
        delay = self._MIN_RECONNECT_DELAY
        while True:
            connect_start = asyncio.get_event_loop().time()
            try:
                await self._connect_and_listen()
                delay = self._MIN_RECONNECT_DELAY  # reset on clean exit
            except asyncio.CancelledError:
                raise
            except Exception as exc:
                connected_for = asyncio.get_event_loop().time() - connect_start
                # Reset backoff if session was stable (>30s)
                if connected_for > 30:
                    delay = self._MIN_RECONNECT_DELAY
                log.warning(
                    "ws_disconnected",
                    error=str(exc),
                    reconnect_in=delay,
                    session_secs=connected_for,
                )
                await asyncio.sleep(delay)
                delay = min(delay * 2, self._MAX_RECONNECT_DELAY)
```

### Step 5: Implement connect, subscribe, and message handling

```python
import websockets

async def _connect_and_listen(self) -> None:
    log.info("ws_connecting", url=self._ws_url)

    async with websockets.connect(
        self._ws_url,
        additional_headers=self._auth_headers,
        ping_interval=20,    # Send ping every 20s
        ping_timeout=10,     # Wait 10s for pong
        compression=None,    # Disable for lower latency
    ) as ws:
        log.info("ws_connected")
        await self._subscribe(ws)
        async for raw in ws:
            self._handle_message(raw)

async def _subscribe(self, ws) -> None:
    msg = json.dumps({
        "cmd": "subscribe",
        "params": {"channels": self._channels},
    })
    await ws.send(msg)
    log.info("ws_subscribed", channels=self._channels)

def _handle_message(self, raw: str) -> None:
    try:
        msg = json.loads(raw)
    except json.JSONDecodeError:
        log.warning("ws_bad_json", raw=raw[:100])
        return

    msg_type = msg.get("type", "")

    if msg_type == "error":
        log.warning("ws_error", msg=str(msg)[:500])
    elif msg_type in {"subscribed", "ok", "pong"}:
        log.debug("ws_ctrl_msg", type=msg_type)
    else:
        self._on_message(msg)
```

### Step 6: Add sequence gap detection (for ordered streams)

```python
def _handle_ordered_message(self, msg: dict) -> None:
    """Detect sequence gaps in ordered message streams."""
    channel = msg.get("channel", "")
    seq = msg.get("seq")
    if seq is not None:
        expected = self._seqs.get(channel, seq)
        if seq != expected and seq != expected + 1:
            log.warning("ws_seq_gap", channel=channel, expected=expected, got=seq)
        self._seqs[channel] = seq
    self._on_message(msg)
```

## Part 3: Combining Both in a Runner

### Step 7: Orchestrate REST polling + WebSocket in one event loop

```python
class DataCollector:
    async def run(self) -> None:
        stop_event = asyncio.Event()

        # Handle shutdown signals
        loop = asyncio.get_running_loop()
        for sig in (signal.SIGINT, signal.SIGTERM):
            loop.add_signal_handler(sig, stop_event.set)

        tasks = [
            asyncio.create_task(self._rest_poll_loop(), name="rest_poll"),
            asyncio.create_task(self._ws_loop(), name="ws_listener"),
        ]

        # Crash one task = shut down everything
        def _on_crash(task: asyncio.Task) -> None:
            if not task.cancelled() and task.exception():
                log.error("task_crashed", task=task.get_name(), error=str(task.exception()))
                stop_event.set()

        for task in tasks:
            task.add_done_callback(_on_crash)

        try:
            await stop_event.wait()
        finally:
            for task in tasks:
                task.cancel()
            await asyncio.gather(*tasks, return_exceptions=True)
```

## Key Principles

- **Rate limit at the client level**: Use `aiolimiter.AsyncLimiter` wrapping every outbound request. Never rely on server-side rate limit headers alone.
- **Retry only transient errors**: 429, 5xx, and transport errors. Never retry 4xx (except 429).
- **Reset backoff after stable sessions**: If the WebSocket stayed connected for 30+ seconds, reset the backoff delay. Only escalate backoff for rapid repeated failures.
- **Always re-raise CancelledError**: Never swallow `asyncio.CancelledError` in your catch blocks. This is how graceful shutdown works.
- **Ping/pong for liveness**: Set `ping_interval` and `ping_timeout` on the WebSocket connection so dead connections are detected quickly.
- **Callable ticker lists**: Pass a callable (not a static list) to the WebSocket listener so it can refresh subscriptions on reconnect.

## Dependencies

```
httpx
aiolimiter
tenacity
websockets
```
