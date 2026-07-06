# 20: Station WS client library

## Summary

`web/src/lib/ws.ts`: the typed, auto-reconnecting WebSocket client that owns
the station's live link — auth handshake, heartbeat, backoff, resync events
— feeding the app store.

## Context

DESIGN §7.3 fixes behavior; §6.8 the wire contract (types from
`lib/protocol.ts`, issue 19). Every screen renders from store state derived
from this client; its correctness defines perceived reliability.

## Scope

- `lib/ws.ts` + its vitest suite
- Store integration events (connection state signal, message dispatch)

## Detailed Requirements

1. Public surface:

   ```ts
   type ConnState = 'idle'|'connecting'|'authing'|'online'|'reconnecting'|'superseded'|'revoked';
   class StationSocket {
     constructor(opts: { getToken(): string|null,
                         onMessage(m: ServerMessage): void,
                         onState(s: ConnState): void });
     start(): void; stop(): void;
     send(m: ClientMessage): boolean;  // false when not online (caller decides UX)
   }
   ```
2. Connect to `wss://${location.host}/api/v1/ws`; on open send
   `auth {token}` immediately; treat `auth.ok` as transition to `online`
   (deliver it as a message too — store consumes roster/active_call/settings
   from it); right after `auth.ok`, automatically send `station.status`
   with the current mic state (provided via an injected `getMicState()`
   option — keeps issue 21/23 decoupled).
3. Heartbeat: `ping {t: Date.now()}` every 25 s while online; watchdog: if
   no inbound frame for 40 s → force close + reconnect path.
4. Reconnect: backoff table `[1,2,4,8,15,30]` s + full jitter
   (`delay = random(0, base)`), reset on successful auth; infinite retries;
   `onState('reconnecting')` after the first failure (UI shows the banner
   after 5 s — banner timing is UI's concern, state emission is immediate).
   On `visibilitychange → visible` and on `online` browser event: if not
   open, retry immediately (reset current backoff step).
5. Terminal states: close 4401 or `auth.err` → wipe token via storage,
   state `revoked`, stop (no retry). Close 4409 → state `superseded`, stop
   (screen from issue 22 offers a "use here instead" button that calls
   `start()` again). Server 1001 → normal reconnect path.
6. Envelope handling: parse+validate against protocol types (unknown type →
   console.warn + ignore — forward compatibility); outbound `seq` counter
   attached to every C→S message; surface `error` messages to `onMessage`
   like any other (store maps them to UX).
7. No DOM/global access besides `location`, `WebSocket`, timers, visibility
   — all injectable for tests (constructor `opts.wsFactory?`, `opts.now?`,
   timers via injected scheduler or vitest fake timers).
8. Single instance per app enforced by module-level guard (dev warning).

## Acceptance Criteria

- [ ] Happy path (mock server): start → auth sent → `auth.ok` → `online`,
      mic status auto-sent, messages dispatched in order.
- [ ] Heartbeat: fake timers show ping cadence 25 s; 40 s silence triggers
      reconnect; pong resets watchdog.
- [ ] Backoff sequence matches the table with jitter bounds (statistical
      assertion over 100 runs with seeded RNG injection); resets after
      success.
- [ ] 4401 → token cleared + `revoked`, no further connection attempts;
      4409 → `superseded`, no retries; explicit `start()` reconnects.
- [ ] `send()` while offline returns false and sends nothing.
- [ ] Unknown server message type ignored without state corruption.
- [ ] Type-safe: `send({type:'call.invite', payload:{callee_id}})` compiles;
      wrong payload shape fails `tsc` (compile-time test file).

## Validation

`npm test` (vitest, fake timers, in-memory mock WS server helper committed
under `web/src/test/`); coverage of `ws.ts` ≥ 90% lines.

## Dependencies

19.

## Non-goals

Rendering (21/22/24), WebRTC (23), server behavior changes.

## Design References

DESIGN.md §6.8, §7.3; issue 17 (reference choreography).
