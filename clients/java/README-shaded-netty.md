# camunda-client-java with shaded Netty

This module ships a Maven profile, `shade-netty`, that produces a build of
`io.camunda:camunda-client-java` in which Netty and `io.grpc:grpc-netty`
are repackaged into an internal namespace:

| Original package | Shaded package                           |
|------------------|------------------------------------------|
| `io.netty`       | `io.camunda.client.shaded.io.netty`      |
| `io.grpc.netty`  | `io.camunda.client.shaded.io.grpc.netty` |

`io.netty.internal.tcnative` is intentionally **not** relocated, because
`netty-tcnative-boringssl-static` registers its JNI bindings against the
original fully-qualified class names. It is retained as a runtime dep of
the dependency-reduced POM.

The shaded jar is published under the same `groupId:artifactId`
(`io.camunda:camunda-client-java`) but at whichever version you pass in
`-Dshaded.deploy.version`. The build **does not** change
`project.version` itself — that would break resolution of the internal
`zeebe-*` dependencies, which the Camunda parent declares with
`${project.version}`. Instead, the profile post-processes
`dependency-reduced-pom.xml` and invokes `install:install-file` /
`deploy:deploy-file` under the custom coordinates.

## Building and installing locally

```bash
./mvnw -pl clients/java -am -Dquickly \
  -Pshade-netty \
  -Dshaded.deploy.version=8.9.0-netty42.1 \
  install
```

This produces `clients/java/target/camunda-client-java-8.9.0.jar`
(shaded) and installs it into the local Maven repo under
`io.camunda:camunda-client-java:8.9.0-netty42.1`.

Pick any `shaded.deploy.version` scheme you like —
`8.9.0-netty42.<n>` keeps the relationship to upstream visible.

### Verifying the shaded output

```bash
unzip -l clients/java/target/camunda-client-java-8.9.0.jar \
  | awk '{print $4}' | grep -E "^io/(netty|grpc/netty)/"
```

Should print nothing — original Netty/grpc-netty packages are fully
relocated.

```bash
unzip -l clients/java/target/camunda-client-java-8.9.0.jar \
  | awk '{print $4}' \
  | grep -c "^io/camunda/client/shaded/io/netty/"
# ~2500+ classes
```

Check the service registration for gRPC's transport providers:

```bash
unzip -p clients/java/target/camunda-client-java-8.9.0.jar \
  META-INF/services/io.grpc.ManagedChannelProvider
# io.camunda.client.shaded.io.grpc.netty.NettyChannelProvider
# io.camunda.client.shaded.io.grpc.netty.UdsNettyChannelProvider
```

And the reduced pom — the embedded artifacts should **not** appear:

```bash
grep -E "netty-handler|netty-common|grpc-netty" \
  clients/java/target/shaded-deploy-pom.xml
# (no output)
```

## Deploying to your internal Maven repository

1. Add a matching `<server>` entry to `~/.m2/settings.xml` so credentials
   are available:

   ```xml
   <settings>
     <servers>
       <server>
         <id>internal-releases</id>
         <username>${env.MVN_REPO_USER}</username>
         <password>${env.MVN_REPO_PASSWORD}</password>
       </server>
     </servers>
   </settings>
   ```

2. Deploy in **two steps** — first install the upstream reactor modules
   locally, then deploy only `clients/java` so the upstream modules never
   hit the `deploy` phase:

   ```bash
   # Step 1: install upstream + client locally (no deploy phase reached).
   ./mvnw -pl clients/java -am -Dquickly \
     -Pshade-netty \
     -Dshaded.deploy.version=8.9.0-netty42.1 \
     install

   # Step 2: deploy only the shaded client. No -am, so upstream modules
   # are resolved from the local repo populated in step 1.
   ./mvnw -pl clients/java -Dquickly \
     -Pshade-netty \
     -Dshaded.deploy.version=8.9.0-netty42.1 \
     -Dshaded.deploy.repositoryId=internal-releases \
     -Dshaded.deploy.url=https://maven.example.com/repository/releases/ \
     deploy
   ```

   Why two steps? The Camunda BOM registers `nexus-staging-maven-plugin`
   with `<extensions>true</extensions>`, which hijacks the `deploy` phase
   and pushes to `artifacts.camunda.com`. The `shade-netty` profile
   suppresses it for the client module, but upstream modules (e.g.
   `zeebe-gateway-protocol`) do not activate this profile — running
   `-am deploy` in a single shot would try to upload them to the upstream
   repo and fail with `401 Unauthorized`.

The profile deliberately does not hard-code a distribution URL — pass
`shaded.deploy.url` (and `shaded.deploy.repositoryId` for credentials
lookup) on every deploy. A snapshot deploy is just a matter of pointing
`shaded.deploy.url` at the snapshot repo:

```bash
# Step 1 is the same as above.

# Step 2, for snapshots:
./mvnw -pl clients/java -Dquickly \
  -Pshade-netty \
  -Dshaded.deploy.version=8.9.0-netty42.1-SNAPSHOT \
  -Dshaded.deploy.repositoryId=internal-snapshots \
  -Dshaded.deploy.url=https://maven.example.com/repository/snapshots/ \
  deploy
```

## Consuming the shaded client

Downstream projects depend on the shaded client exactly as they would on
the upstream artifact — same coordinates, new version:

```xml
<dependency>
  <groupId>io.camunda</groupId>
  <artifactId>camunda-client-java</artifactId>
  <version>8.9.0-netty42.1</version>
</dependency>
```

Consumers that used to import `io.netty.*` or `io.grpc.netty.*` types
**transitively** through this client must now either:

- depend on Netty directly (recommended — the shaded copy is private to
  this client), or
- import from `io.camunda.client.shaded.io.netty.*` if they truly want to
  reuse the vendored copy.

## Implementation notes and caveats

- **Why not just bump `project.version`?** The Camunda parent's
  `dependencyManagement` declares `zeebe-bpmn-model`,
  `zeebe-gateway-protocol-impl`, and `zeebe-protocol` with
  `<version>${project.version}</version>`. Changing
  `camunda-client-java`'s `project.version` would therefore force Maven
  to resolve those deps at the custom version too, where they do not
  exist. The profile avoids the problem by leaving `project.version` at
  the tag's version (e.g. 8.9.0), then re-publishing with `deploy-file`.
- **Guava is declared explicitly.** `grpc-netty`'s static initializers
  reference `com.google.common.base.Supplier` and friends. Guava
  normally reaches the classpath as a transitive of `grpc-netty`, but
  once `grpc-netty` is vendored in, that transitive disappears, and the
  remaining grpc-\* modules in the Camunda parent's dep-mgmt `<exclusion>`
  Guava. The profile adds `com.google.guava:guava` at `runtime` scope so
  it shows up in the reduced pom.
- **tcnative stays external.** The relocation excludes
  `io.netty.internal.tcnative.**`, and the artifactSet excludes
  `netty-tcnative*`. The reduced pom retains `netty-tcnative-boringssl-static`
  (plus its platform classifiers) as runtime deps. Native TLS works
  out of the box.
- **Public API.** The client's own public API does not expose Netty
  types, so relocation is transparent to typical users. Advanced
  integrations that touched `NettyChannelBuilder` directly will need to
  import the shaded name.
- **No sources jar.** The profile skips `maven-source-plugin`'s
  `attach-sources` and sets `createSourcesJar=false` on the shade plugin.
  `deploy:deploy-file` would otherwise also publish the project's
  attached sources under `project.version` (8.9.0), next to the shaded
  main jar at the custom version — a strict repo rejects the
  version-mismatched upload. If you need sources, pull them from the
  upstream release at `io.camunda:camunda-client-java:8.9.0:sources`;
  the relocated namespace is only relevant for bytecode, so the upstream
  sources are a perfectly usable read-only reference.
- **revapi** is skipped in the profile — every relocated package looks
  like an API break. Run revapi against the non-shaded build before
  shading if you still want that signal.
- **GraalVM native image.** Netty's
  `META-INF/native-image/**` configuration is stripped during shading
  (it references the original package names). Re-run `native-image-agent`
  against your application if you rely on it.
