# Spring AI Inspector

A live web UI that shows what each demo sends to the model, in three layers:

1. **Your app sent**: the prompt exactly as the `ChatClient` call wrote it.
2. **After the advisors**: the prompt after memory, RAG, guardrails and other advisors ran. Messages the advisors added or removed are highlighted.
3. **On the wire**: every raw HTTP round-trip to the model, with request and response JSON, token usage and timing. Anthropic, OpenAI, Ollama, Mistral and DeepSeek traffic is rendered as one readable conversation. TypeSafe Jev
   `systemOne` calls (guardrails, judges, RAG filters) get a questions-and-answers view with the answer probabilities. Messages re-sent from an earlier round-trip are marked `re-sent`, new ones `new`. API keys are redacted.

Nested `ChatClient` calls (for example sub-agents) are shown inside the call that triggered them. Tool executions
(including MCP tools) appear between the round-trips that requested and consumed them, with arguments, result,
errors and duration.

## RAG and memory

- **Retrieval (RAG)** step (05, 05-1): the configured pipeline stages, read from the `QuestionAnswerAdvisor` /
  `RetrievalAugmentationAdvisor`; every vector search with its query and scored hits; the funnel
  *queries → hits → unique → in the prompt*; the documents that reached the prompt (with Jev rerank scores and
  classifications) and the ones dropped by joining or post-processing. Ingestion shows as one line at the start of the
  run. Vector stores are observed by wrapping `VectorStore` beans, because the demos build `SimpleVectorStore`
  without an ObservationRegistry.
- **Memory after this call** step (03, 09, 19, 20): what each memory store holds after the call, with the entries this
  call wrote highlighted. Chat memory (any advisor holding a `ChatMemory`), spring-ai-session events (archived and
  summary events marked), and memory files (`spring.ai.inspector.memory-dirs`, defaults to `agent.memory.dir`).
- Runs of 3+ systemOne checks (e.g. per-document RAG filtering) fold into one group under "On the wire". systemOne
  round-trips are labelled with who served them, e.g. `typesafe · system-one` or `ollama · system-one`.

## Sequence view and linked agents

Each run has a **Cards | Sequence** toggle. The sequence view draws the run as lanes (app, advisors, sub-agents, each
model, systemOne model, tools, vector store) with arrows in time order: requests solid, returns dashed, labelled with message
counts, `tool_use` names, stop reasons and latencies. **to scale** spaces rows by elapsed time, so the audience sees
where the time goes. Clicking an arrow opens it in the Cards view. In-process sub-agents (16) get their own lane.

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
  - `spring.ai.anthropic.base-url`: always. The proxy forwards to the base-url the app had before (a gateway, a
    mitmweb, ...), or to `https://api.anthropic.com` when none was set.
  - `spring.ai.openai.base-url`: only if unset or pointing at `api.openai.com`, so Azure or GitHub Models setups are untouched.
    `spring.ai.inspector.route.openai=always` routes it anyway, for an OpenAI-compatible endpoint whose base URL ends in `/v1`
    (e.g. Amazon Bedrock mantle).
  - `spring.ai.ollama.base-url`: only if unset or pointing at `localhost:11434`.
  - `spring.ai.mistralai.base-url` and `spring.ai.mistralai.chat.base-url`: only if unset or pointing at `api.mistral.ai`.
    Both are set because the chat properties preset their own base-url, which wins over the common one.
  - `spring.ai.deepseek.base-url`: only if unset or pointing at `api.deepseek.com`.
  - `spring.ai.typesafe.base-url`: always, like Anthropic. The proxy forwards to the base-url the app had before (e.g. a
    local Ollama serving Jev models), or to `https://api.typesafe.ai` when none was set.
- It also adds two `InspectorAdvisor`s to every auto-configured `ChatClient.Builder`: one at the start of the
  advisor chain and one right before the model.
- Tool executions are reported from Spring AI's tool-calling observations. The demos don't include Boot's
  observation support, so `common` provides an `ObservationRegistry` when none exists (otherwise it attaches to
  the existing one).
- If the inspector is not running, the application behaves exactly as before.

Settings, for the instrumented applications:

| Property | Default | |
|---|---|---|
| `spring.ai.inspector.enabled` | `true` | set to `false` to opt a demo out |
| `spring.ai.inspector.url` | `http://localhost:9001` | where the inspector runs |
| `spring.ai.inspector.memory-dirs` | `${agent.memory.dir}` | comma-separated folders shown as file-based memory |
| `spring.ai.inspector.route.openai` | | `always` routes a non-default OpenAI base URL (an OpenAI-compatible endpoint ending in `/v1`) |

The starter only routes traffic when `<url>/api/ping` identifies itself as the inspector, so another service on the
same port is never used by mistake. Each run reports the original base URL of every provider it routes, and the
proxy forwards that run's calls there.

Settings, for the inspector server:

| Property | Default | |
|---|---|---|
| `server.port` | `9001` | |
| `server.address` | `127.0.0.1` | local only: the event stream contains prompts, tool results and memory contents |
| `spring.ai.inspector.upstreams.<provider>` | provider APIs | fallback upstream when a run didn't report its own |
| `spring.ai.inspector.preload-dir` | | folder of exported runs to load at startup |

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
    ├── providers.js     wire-format adapters (Anthropic, OpenAI, Ollama, Mistral, DeepSeek, TypeSafe)
    ├── util.js          escaping, formatting, JSON highlighting
    ├── io.js · replay.js  export/import, replay
    └── render/          cards, wire, messages, rag, memory, sequence, page
```

Every module except `main.js` can be imported without a browser. Recorded demo runs in `src/test/js/fixtures`
drive the real model and render code in Node tests:

```bash
node --test spring-ai-inspector-server/src/test/js/*.test.mjs
```

## Limitations

- **Keep the inspector running while instrumented apps run.** The starter points the provider base URLs at the
  inspector's proxy once, at startup. If the inspector stops while an app is still running, that app's model calls
  fail (connection refused) until the inspector is back on the same port. Restarting the inspector is fine; stopping
  it for good means restarting the apps too (they then talk to the providers directly again).

- Wire capture covers Anthropic, OpenAI, Mistral and DeepSeek (all Chat Completions style except Anthropic) and Ollama
  (`/api/chat`, `/api/generate`). Google GenAI and Bedrock are not proxied; those demos still show the advisor layers.
- Other endpoints that go through the proxy, such as embeddings, are captured but shown as raw JSON only.
- A `ChatClient` built with `ChatClient.builder(chatModel)` instead of the injected builder gets no advisor
  events, but its wire traffic is still captured.
- Events are kept in memory and are lost when the inspector restarts. **Clear** resets the view between talk sections;
  use **Export** / `spring.ai.inspector.preload-dir` to keep runs.
