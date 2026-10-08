# Spring AI Inspector

A live web UI that shows what each demo sends to the model, in three layers:

1. **Your app sent**: the prompt exactly as the `ChatClient` call wrote it.
2. **After the advisors**: the prompt after memory, RAG, guardrails and other advisors ran. Messages the advisors added or removed are highlighted.
3. **On the wire**: every raw HTTP round-trip to the model, with request and response JSON, token usage and timing. Anthropic, OpenAI, Ollama, Mistral and DeepSeek traffic is rendered as one readable conversation. TypeSafe Jev
   `systemOne` calls (guardrails, judges, RAG filters) get a questions-and-answers view with the answer probabilities. Messages re-sent from an earlier round-trip are marked `re-sent`, new ones `new`. API keys are redacted.
   What a tool did while it ran is shown inside its card: the round-trips it made (e.g. the systemOne check of a Jev tool
   search) and its vector store adds and searches. These appear where they ran, with the embedding round-trips they
   made inside them and, for a search, its hits, each opening the search in the Retrieval step.

Nested `ChatClient` calls (for example sub-agents) are shown inside the call that triggered them. Tool executions
(including MCP tools) appear between the round-trips that requested and consumed them, with arguments, result,
errors and duration. MCP tools are tagged with the MCP connection and server they come from (an
`MCP · <connection>` badge, and a lane per MCP connection in the sequence view). The origins are learned from each
connection's `tools/list` responses, which covers every MCP tool callback provider, including ones built by hand, and
from the auto-configured `McpToolNamePrefixGenerator`, which knows the exact names of tools it renames on a clash
(`alt_<n>_<name>`). An application tool with the same name as an MCP tool is told apart by its description.

The MCP messages themselves are recorded too, by wrapping the MCP client transports Spring AI auto-configures (stdio,
Streamable HTTP, SSE): every JSON-RPC message in both directions, serialized off the I/O threads, with long strings cut.
Messages exchanged while an MCP tool runs (the `tools/call`, the server's log and progress notifications, its sampling
requests and the client's answers) are listed on that tool: a `tools/call` and its response are matched to the tool
run by tool name and request id, what the server sends in between goes to the connection's latest tool run (MCP does
not tie it to a request at this level), so parallel calls on one connection can mix logs. Logs and sampling requests also appear as notes on the tool's
sequence lane, with the `ChatClient` call that answers a sampling request on an "MCP sampling" lane. The rest (`initialize`, `tools/list`, ...) is shown per
connection in the **MCP connections** panel, with the server's name, version, protocol and tools.

## Images, speech, transcription and moderation

Non-chat calls that go through the proxy get a view of their own (23.2, 23.3), recognized by the API's path whoever
serves it: **image generation** (`/v1/images/generations`, `/edits`, `/variations`: the prompt and parameters, the
images, the revised prompt, the usage), **text to speech** (`/v1/audio/speech`: the text and voice, the audio
with a player), **transcription** (`/v1/audio/transcriptions`, `/translations`: the uploaded file, the
transcript, language and duration) and **moderation** (`/v1/moderations`: each input with its verdict and
top category scores). Images, PDFs and audio sent inline in chat messages (vision, audio input: 23.1, 23.4) are
shown in the message too, and so are a chat model's audio answer (`gpt-audio`, 23.4) and an image a Responses
call made with the hosted image tool. A chat model that answers with an image through an SDK the proxy never
sees (Gemini's image models, 23.5) is shown from the ChatClient advisors: the starter uploads the media of the
messages and generations once per item, so the drawn image and the edit requested with it are both previewed.
(Demo 23.5 runs on Spring AI 2.1.0-SNAPSHOT: 2.1.0-M1 drops the image parts of a Gemini answer.)

The same views show the calls of `ImageModel`, `TextToSpeechModel`, `TranscriptionModel` and `ModerationModel`
beans that make no HTTP round-trip the proxy sees (a model in the JVM, Google GenAI or Bedrock through their
SDKs), marked "no HTTP": the starter reports them from the beans, with their media uploaded for previews.

The media itself (inline base64 of a recognized type, binary bodies, uploaded files) is kept out of the
events, in memory under `spring.ai.inspector.max-blob-bytes`, and served to the UI from `/api/blobs/<id>`; the
events carry a `<base64 N chars TYPE blob:ID>` marker where the payload was. Long base64 that is not media (an
embedding vector in base64, a signature) is cut to the marker only. Audio a chat model streams back in small
deltas (`gpt-audio` with `stream`) is put together into one playable clip (raw PCM gets a WAV header). An
export carries the blobs the inspector still holds, so an imported run has its previews; a blob goes when the
last event pointing at it has been dropped from the log.

## RAG and memory

- **Retrieval (RAG)** step (05, 05-1): the configured pipeline stages, read from the `QuestionAnswerAdvisor` /
  `RetrievalAugmentationAdvisor`; every vector search with its query and scored hits; the funnel
  *queries → hits → unique → in the prompt*; the documents that reached the prompt (with Jev rerank scores and
  classifications) and the ones dropped by joining or post-processing. Ingestion shows as one line at the start of the
  run. Vector stores are observed by wrapping `VectorStore` beans, because the demos build `SimpleVectorStore`
  without an ObservationRegistry.
- **Memory after this call** step (03, 09, 19, 20): what each memory store holds after the call, with the entries this
  call wrote highlighted. Chat memory (any advisor holding a `ChatMemory`), spring-ai-session events (archived and
  summary events marked), and memory files (the folders named by `spring.ai.inspector.memory-dirs`).
- Runs of 3+ systemOne checks (e.g. per-document RAG filtering) fold into one group under "On the wire". systemOne
  round-trips are labelled with who served them, e.g. `typesafe · system-one` or `ollama · system-one`.
- Embedding calls (OpenAI-compatible `/embeddings`, Ollama `/api/embed`) show their model, inputs, vectors and prompt
  tokens; runs of 3+ (e.g. ingesting documents for RAG) fold into one group, in the cards and in the sequence view.

## Color schemes

Next to light / dark (◐), 🌱 switches to the **Spring** color scheme: spring.io's brand green and dark slate, with the
Spring AI logo in the header (in its light- or dark-background variant). Both schemes come in light and dark, and the
choice is remembered per browser.

## Sequence view and linked agents

Each run has a **Cards | Sequence** toggle. The sequence view draws the run as lanes (app, advisors, sub-agents, each
model, systemOne model, tools, vector store) with arrows in time order: requests solid, returns dashed, labelled with message
counts, `tool_use` names, stop reasons and latencies. **to scale**, on the diagram's header, spaces rows by elapsed
time, so the audience sees where the time goes. Clicking an arrow opens it in the Cards view. In-process sub-agents (16) get their own lane. The
lane heads stay in view while a long diagram scrolls (under the pinned tokens panel, when it is pinned).

Calls in another JVM are **linked by timing**. When a ChatClient call starts while another run has a tool call open
(e.g. 18's `Task` tool calling the A2A airbnb-agent), the inspector nests the remote call under that tool, in the Cards
view and as a dashed lane group in the Sequence view. A call made on another thread of the same run while one of
its tools is running (e.g. a background sub-agent started by `Task`) nests under the call that owns the tool and is
tagged *inferred*; without a running tool it stays top-level, so concurrent requests in a server app are not nested
under each other. Both rules assume one agent conversation at a time: unrelated demos running in parallel can be
linked wrongly. Export and replay work per run; a linked remote run is
exported and replayed separately, and a replayed caller run doesn't show the remote lanes.

## Recording and replaying runs

Every run has an **Export** button that saves it as JSON. **Import** (top bar) loads saved runs back. **▶ Replay** plays
a run again as a new run, without calling any model: step by step (<kbd>→</kbd> or <kbd>n</kbd>), or at 1×/2×/4× of
the original pace (<kbd>p</kbd> pauses, <kbd>Esc</kbd> stops). If the network fails during a talk, replay
the runs you recorded beforehand.

An exported run is the array of its events; [EVENTS.md](EVENTS.md) lists every event type with its fields and
who emits it. Each event carries the format version (`v`); a recording from a newer inspector is refused on
import rather than shown wrong, older ones are read as they are.

To have recordings available right after startup, put the exported files in a folder:

```bash
java -jar spring-ai-inspector-server/build/libs/spring-ai-inspector-server.jar --spring.ai.inspector.preload-dir=./recordings
```

## Modules

| Module | What it is |
|---|---|
| `spring-ai-inspector-server` | The inspector app: web UI, recording proxy, event store. Run it once. |
| `spring-ai-inspector-starter` | Auto-configured instrumentation for a Spring AI app. Add it as a dependency; it stays inactive unless an inspector is reachable. Depends only on `spring-ai-client-chat` and Boot auto-configuration (no web stack); `spring-ai-vector-store` is optional. |

In this repo, `common` depends on the starter, so every demo gets it transitively.

## Usage

```bash
./gradlew :spring-ai-inspector-server:bootRun
# or: java -jar spring-ai-inspector-server/build/libs/spring-ai-inspector-server.jar
open http://localhost:9001
```

Run these commands from the spring-ai-inspector directory (use .\gradlew.bat on Windows). Then run any demo as usual. Nothing in the demos needs to change.

To inspect any other Spring AI application, add the starter:

```xml
<dependency>
	<groupId>org.springaicommunity</groupId>
	<artifactId>spring-ai-inspector-starter</artifactId>
	<version>0.0.1-SNAPSHOT</version>
</dependency>
```

How the starter hooks in:

- At startup, the starter checks `<spring.ai.inspector.url>/api/ping`. If the inspector answers, it points the
  provider base URLs at the inspector's recording proxy (`/r/<runId>/<provider>`):
  - a built-in routing table: Anthropic and TypeSafe always (the proxy forwards to the base-url the app had before,
    a gateway, a mitmweb, a local Ollama serving Jev models, or to the provider's own endpoint); OpenAI (both the
    common and the Responses API base-url), Ollama, Mistral (all its model types), DeepSeek, ElevenLabs and Stability
    only when unset or pointing at their own endpoints, so Azure or GitHub Models setups are untouched.
  - the table changed or extended by properties, `spring.ai.inspector.routes.<provider>.{properties, default-hosts,
    mode, upstream, suffix, enabled}`: e.g. `routes.openai.mode=always` for an OpenAI-compatible endpoint whose base
    URL ends in `/v1` (Amazon Bedrock mantle), `routes.groq.properties=spring.ai.openai.base-url` for a provider of
    its own, `routes.anthropic.enabled=false` to leave one alone. The inspector recognizes the wire format by the
    request path (OpenAI-compatible, Anthropic, Ollama, embeddings, images, speech, transcription, moderation);
    see the [starter README](spring-ai-inspector-starter/README.md#6-route-other-model-providers).
- Models that make no HTTP calls (e.g. jinfer running in the JVM) or providers that aren't routed are still shown:
  chat round-trips come from the advisor right before the model, with the prompt, the response, tool calls, the
  model and token usage it reported; embedding, image, speech, transcription and moderation calls come from the
  model beans (all marked "no HTTP"), in the cards, the tokens panel and the sequence view.
- The run header lists the configured chat models of any provider (`spring.ai.<provider>.chat[.options].model`, and `spring.ai.<provider>.responses.model` for OpenAI's Responses API).
- It also adds two `InspectorAdvisor`s to every auto-configured `ChatClient.Builder`: one at the start of the
  advisor chain and one right before the model.
- Tool executions are reported from Spring AI's tool-calling observations. The demos don't include Boot's
  observation support, so the starter provides an `ObservationRegistry` when none exists (otherwise it attaches
  to the existing one). That also switches on Spring AI's own chat, vector store and advisor observations,
  which the starter uses to link calls, tool runs and HTTP round-trips to each other.
- If the inspector is not running, the application behaves exactly as before.

Settings, for the instrumented applications:

| Property | Default | |
|---|---|---|
| `spring.ai.inspector.enabled` | `true` | set to `false` to opt a demo out |
| `spring.ai.inspector.url` | `http://localhost:9001` | where the inspector runs |
| `spring.ai.inspector.memory-dirs` | | comma-separated folders shown as file-based memory (19 sets `${agent.memory.dir}`) |
| `spring.ai.inspector.routes.<provider>.*` | the built-in table | changes or extends the routing table (`properties`, `default-hosts`, `mode`, `upstream`, `suffix`, `enabled`); `route.openai` and `proxy.<name>` are the older spellings |

The starter only routes traffic when `<url>/api/ping` identifies itself as the inspector, so another service on the
same port is never used by mistake. Each run reports the original base URL of every provider it routes, and the
proxy forwards that run's calls there.

Settings, for the inspector server:

| Property | Default | |
|---|---|---|
| `server.port` | `9001` | |
| `server.address` | `127.0.0.1` | local only: the event stream contains prompts, tool results and memory contents |
| `spring.ai.inspector.upstreams.<provider>` | provider APIs | fallback upstream when a run didn't report its own (off when a token is set) |
| `spring.ai.inspector.preload-dir` | | folder of exported runs to load at startup |
| `spring.ai.inspector.token` | | shared secret; when set, `/api/**` needs it and the proxy serves only runs that registered with it |
| `spring.ai.inspector.allowed-hosts` | | extra `Host` names accepted next to `localhost` / `127.0.0.1` |
| `spring.ai.inspector.max-body-chars` | `512000` | longest recorded request/response body; longer ones are cut and marked |
| `spring.ai.inspector.max-total-bytes` | `268435456` | byte budget of the in-memory event log (oldest events dropped) |
| `spring.ai.inspector.max-request-bytes` | `16777216` | largest event post accepted |
| `spring.ai.inspector.max-import-bytes` | `268435456` | largest recording accepted by Import |
| `spring.ai.inspector.max-blob-bytes` | `67108864` | budget for the media kept for previews (images, audio); `0` keeps none |
| `spring.ai.inspector.max-blob-size` | `16777216` | largest single media item kept |

## Access and what gets recorded

The inspector sees everything the apps send: prompts, tool results, memory files, MCP messages, and the
`run-start` event tells the proxy where to forward each run's API keys. So:

- It listens on `127.0.0.1` and refuses requests whose `Host` header is not a loopback name (a page in your own
  browser whose DNS name points at 127.0.0.1 would otherwise read the stream). Listening on another address
  (`server.address`) accepts that address too; name more with `spring.ai.inspector.allowed-hosts`.
- A run's upstreams are fixed by its first `run-start` and must be `http(s)` URLs; a later event can't redirect
  a run. Imported recordings register nothing.
- To require a secret, set `spring.ai.inspector.token` on the server and in the apps. The UI gets it once from
  the address bar, `http://localhost:9001/#token=<value>`, and keeps it in the browser.
- Secret headers (`x-api-key`, `Authorization`, cookies, ...) and query parameters (`key`, `token`, ...) are
  recorded as `…redacted`; inline base64 (images, audio, documents) and binary bodies are replaced by a marker,
  their bytes kept apart for previews (see above) under a budget.
- **Exports and `preload-dir` files contain the recorded prompts, tool results and memory contents.** Treat them
  like logs.

## Deep links

The address bar tracks the selected run and view: `http://localhost:9001/#run=<runId>&view=sequence&scale=scaled`.
Bookmark one to open a (preloaded) recording directly in the right view during a talk.

## Building

The Inspector has an independent Gradle build containing only the starter and server.
The build uses a Java 25 toolchain. The Foojay resolver downloads a matching JDK automatically
if none is installed. Start Gradle with your existing JDK 25; no separate JDK 17 installation is needed
for the normal build. Gradle still needs an installed Java runtime to start.
The starter is compiled with `--release 17` and can be used on Java 17 or later; the server requires Java 25.
Node.js 22 or later is needed for the existing UI tests. CI uses JDK 25 and Node.js 24.
The wrapper pins Gradle 9.8.1 and verifies its distribution checksum; no Gradle installation is needed.
Both modules use the Spring Boot and dependency-management plugins. Boot dependency management is
imported automatically; the starter imports the Spring AI BOM through `dependencyManagement`.

From the `spring-ai-inspector` directory:

```bash
./gradlew build
./gradlew :spring-ai-inspector-starter:build
./gradlew :spring-ai-inspector-server:build
./gradlew :spring-ai-inspector-starter:publishToMavenLocal
```

On Windows use `.\gradlew.bat`. `build` runs Java tests and the server's Node UI tests.
The optional `./gradlew :spring-ai-inspector-starter:java17Test` task runs the starter tests on Java 17.
CI runs it in addition to the normal build; Foojay can provision Java 17 for this compatibility check.
The starter produces a normal library JAR and a sources JAR; the server produces
`spring-ai-inspector-server/build/libs/spring-ai-inspector-server.jar`.
The optional VectorStore integration does not add a vector-store dependency to consuming applications.

The existing Maven POMs are retained for the demo reactor. Both build definitions must keep their
Boot/AI versions and dependencies in sync. After changing the starter, either publish it to Maven local
with Gradle before rebuilding demos, or run `mvn clean install` for the Maven reactor. The existing demo
builds use `0.0.1-SNAPSHOT`; using a release also requires updating their managed starter version.

### GitHub Actions and publishing

The repository-root workflow `.github/workflows/inspector.yml` builds only Inspector when its files
or the workflow change. It runs Java/UI tests, builds the server image, and smoke-tests its API and UI.
Pull requests and branch pushes do not publish packages.

For publishing, configure the following in **urferr/voxxeddays2026-demo → Settings → Secrets and variables → Actions**:

- Secret `PACKAGES_TOKEN`: a personal access token (classic) with `write:packages` and access to
  `urferr/maven-artefacts`. The Maven registry is associated with that separate repository, so this
  workflow uses a dedicated token rather than assuming its own `GITHUB_TOKEN` can write there.
- Optional variable `MAVEN_PACKAGES_USERNAME`: the token owner's GitHub login; defaults to `urferr`.
- The server image uses the workflow's `GITHUB_TOKEN` with `packages: write`, without an additional secret.
  If the GHCR package already exists, grant the source repository Actions access to it.

Commit and push the branch, then merge the workflow into the default branch so it appears under
**Actions → Spring AI Inspector**. To publish, either run it manually with `publish=true` and the desired
`version`, or push an Inspector-specific tag pointing to a commit containing this workflow:

```bash
git tag inspector-v0.0.1
git push origin inspector-v0.0.1
```

Tag `inspector-v0.0.1` publishes starter version `0.0.1` and image tags `0.0.1`, `sha-<commit>`, and
`latest`. Prereleases and snapshots do not update `latest`; image tags are lowercase
(for example `0.0.2-snapshot`). Use a new version for each release. The Maven and container publish
jobs are independent after successful verification; one may succeed while the other fails.

For a local Maven publication, use your existing `gpr.user` and `gpr.key` properties in
`~/.gradle/gradle.properties`. These take precedence over the environment-variable fallbacks
`PACKAGES_USER` and `PACKAGES_TOKEN` (used by CI). Then run:

```bash
./gradlew :spring-ai-inspector-starter:publishMavenJavaPublicationToGitHubPackagesRepository -PinspectorVersion=0.0.1
```

Versions and the Maven target are defined in `gradle.properties`; override the target with
`-PgithubPackagesRepository=owner/repository` if needed.

### Using the published starter

GitHub's Maven registry requires authentication even for public packages. Consumers need a token
(classic) with `read:packages` and access to `urferr/maven-artefacts`.
Keep credentials in environment variables or your user configuration, not in committed build files.

For Gradle (Groovy DSL):

```groovy
repositories {
	mavenCentral()
	maven {
		url = uri('https://maven.pkg.github.com/urferr/maven-artefacts')
		credentials {
			username = providers.gradleProperty('gpr.user')
				.orElse(providers.environmentVariable('PACKAGES_USER')).orNull
			password = providers.gradleProperty('gpr.key')
				.orElse(providers.environmentVariable('PACKAGES_TOKEN')).orNull
		}
		content {
			includeModule('org.springaicommunity', 'spring-ai-inspector-starter')
		}
	}
}

dependencies {
	implementation 'org.springaicommunity:spring-ai-inspector-starter:0.0.1'
}
```

For Maven, add the dependency shown above with version `0.0.1` and this repository to your POM:

```xml
<repositories>
	<repository>
		<id>github-inspector</id>
		<url>https://maven.pkg.github.com/urferr/maven-artefacts</url>
	</repository>
</repositories>
```

Add matching credentials to your user `~/.m2/settings.xml`:

```xml
<settings>
	<servers>
		<server>
			<id>github-inspector</id>
			<username>${env.PACKAGES_USER}</username>
			<password>${env.PACKAGES_TOKEN}</password>
		</server>
	</servers>
</settings>
```

These release examples become usable after the first successful publication.

### Server image

CI publishes `linux/amd64` images to `ghcr.io/urferr/spring-ai-inspector-server`.
For a private image, first authenticate with a token (classic) with `read:packages`:

```bash
printf '%s' "$GHCR_TOKEN" | docker login ghcr.io -u urferr --password-stdin
docker pull ghcr.io/urferr/spring-ai-inspector-server:0.0.1
docker run --rm -p 127.0.0.1:9001:9001 ghcr.io/urferr/spring-ai-inspector-server:0.0.1
```

To allow unauthenticated pulls, change the container package visibility to public in GitHub's package settings.

For a local image, build the JAR first:

```bash
./gradlew :spring-ai-inspector-server:build
docker build -t spring-ai-inspector-server:local spring-ai-inspector-server
docker run --rm -p 127.0.0.1:9001:9001 spring-ai-inspector-server:local
```

The container runs as a non-root user on JRE 25 and sets `SERVER_ADDRESS=0.0.0.0` so Docker can forward
connections. The host port above stays bound to loopback because the Inspector exposes prompts, tool
results and memory. The regular JAR keeps its existing local-only default. Open `http://localhost:9001`
and start instrumented applications with `spring.ai.inspector.url=http://localhost:9001`.

A provider upstream reported as `localhost` points inside the container, not at the Docker host.
In particular, the current starter routes Ollama only for its default localhost base URL; changing
that URL to `host.docker.internal` bypasses its wire capture. For local Ollama inspection, use the JAR
directly or arrange container networking so the reported upstream is reachable. The server fallback
`SPRING_AI_INSPECTOR_UPSTREAMS_OLLAMA` applies only when the run did not report an upstream; it cannot
override a run's reported localhost URL.

### UI code

The UI is plain ES modules, served as static files; there is no build step.

```
spring-ai-inspector-server/src/main/resources/static/
├── index.html · css/inspector.css
└── js/
    ├── main.js          entry point: DOM listeners, live event stream, deep links (the only module touching the DOM on load)
    ├── state.js         UI state and preferences
    ├── model.js         turns events into runs, calls, round-trips, tool runs, links
    ├── providers.js     wire-format adapters (Anthropic, OpenAI Chat Completions and Responses, Ollama, Mistral, DeepSeek, TypeSafe,
    │                    embeddings, images, speech, transcription, moderation)
    ├── util.js          escaping, formatting, JSON highlighting
    ├── io.js · replay.js  export/import, replay
    └── render/          cards, wire, messages, rag, memory, sequence, page
```

The events the modules consume are documented in [EVENTS.md](EVENTS.md). Every module except `main.js` can be imported without a browser. Recorded demo runs in `src/test/js/fixtures`
drive the real model and render code in Node tests:

```bash
node --test spring-ai-inspector-server/src/test/js/*.test.mjs
```

## Limitations

- **Keep the inspector running while instrumented apps run.** The starter points the provider base URLs at the
  inspector's proxy once, at startup. If the inspector stops while an app is still running, that app's model calls
  fail (connection refused) until the inspector is back on the same port. Restarting the inspector is fine (the
  starter announces its run again, so a token-protected inspector knows the run's upstreams); stopping
  it for good means restarting the apps too (they then talk to the providers directly again).

- Wire capture covers Anthropic, OpenAI, Mistral and DeepSeek (all Chat Completions style except Anthropic), OpenAI's
  Responses API (`/v1/responses`, also from OpenAI-compatible providers) and Ollama (`/api/chat`, `/api/generate`). Google GenAI and Bedrock are not proxied; those demos still show the advisor layers.
- Endpoints without an adapter (anything but chat, embeddings, images, speech, transcription and moderation)
  are captured but shown as raw JSON only.
- A `ChatClient` built with `ChatClient.builder(chatModel)` instead of the injected builder gets no advisor
  events, but its wire traffic is still captured. It also has no observations, so tool runs and searches made
  inside it are attributed by timing rather than by the call that made them.
- Events are kept in memory and are lost when the inspector restarts. **Clear** resets the view between talk sections;
  use **Export** / `spring.ai.inspector.preload-dir` to keep runs.
