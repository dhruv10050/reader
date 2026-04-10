# ADR 6: Multi-Platform Distribution Strategy

## Status
Accepted

## Context
Sismics Reader targets a broad audience of users across different operating systems and deployment preferences. The application needs to be easily installable on Windows, macOS, Linux (Debian-based and Red Hat-based), Docker containers, and as a standalone Java application. Each platform has its own packaging conventions, installation mechanisms, and user expectations.

Options considered:
- **Single JAR distribution only** — Simplest approach but requires users to have Java installed and manage startup manually.
- **Docker-only distribution** — Modern and portable but excludes users unfamiliar with container technology.
- **Platform-native packages only** — Best user experience per platform but requires maintaining multiple packaging pipelines.
- **Multi-format distribution** — Support all major platforms with native packaging formats plus Docker and standalone options.

## Decision
We will maintain **six distribution modules** as Maven sub-projects, each producing platform-appropriate packages:

1. **reader-distribution-standalone** — Standalone executable JAR with embedded Jetty for any platform with Java installed.
2. **reader-distribution-debian** — `.deb` package using jDeb Maven plugin with systemd service integration, for Debian/Ubuntu systems.
3. **reader-distribution-redhat** — `.rpm` package using RPM Maven plugin with service configuration, for RHEL/CentOS/Fedora systems.
4. **reader-distribution-mac** — `.app` bundle and `.dmg` image using OSXAppBundle Maven plugin with native macOS integration.
5. **reader-distribution-windows** — `.exe` installer using NSIS Maven plugin and Launch4j wrapper with system tray integration.
6. **reader-distribution-docker** — Docker image based on `sismics/jetty:9.3.11` with Docker Compose support.

Distribution modules are only built under the `prod` Maven profile to keep development builds fast.

## Consequences

### Positive
- **Wide platform coverage**: Users on any major OS can install Reader using their platform's native package manager or preferred method.
- **Native integration**: Platform-specific packages include proper service management (systemd, Windows services), system tray icons, and menu integration.
- **Docker support**: Enables easy deployment on servers and cloud platforms with minimal configuration.
- **Separation of concerns**: Each distribution module is isolated, allowing platform-specific customization without affecting the core application.
- **Conditional builds**: The `prod` profile ensures distribution packaging doesn't slow down development builds.

### Negative
- **High maintenance burden**: Six packaging formats require maintaining platform-specific build configurations, scripts, and testing across all target platforms.
- **Build tool complexity**: Multiple Maven plugins (jDeb, RPM, NSIS, OSXAppBundle, Launch4j) each have their own configuration syntax and quirks.
- **Platform-specific testing**: Each distribution format needs testing on its target platform, which is difficult to automate in a single CI environment.
- **Version synchronization**: All six distribution modules must stay in sync with the core application version, increasing release management complexity.
