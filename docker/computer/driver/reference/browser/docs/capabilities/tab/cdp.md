# Tab Capability: cdp

```ts
const capability = await tab.capabilities.get("cdp");

type CdpEventsOptions = {
  afterSequence?: number; // Return events after this cursor; omit to start at the current position.
  limit?: number; // Maximum number of events to return, from 1 to 1000.
  methods?: Array<string>; // Return only these CDP event methods; must not be empty.
  target?: CdpTarget; // Filter by child target, including buffered events after it detaches.
  timeoutMs?: number; // Wait up to this many milliseconds for the first match.
};

type CdpCommandParams = Record<string, unknown>;

type CdpSendOptions = {
  target?: CdpTarget; // An attached child target; omit to send the command to the tab itself.
  timeoutMs?: number; // Maximum command wait in milliseconds.
};

type CdpTarget = { sessionId: string; targetId?: never } | { sessionId?: never; targetId: string };

interface CdpTabCapability {
  readEvents(options?: CdpEventsOptions): Promise<{ cursor: number; events: Array<{ method: string; params?: Record<string, unknown>; sequence: number; source: { extensionId?: string; sessionId?: string; tabId?: number; targetId?: string } }>; hasMore: boolean; truncated: boolean }>; // Read buffered CDP events, optionally waiting for the first match.
  send(method: string, params?: CdpCommandParams, options?: CdpSendOptions): Promise<unknown>; // Send a permitted CDP command to this tab or an attached child target.
}
```
