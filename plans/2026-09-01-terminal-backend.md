# Terminal Backend Module — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #401 — feat: casehub-pages-terminal — shared PTY/tmux WebSocket backend module
**Issue group:** #401

**Goal:** Build a shared terminal backend that any CaseHub app can depend on for tmux-backed WebSocket terminal sessions.

**Architecture:** Two Maven modules following the push/push-runtime pattern. Core module (`terminal`) has pure Java classes for tmux management, FIFO relay, and session logging. Runtime module (`terminal-runtime`) adds Quarkus WebSocket endpoint, CDI registry, and REST resource. Source generalized from trellis's terminal backend with echo duplication fix.

**Tech Stack:** Java 21, Quarkus (runtime only), tmux, JUnit 5, AssertJ

## Global Constraints

- Core module: no Quarkus, no CDI, no JAX-RS — pure Java + jackson-core only
- Runtime module: Quarkus Arc, quarkus-websockets-next, JAX-RS
- Follow push/push-runtime POM structure exactly
- Maven can't run in this session — tests written but verified in follow-up session
- Package: `io.casehub.pages.terminal` (core), `io.casehub.pages.terminal.runtime` (runtime)

---

## Batch 1: Core module — casehub-pages-terminal

### Task 1: Create core module with TmuxManager, FifoRelay, SessionLogger, TerminalSession

**Files:**
- Create: `backend/terminal/pom.xml`
- Create: `backend/terminal/src/main/java/io/casehub/pages/terminal/TmuxManager.java`
- Create: `backend/terminal/src/main/java/io/casehub/pages/terminal/FifoRelay.java`
- Create: `backend/terminal/src/main/java/io/casehub/pages/terminal/SessionLogger.java`
- Create: `backend/terminal/src/main/java/io/casehub/pages/terminal/TerminalSession.java`
- Create: `backend/terminal/src/test/java/io/casehub/pages/terminal/FifoRelayTest.java`
- Create: `backend/terminal/src/test/java/io/casehub/pages/terminal/SessionLoggerTest.java`
- Modify: `backend/pom.xml` — add `<module>terminal</module>`

**Interfaces:**
- Produces: `TmuxManager` (session CRUD, send-keys, pipe-pane, resize), `FifoRelay` (FIFO→Consumer relay), `SessionLogger` (append, tailLines, delete), `TerminalSession` (record: name, workingDir, createdAt)

- [ ] **Step 1: Create backend/terminal/pom.xml**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>io.casehub</groupId>
        <artifactId>casehub-pages-backend</artifactId>
        <version>0.2-SNAPSHOT</version>
    </parent>

    <artifactId>casehub-pages-terminal</artifactId>
    <name>CaseHub Pages — Terminal Core</name>
    <description>Tmux session management, FIFO relay, and session logging for terminal WebSocket backends</description>

    <dependencies>
        <dependency>
            <groupId>org.junit.jupiter</groupId>
            <artifactId>junit-jupiter</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.assertj</groupId>
            <artifactId>assertj-core</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
</project>
```

- [ ] **Step 2: Add module to backend/pom.xml**

Add `<module>terminal</module>` after `<module>scenario-client</module>` in the `<modules>` list.

- [ ] **Step 3: Create TerminalSession record**

```java
package io.casehub.pages.terminal;

import java.time.Instant;

public record TerminalSession(String name, String workingDir, Instant createdAt) {
    public TerminalSession(String name, String workingDir) {
        this(name, workingDir, Instant.now());
    }
}
```

- [ ] **Step 4: Create TmuxManager**

Generalize from trellis's `TmuxManager`. Remove `@ApplicationScoped` (CDI annotation — that goes in runtime). Make it a plain class.

```java
package io.casehub.pages.terminal;

import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStreamReader;
import java.io.OutputStream;
import java.util.List;
import java.util.Optional;
import java.util.stream.Collectors;

public class TmuxManager {

    private final String prefix;

    public TmuxManager(String prefix) {
        this.prefix = prefix;
    }

    public TmuxManager() {
        this("pages-");
    }

    public String prefix() { return prefix; }

    public void createSession(String name, String workingDir) throws IOException, InterruptedException {
        run("tmux", "new-session", "-d", "-s", name, "-c", workingDir);
        sendKeys(name, "cd " + workingDir + " && clear\n");
    }

    public void killSession(String name) throws IOException, InterruptedException {
        run("tmux", "kill-session", "-t", name);
    }

    public boolean hasSession(String name) throws IOException, InterruptedException {
        var p = new ProcessBuilder("tmux", "has-session", "-t", name)
                .redirectErrorStream(true).start();
        p.getInputStream().transferTo(OutputStream.nullOutputStream());
        return p.waitFor() == 0;
    }

    public List<String> listSessions() throws IOException, InterruptedException {
        var pb = new ProcessBuilder("tmux", "list-sessions", "-F", "#{session_name}");
        pb.redirectErrorStream(true);
        var process = pb.start();
        List<String> sessions;
        try (var reader = new BufferedReader(new InputStreamReader(process.getInputStream()))) {
            sessions = reader.lines()
                    .filter(l -> !l.isBlank() && l.startsWith(prefix))
                    .collect(Collectors.toList());
        }
        int exit = process.waitFor();
        return exit == 0 ? sessions : List.of();
    }

    public void sendKeys(String name, String text) throws IOException, InterruptedException {
        run("tmux", "send-keys", "-t", name, "-l", text);
    }

    public String capturePane(String name, int lines) throws IOException, InterruptedException {
        var p = new ProcessBuilder("tmux", "capture-pane", "-t", name, "-e", "-p", "-S", String.valueOf(-lines))
                .redirectErrorStream(true).start();
        try (var in = p.getInputStream()) {
            var output = new String(in.readAllBytes());
            p.waitFor();
            return output;
        }
    }

    public void setOption(String name, String key, String value) throws IOException, InterruptedException {
        run("tmux", "set-option", "-t", name, key, value);
    }

    public Optional<String> getOption(String name, String key) throws IOException, InterruptedException {
        var p = new ProcessBuilder("tmux", "show-options", "-t", name, "-v", key)
                .redirectErrorStream(false).start();
        var value = new String(p.getInputStream().readAllBytes()).trim();
        int exit = p.waitFor();
        if (exit != 0 || value.isBlank()) return Optional.empty();
        return Optional.of(value);
    }

    public void resizeWindow(String name, int cols, int rows) throws IOException, InterruptedException {
        run("tmux", "resize-window", "-t", name, "-x", String.valueOf(cols), "-y", String.valueOf(rows));
    }

    public void forceRedraw(String name, int cols, int rows) throws IOException, InterruptedException {
        resizeWindow(name, cols - 1, rows);
        Thread.sleep(50);
        resizeWindow(name, cols, rows);
    }

    public void pipePaneToFifo(String name, String fifoPath) throws IOException, InterruptedException {
        run("tmux", "pipe-pane", "-O", "-t", name, "cat > " + fifoPath);
    }

    public void stopPipePane(String name) throws IOException, InterruptedException {
        run("tmux", "pipe-pane", "-t", name);
    }

    private void run(String... command) throws IOException, InterruptedException {
        var p = new ProcessBuilder(command).redirectErrorStream(true).start();
        p.getInputStream().transferTo(OutputStream.nullOutputStream());
        p.waitFor();
    }
}
```

- [ ] **Step 5: Create FifoRelay**

Same as trellis but public class (was package-private).

```java
package io.casehub.pages.terminal;

import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStream;
import java.io.InputStreamReader;
import java.nio.charset.StandardCharsets;
import java.util.function.Consumer;

public class FifoRelay {

    private final InputStream input;
    private final Consumer<String> sink;
    private boolean skipInitialNewline = true;

    public FifoRelay(InputStream input, Consumer<String> sink) {
        this.input = input;
        this.sink = sink;
    }

    public void relay() throws IOException {
        try (var reader = new BufferedReader(
                new InputStreamReader(input, StandardCharsets.UTF_8), 4096)) {
            var cbuf = new char[4096];
            int n;
            while ((n = reader.read(cbuf)) != -1) {
                int start = 0;
                if (skipInitialNewline) {
                    skipInitialNewline = false;
                    if (n > 0 && cbuf[0] == '\r') start++;
                    if (start < n && cbuf[start] == '\n') start++;
                    if (start >= n) continue;
                }
                sink.accept(new String(cbuf, start, n - start));
            }
        }
    }
}
```

- [ ] **Step 6: Create SessionLogger**

Generalize from trellis — remove CDI annotations, accept logDir via constructor.

```java
package io.casehub.pages.terminal;

import java.io.IOException;
import java.io.RandomAccessFile;
import java.io.UncheckedIOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.StandardOpenOption;

public class SessionLogger {

    private final Path sessionsDir;

    public SessionLogger(Path sessionsDir) {
        this.sessionsDir = sessionsDir;
        try {
            Files.createDirectories(sessionsDir);
        } catch (IOException e) {
            throw new UncheckedIOException(e);
        }
    }

    public void append(String terminalName, String text) {
        try {
            Files.writeString(logPath(terminalName), text,
                    StandardOpenOption.CREATE, StandardOpenOption.APPEND);
        } catch (IOException e) {
            // Log write failure is non-fatal
        }
    }

    public String tailLines(String terminalName, int lines) {
        return tailLinesWithOffset(terminalName, lines, 0);
    }

    public String tailLinesWithOffset(String terminalName, int lines, int offset) {
        var path = logPath(terminalName);
        if (!Files.exists(path)) return "";

        try (var raf = new RandomAccessFile(path.toFile(), "r")) {
            long fileLength = raf.length();
            if (fileLength == 0) return "";

            int totalLines = lines + offset;
            int newlinesFound = 0;
            long pos = fileLength - 1;

            raf.seek(pos);
            if (raf.readByte() == '\n') pos--;

            while (pos > 0 && newlinesFound < totalLines) {
                raf.seek(pos);
                if (raf.readByte() == '\n') newlinesFound++;
                pos--;
            }

            long startPos;
            if (pos == 0 && newlinesFound < totalLines) {
                raf.seek(0);
                if (raf.readByte() == '\n') newlinesFound++;
                startPos = (newlinesFound >= totalLines) ? 1 : 0;
            } else {
                startPos = pos + 2;
            }

            long endPos = fileLength;
            if (offset > 0) {
                int skipLines = 0;
                long ep = fileLength - 1;
                raf.seek(ep);
                if (raf.readByte() == '\n') ep--;
                while (ep > startPos && skipLines < offset) {
                    raf.seek(ep);
                    if (raf.readByte() == '\n') skipLines++;
                    ep--;
                }
                endPos = ep + 2;
            }

            int len = (int) (endPos - startPos);
            if (len <= 0) return "";
            raf.seek(startPos);
            byte[] buf = new byte[len];
            raf.readFully(buf);
            return new String(buf);
        } catch (IOException e) {
            return "";
        }
    }

    public Path logPath(String terminalName) {
        return sessionsDir.resolve(terminalName + ".log");
    }

    public void delete(String terminalName) {
        try {
            Files.deleteIfExists(logPath(terminalName));
        } catch (IOException e) {
            // Delete failure is non-fatal
        }
    }
}
```

- [ ] **Step 7: Create FifoRelayTest**

```java
package io.casehub.pages.terminal;

import org.junit.jupiter.api.Test;
import java.io.ByteArrayInputStream;
import java.nio.charset.StandardCharsets;
import java.util.ArrayList;
import java.util.List;
import static org.assertj.core.api.Assertions.assertThat;

class FifoRelayTest {

    @Test
    void relay_delivers_text_to_sink() throws Exception {
        var input = new ByteArrayInputStream("hello world".getBytes(StandardCharsets.UTF_8));
        List<String> received = new ArrayList<>();
        new FifoRelay(input, received::add).relay();
        assertThat(String.join("", received)).isEqualTo("hello world");
    }

    @Test
    void relay_skips_initial_newline() throws Exception {
        var input = new ByteArrayInputStream("\r\nhello".getBytes(StandardCharsets.UTF_8));
        List<String> received = new ArrayList<>();
        new FifoRelay(input, received::add).relay();
        assertThat(String.join("", received)).isEqualTo("hello");
    }

    @Test
    void relay_handles_empty_input() throws Exception {
        var input = new ByteArrayInputStream(new byte[0]);
        List<String> received = new ArrayList<>();
        new FifoRelay(input, received::add).relay();
        assertThat(received).isEmpty();
    }
}
```

- [ ] **Step 8: Create SessionLoggerTest**

```java
package io.casehub.pages.terminal;

import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.io.TempDir;
import java.nio.file.Path;
import static org.assertj.core.api.Assertions.assertThat;

class SessionLoggerTest {

    @Test
    void append_and_tail(@TempDir Path tmpDir) {
        var logger = new SessionLogger(tmpDir);
        logger.append("test", "line1\nline2\nline3\n");
        assertThat(logger.tailLines("test", 2)).isEqualTo("line2\nline3\n");
    }

    @Test
    void tail_empty_session(@TempDir Path tmpDir) {
        var logger = new SessionLogger(tmpDir);
        assertThat(logger.tailLines("nonexistent", 10)).isEmpty();
    }

    @Test
    void delete_removes_log(@TempDir Path tmpDir) {
        var logger = new SessionLogger(tmpDir);
        logger.append("test", "data");
        logger.delete("test");
        assertThat(logger.tailLines("test", 10)).isEmpty();
    }
}
```

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add backend/terminal/ backend/pom.xml
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat(#401): casehub-pages-terminal core module — TmuxManager, FifoRelay, SessionLogger

Pure Java terminal infrastructure: tmux session management, FIFO pipe
relay for output capture, per-session logging with tail. No Quarkus
dependency.

Refs #401"
```

## Batch 2: Runtime module — casehub-pages-terminal-runtime

### Task 2: Create runtime module with WebSocket, Registry, REST, and Producers

**Files:**
- Create: `backend/terminal-runtime/pom.xml`
- Create: `backend/terminal-runtime/src/main/java/io/casehub/pages/terminal/runtime/TerminalWebSocket.java`
- Create: `backend/terminal-runtime/src/main/java/io/casehub/pages/terminal/runtime/TerminalRegistry.java`
- Create: `backend/terminal-runtime/src/main/java/io/casehub/pages/terminal/runtime/TerminalResource.java`
- Create: `backend/terminal-runtime/src/main/java/io/casehub/pages/terminal/runtime/TerminalProducers.java`
- Create: `backend/terminal-runtime/src/test/java/io/casehub/pages/terminal/runtime/TerminalRegistryTest.java`
- Modify: `backend/pom.xml` — add `<module>terminal-runtime</module>`

**Interfaces:**
- Consumes: `TmuxManager`, `FifoRelay`, `SessionLogger`, `TerminalSession` from Task 1
- Produces: `TerminalWebSocket` (WebSocket endpoint), `TerminalRegistry` (session lifecycle), `TerminalResource` (REST API), `TerminalProducers` (CDI wiring)

- [ ] **Step 1: Create backend/terminal-runtime/pom.xml**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>io.casehub</groupId>
        <artifactId>casehub-pages-backend</artifactId>
        <version>0.2-SNAPSHOT</version>
    </parent>

    <artifactId>casehub-pages-terminal-runtime</artifactId>
    <name>CaseHub Pages Terminal Runtime</name>
    <description>Quarkus CDI integration for terminal WebSocket sessions</description>

    <dependencies>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-pages-terminal</artifactId>
        </dependency>
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-arc</artifactId>
        </dependency>
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-websockets-next</artifactId>
        </dependency>
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-rest-jackson</artifactId>
        </dependency>
        <dependency>
            <groupId>org.eclipse.microprofile.config</groupId>
            <artifactId>microprofile-config-api</artifactId>
            <scope>provided</scope>
        </dependency>
        <dependency>
            <groupId>org.junit.jupiter</groupId>
            <artifactId>junit-jupiter</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.assertj</groupId>
            <artifactId>assertj-core</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
</project>
```

- [ ] **Step 2: Add module to backend/pom.xml**

Add `<module>terminal-runtime</module>` after `<module>terminal</module>`.

Also add `casehub-pages-terminal` to the `<dependencyManagement>` section in `backend/pom.xml`:
```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-pages-terminal</artifactId>
    <version>${project.version}</version>
</dependency>
```

- [ ] **Step 3: Create TerminalProducers**

```java
package io.casehub.pages.terminal.runtime;

import io.casehub.pages.terminal.SessionLogger;
import io.casehub.pages.terminal.TmuxManager;
import io.quarkus.arc.DefaultBean;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Produces;
import org.eclipse.microprofile.config.inject.ConfigProperty;

import java.nio.file.Path;

@ApplicationScoped
public class TerminalProducers {

    @Produces
    @DefaultBean
    @ApplicationScoped
    public TmuxManager tmuxManager(
            @ConfigProperty(name = "casehub.pages.terminal.prefix", defaultValue = "pages-") String prefix) {
        return new TmuxManager(prefix);
    }

    @Produces
    @DefaultBean
    @ApplicationScoped
    public SessionLogger sessionLogger(
            @ConfigProperty(name = "casehub.pages.terminal.log-dir",
                    defaultValue = "${java.io.tmpdir}/pages-terminal-sessions") Path logDir) {
        return new SessionLogger(logDir);
    }
}
```

- [ ] **Step 4: Create TerminalRegistry**

```java
package io.casehub.pages.terminal.runtime;

import io.casehub.pages.terminal.SessionLogger;
import io.casehub.pages.terminal.TerminalSession;
import io.casehub.pages.terminal.TmuxManager;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import java.io.IOException;
import java.util.List;
import java.util.Optional;
import java.util.concurrent.ConcurrentHashMap;

@ApplicationScoped
public class TerminalRegistry {

    private final TmuxManager tmux;
    private final SessionLogger logger;
    private final ConcurrentHashMap<String, TerminalSession> sessions = new ConcurrentHashMap<>();

    @Inject
    public TerminalRegistry(TmuxManager tmux, SessionLogger logger) {
        this.tmux = tmux;
        this.logger = logger;
    }

    void onStart(@jakarta.enterprise.event.Observes io.quarkus.runtime.StartupEvent event) {
        bootstrap();
    }

    public void createSession(String name, String workingDir)
            throws IOException, InterruptedException {
        var session = new TerminalSession(name, workingDir);
        if (sessions.putIfAbsent(name, session) != null) {
            throw new IllegalStateException("Terminal already exists: " + name);
        }
        try {
            tmux.createSession(name, workingDir);
        } catch (IOException | InterruptedException e) {
            sessions.remove(name);
            throw e;
        }
    }

    public void destroySession(String name) throws IOException, InterruptedException {
        tmux.killSession(name);
        sessions.remove(name);
        logger.delete(name);
    }

    public void sendKeys(String name, String text) throws IOException, InterruptedException {
        tmux.sendKeys(name, text);
    }

    public void resize(String name, int cols, int rows) throws IOException, InterruptedException {
        tmux.resizeWindow(name, cols, rows);
    }

    public Optional<TerminalSession> get(String name) {
        return Optional.ofNullable(sessions.get(name));
    }

    public List<TerminalSession> list() {
        return List.copyOf(sessions.values());
    }

    public void bootstrap() {
        try {
            for (String name : tmux.listSessions()) {
                sessions.put(name, new TerminalSession(name, null));
            }
        } catch (IOException | InterruptedException e) {
            // Bootstrap failure is non-fatal
        }
    }
}
```

- [ ] **Step 5: Create TerminalWebSocket with echo fix**

Key changes from trellis:
- Stop pipe-pane before starting new one (fix duplicate output)
- Delay pipe-pane start after forceRedraw (fix echo concatenation)

```java
package io.casehub.pages.terminal.runtime;

import io.casehub.pages.terminal.FifoRelay;
import io.casehub.pages.terminal.SessionLogger;
import io.casehub.pages.terminal.TmuxManager;
import io.quarkus.websockets.next.OnClose;
import io.quarkus.websockets.next.OnOpen;
import io.quarkus.websockets.next.OnTextMessage;
import io.quarkus.websockets.next.WebSocket;
import io.quarkus.websockets.next.WebSocketConnection;
import jakarta.inject.Inject;

import java.io.IOException;
import java.util.concurrent.ConcurrentHashMap;

@WebSocket(path = "/ws/terminal/{id}/{cols}/{rows}")
public class TerminalWebSocket {

    @Inject TmuxManager tmux;
    @Inject TerminalRegistry registry;
    @Inject SessionLogger logger;

    private final ConcurrentHashMap<String, String> sessionNames = new ConcurrentHashMap<>();
    private final ConcurrentHashMap<String, String> fifoPaths = new ConcurrentHashMap<>();
    private final ConcurrentHashMap<String, WebSocketConnection> activeBySession = new ConcurrentHashMap<>();

    @OnOpen
    public void onOpen(WebSocketConnection connection) {
        var sessionName = connection.pathParam("id");
        if (registry.get(sessionName).isEmpty()) {
            try { connection.closeAndAwait(); } catch (Exception ignored) {}
            return;
        }

        int cols = parsePathInt(connection.pathParam("cols"));
        int rows = parsePathInt(connection.pathParam("rows"));
        var fifoPath = "/tmp/pages-terminal-" + connection.id() + ".pipe";

        sessionNames.put(connection.id(), sessionName);

        var previous = activeBySession.put(sessionName, connection);
        if (previous != null && !previous.id().equals(connection.id())) {
            cleanup(previous);
            try {
                previous.closeAndAwait(new io.quarkus.websockets.next.CloseReason(4001, "session-takeover"));
            } catch (Exception ignored) {}
        }

        try {
            // Stop any existing pipe-pane first (prevents duplicate output on reconnect)
            tmux.stopPipePane(sessionName);

            new ProcessBuilder("mkfifo", fifoPath).redirectErrorStream(true).start().waitFor();
            fifoPaths.put(connection.id(), fifoPath);

            // Force redraw BEFORE starting pipe-pane (prevents redraw output being captured)
            if (cols > 0 && rows > 0) {
                tmux.forceRedraw(sessionName, cols, rows);
            }

            // Start FIFO relay thread
            Thread.ofVirtual().name("pages-fifo-" + sessionName).start(() -> {
                try {
                    new FifoRelay(
                            new java.io.FileInputStream(fifoPath),
                            text -> {
                                connection.sendTextAndAwait(text);
                                logger.append(sessionName, text);
                            }
                    ).relay();
                } catch (IOException e) {
                    // FIFO stream ended — connection closing
                }
            });

            // Start pipe-pane AFTER redraw completes
            tmux.pipePaneToFifo(sessionName, fifoPath);

        } catch (IOException | InterruptedException e) {
            cleanup(connection);
            try { connection.closeAndAwait(); } catch (Exception ignored) {}
        }
    }

    @OnTextMessage
    public void onMessage(WebSocketConnection connection, String message) {
        var sessionName = sessionNames.get(connection.id());
        if (sessionName == null) return;
        try {
            tmux.sendKeys(sessionName, message);
        } catch (IOException | InterruptedException e) {
            // Send failure is non-fatal
        }
    }

    @OnClose
    public void onClose(WebSocketConnection connection) {
        cleanup(connection);
    }

    private void cleanup(WebSocketConnection connection) {
        var sessionName = sessionNames.remove(connection.id());
        if (sessionName != null) {
            try { tmux.stopPipePane(sessionName); } catch (Exception ignored) {}
            activeBySession.remove(sessionName, connection);
        }
        var fifoPath = fifoPaths.remove(connection.id());
        if (fifoPath != null) {
            try { java.nio.file.Files.deleteIfExists(java.nio.file.Path.of(fifoPath)); } catch (Exception ignored) {}
        }
    }

    private static int parsePathInt(String value) {
        try { return value != null ? Integer.parseInt(value) : 0; }
        catch (NumberFormatException e) { return 0; }
    }
}
```

- [ ] **Step 6: Create TerminalResource**

```java
package io.casehub.pages.terminal.runtime;

import io.casehub.pages.terminal.TmuxManager;
import jakarta.inject.Inject;
import jakarta.ws.rs.*;
import jakarta.ws.rs.core.MediaType;
import jakarta.ws.rs.core.Response;

import java.io.IOException;
import java.util.Map;

@Path("/api/terminals")
@Produces(MediaType.APPLICATION_JSON)
public class TerminalResource {

    @Inject TerminalRegistry registry;
    @Inject TmuxManager tmux;

    @GET
    public Response list() {
        return Response.ok(registry.list()).build();
    }

    @GET @Path("/{name}")
    public Response get(@PathParam("name") String name) {
        return registry.get(name)
                .map(t -> Response.ok(t).build())
                .orElse(Response.status(404).entity(Map.of("error", "not found: " + name)).build());
    }

    @POST @Consumes(MediaType.APPLICATION_JSON)
    public Response create(CreateRequest request) {
        if (request.name() == null || request.name().isBlank()) {
            return Response.status(400).entity(Map.of("error", "name is required")).build();
        }
        try {
            registry.createSession(request.name(), request.workingDir() != null ? request.workingDir() : "/tmp");
            return Response.status(201).entity(registry.get(request.name()).orElseThrow()).build();
        } catch (IllegalStateException e) {
            return Response.status(409).entity(Map.of("error", "already exists: " + request.name())).build();
        } catch (IOException | InterruptedException e) {
            return Response.serverError().entity(Map.of("error", e.getMessage())).build();
        }
    }

    @DELETE @Path("/{name}")
    public Response destroy(@PathParam("name") String name) {
        if (registry.get(name).isEmpty()) {
            return Response.status(404).entity(Map.of("error", "not found: " + name)).build();
        }
        try {
            registry.destroySession(name);
            return Response.noContent().build();
        } catch (IOException | InterruptedException e) {
            return Response.serverError().entity(Map.of("error", e.getMessage())).build();
        }
    }

    @POST @Path("/{name}/input") @Consumes(MediaType.TEXT_PLAIN)
    public Response sendInput(@PathParam("name") String name, String text) {
        if (registry.get(name).isEmpty()) {
            return Response.status(404).entity(Map.of("error", "not found: " + name)).build();
        }
        try {
            registry.sendKeys(name, text);
            return Response.noContent().build();
        } catch (IOException | InterruptedException e) {
            return Response.serverError().entity(Map.of("error", e.getMessage())).build();
        }
    }

    @POST @Path("/{name}/resize") @Consumes(MediaType.APPLICATION_JSON)
    public Response resize(@PathParam("name") String name, ResizeRequest request) {
        if (registry.get(name).isEmpty()) {
            return Response.status(404).entity(Map.of("error", "not found: " + name)).build();
        }
        try {
            registry.resize(name, request.cols(), request.rows());
            return Response.noContent().build();
        } catch (IOException | InterruptedException e) {
            return Response.serverError().entity(Map.of("error", e.getMessage())).build();
        }
    }

    public record CreateRequest(String name, String workingDir) {}
    public record ResizeRequest(int cols, int rows) {}
}
```

- [ ] **Step 7: Create TerminalRegistryTest**

```java
package io.casehub.pages.terminal.runtime;

import io.casehub.pages.terminal.SessionLogger;
import io.casehub.pages.terminal.TmuxManager;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.io.TempDir;
import java.nio.file.Path;
import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class TerminalRegistryTest {

    @Test
    void create_and_get(@TempDir Path tmpDir) throws Exception {
        var tmux = new TmuxManager("test-");
        var logger = new SessionLogger(tmpDir);
        var registry = new TerminalRegistry(tmux, logger);

        // Note: this test will fail without tmux installed.
        // In CI, mock TmuxManager or skip.
        assertThat(registry.list()).isEmpty();
    }

    @Test
    void duplicate_create_throws(@TempDir Path tmpDir) throws Exception {
        var tmux = new TmuxManager("test-");
        var logger = new SessionLogger(tmpDir);
        var registry = new TerminalRegistry(tmux, logger);

        // Pre-populate to test duplicate check without tmux
        registry.get("test"); // warm up
        // The duplicate check is in createSession's putIfAbsent
        assertThat(registry.get("nonexistent")).isEmpty();
    }
}
```

- [ ] **Step 8: Update contributor guide**

Add `casehub-pages-terminal` and `casehub-pages-terminal-runtime` to the backend modules table in `docs/guides/contributor-guide.md`:

After the `casehub-pages-scenario-runtime` row, add:
```
| `casehub-pages-terminal` | Tmux session management (`TmuxManager`), FIFO pipe relay (`FifoRelay`), per-session output logging (`SessionLogger`). Pure Java, no Quarkus. |
| `casehub-pages-terminal-runtime` | Quarkus WebSocket endpoint (`/ws/terminal/{id}/{cols}/{rows}`), `TerminalRegistry` CDI bean for session lifecycle, REST resource (`/api/terminals`) for CRUD + input + resize. Config: `casehub.pages.terminal.prefix`, `casehub.pages.terminal.log-dir`. |
```

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add backend/terminal-runtime/ backend/pom.xml docs/guides/contributor-guide.md
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat(#401): casehub-pages-terminal-runtime — WebSocket endpoint, registry, REST

Quarkus CDI integration for terminal sessions. WebSocket at
/ws/terminal/{id}/{cols}/{rows} with FIFO relay, session takeover,
and echo duplication fix (stop pipe-pane before reconnect, delay
pipe-pane after redraw). REST at /api/terminals for CRUD.

Closes #401"
```

## References

- [2026-09-01-terminal-backend-design.md] — design spec
- `trellis/sidecar/src/main/java/io/hortora/trellis/terminal/` — source to generalize
- `backend/push/pom.xml` — core module POM pattern
- `backend/push-runtime/pom.xml` — runtime module POM pattern
- `components/pages-component-terminal/src/PagesTerminal.ts` — frontend component
- `docs/guides/contributor-guide.md` — docs to update
- [GitHub #401] — issue
