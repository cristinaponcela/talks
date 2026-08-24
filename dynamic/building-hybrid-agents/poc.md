My actual thought process coming up with this talk :)

leaving notes for myself here:

Slide 1:
Title only: Building Hybrid Agents

Slide 2:
Short intro: who am I, what do I do.
Learned to code with AI.
Work at Earendil, where I mostly have been working on Lefos and Pi (from Mario Zechner).

Slide 3:
Part I: Cloud vs Local

Slide 4:
Explain the context: there has been a lot of talk about cloud and local agents.

Slide 5:
Local agents: OpenClaw and Hermes Agent. Runtime lives on your device, but tokens are still sourced from an upstream cloud provider.

Slide 6:
Truly local.

Slide 7:
Llama.cpp Meta-Llama-3.1-8B-Instruct-Q4_K_M.gguf, pi example

Slide 8:
Problems with only local
- not there yet in terms of latency or performance, though very promising.
- GPUs/hardware?
- tailscale is now widely used for its tailnet to allow discoverability of agents living on a machine behind a nat
- Residential IP

Slide 9:
Cloud Lefos

Slide 10:
Problems with only cloud
- I pay 2.5k€ every year for the newest computer as a developer. Why do I also have to pay some monthly subscription for a cloud server service that runs an agentic runtime or loop when I can do that locally anyways?
- Privacy: I'd rather have my pi sessions living on my local disk, even though I'm still streaming my conversations to some token provider whose ZDR may or may not actually keep my data.
- Control: I decide the sandbox, the tools, the files and directories my agent can access, etc

Slide 11:
Residential IP limitation example
IMessage ToS on hardware and why Mac Minis

Slide 12:
Hybrid agents: briefly what I mean, loose example of what could live where.
Why the previous are not what I mean.

Slide 13:
Part II: Decomposing the agent

Slide 14:
What is an agent?

Slide 15:
Diagram:
model, harness, storage, server, client, ...

Slide 16:
Model
Local vs cloud, but via something like pi-ai we don't need to know as a client. Just make a request to `provider-x/model-y` and we should request to what is loaded.

Slide 17:
Example
For instance, llama.cpp example uses localhost "server" so we can communicate between the Hugging Face .gguf model file and the process in which the agent loop is actually turning. For the cloud ones, we call their SDKs.

Slide 18:
Harness
Colin's post: system prompt, tools, choice of sandbox, extensions, etc.
What actually drives the model - the agent loop.

Slide 19:
Example

Slide 20:
Runtime / environment
Where the actual tool calls and side effects occur

Slide 21:
Example of how this is abstracted in pi

Slide 22:
Storage
Transcripts
Stuff like encrypted reasoning and why local and cloud don't play on the same level field.

Slide 23:
Example: jsonl vs sqlite sessions in pi, encrypted reasoning error

Slide 24:
Server

Slide 25:
Control plane
Coordinator, "broker service" for discovery

Slide 26:
Client: tui, Electron desktop app, mobile app, web, mcp?
Unix socket vs ws

Slide 27:
Example

Slide 28:
Part III: Composing the agent

Slide 29:
Each component of the above should be able to live anywhere, arbitrarily, and our agent should be designed in such a way that this is abstracted.

Slide 30:
How to abstract FileSystem, etc 1 by 1 to show the other stuff
We attach to sessions, etc

Slide 31:
Example: Search API (https://github.com/earendil-works/pi/pull/7797/changes)

Slide 32:
Example of early on "orchestrator" or "server" process
Show IPC + RPC, how we could spawn a session remotely
We register pis and machines on a server service we own for discover

Slide 33:
Lessons learned: Lefos, why we need hybrid

Slide 34:
The goal is control, and choice. This is why I like OSS

Slide 35:
Hybrid enables this by design, if truly hybrid. Bc it forces everything to be composible, so you choose what lives where.



part 1: introduce myself
part 2: what we mean by hybrid agents: local vs cloud. also to mention problems with existing agent stuff, e.g. how tailscale is now widely used for its tailnet to allow discoverability of agents living on a machine behind a nat, etc
part 3: examples of "hybrid" agents, and why that's not what i mean
part 4: breaking down the agent. largest chunk of talk, i explain what each part is, how it can be broken down to have components living in arbitrarily different places, how we can then use that to compose part 5: examples: i have been working hard on the search api for pi (see https://github.com/earendil-works/pi/pull/7797/changes), and we can give some real code examples of this and a basic overview of storage in pi to show this level of choice, abstractedness and composability. then i also want to talk about my initial experiment with the server - unix socket, ipc command, we register pis and machines on a server service we own for discover, use the rpc command mode with the new harness, and we could spawn pis.
part 6: lefos: our early experiment of a cloud agent, and why we found we need hybrid


Search composable API code snippets:

1:

```ts
export interface SessionSearchOptions {
  /** Restrict results to specific canonical entry types. */
  readonly entryTypes?: readonly Entry["type"][];

  /** Maximum number of hits to return. */
  readonly limit?: number;

  /** Cancellation, e.g. search-as-you-type. */
  readonly signal?: AbortSignal;
}

export interface SessionSearchHit {
  /** Session owning the entry. */
  readonly sessionId: string;

  /** Entry inside that session. */
  readonly entryId: string;
}

export interface SessionSearch<T extends SessionSearchHit = SessionSearchHit> {
  search(text: string, options?: SessionSearchOptions): AsyncIterable<T>;
}
```

2:

Extends to scanning for in-memory storage, and JSONL:

```ts
export type ScanningReadable<TMetadata extends SessionMetadata = SessionMetadata> =
  Pick<SessionStorage<TMetadata>, "getMetadata" | "findEntries" | "getLabel">;

export interface ScanningSessionSearchHit extends SessionSearchHit {
  readonly timestamp: number;
  readonly snippet: string;
}
```

```ts
// Instead of:
// repo.search({ text: "Warsaw" })

// We do:
const search: SessionSearch = createScanningSessionSearch([sessionA, sessionB]);

for await (const hit of search.search("Warsaw", { limit: 10 })) {
  // portable identity
  hit.sessionId;
  hit.entryId;
}
```
