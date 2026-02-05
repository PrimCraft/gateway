FROM alpine:3.21 AS resolver

RUN apk add --no-cache curl

WORKDIR /build

# Download scaf (auto-detect architecture)
ARG SCAF_VERSION=0.1.0
RUN ARCH=$(uname -m) && \
    case "$ARCH" in x86_64) SCAF_ARCH="amd64";; aarch64) SCAF_ARCH="arm64";; *) exit 1;; esac && \
    curl -fsSL "https://github.com/PrimCraft/scaf/releases/download/v${SCAF_VERSION}/scaf_${SCAF_VERSION}_linux_${SCAF_ARCH}.tar.gz" | tar xz && \
    chmod +x scaf

COPY plugins.yaml .

# Resolve plugin versions (for fresh builds without lock file)
# In CI, we use the pre-resolved lock file instead
ARG USE_LOCK_FILE=false
COPY plugins.lock.yaml* ./
RUN if [ "$USE_LOCK_FILE" = "true" ] && [ -f plugins.lock.yaml ]; then \
        cp plugins.lock.yaml resolved.yaml; \
    else \
        ./scaf resolve --output resolved.yaml; \
    fi

# ------------------------------------------------------------------------------
FROM eclipse-temurin:21-jre-alpine AS downloader

RUN apk add --no-cache curl jq yq

WORKDIR /build
COPY --from=resolver /build/resolved.yaml .

# Download Velocity
RUN VELOCITY_URL=$(yq -r '.velocity.url' resolved.yaml) && \
    curl -fsSL -o velocity.jar "$VELOCITY_URL"

# Download plugins
RUN mkdir -p plugins && \
    for plugin in $(yq -r '.plugins | keys | .[]' resolved.yaml); do \
        url=$(yq -r ".plugins.${plugin}.url" resolved.yaml); \
        if [ "$url" != "null" ] && [ -n "$url" ]; then \
            echo "Downloading $plugin from $url"; \
            curl -fsSL -o "plugins/${plugin}.jar" "$url"; \
        fi; \
    done

# ------------------------------------------------------------------------------
FROM eclipse-temurin:21-jre-alpine

RUN apk add --no-cache tini

# Create velocity user
RUN addgroup -g 1000 velocity && \
    adduser -u 1000 -G velocity -h /velocity -D velocity

WORKDIR /velocity

# Copy artifacts
COPY --from=downloader /build/velocity.jar .
COPY --from=downloader /build/plugins ./plugins/
COPY docker/server-icon.png .

# Config will be mounted via ConfigMap
# forwarding.secret will be mounted via Secret

RUN chown -R velocity:velocity /velocity

USER velocity

EXPOSE 25565

ENTRYPOINT ["/sbin/tini", "--"]
CMD ["java", "-Xms512M", "-Xmx512M", "-XX:+UseG1GC", "-XX:G1HeapRegionSize=4M", \
     "-XX:+UnlockExperimentalVMOptions", "-XX:+ParallelRefProcEnabled", \
     "-XX:+AlwaysPreTouch", "-XX:MaxInlineLevel=15", "-jar", "velocity.jar"]
