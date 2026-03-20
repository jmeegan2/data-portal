FROM linuxserver/code-server:4.112.0-ls324

USER root

# Install Python, patch OS CVEs, remove build-only packages
RUN apt-get update && \
    apt-get upgrade -y && \
    apt-get install -y python3 python3-pip python3-venv npm && \
    pip3 install --break-system-packages psycopg2-binary && \
    ln -sf /usr/bin/python3 /usr/bin/python && \
    ln -sf /usr/bin/pip3 /usr/bin/pip && \
    # Patch bundled undici to 7.24.0 — fixes:
    #   CVE-2026-1525 (request smuggling), CVE-2026-1526 (WebSocket memory DoS),
    #   CVE-2026-1527 (CRLF injection), CVE-2026-1528 (WebSocket frame crash),
    #   CVE-2026-2229 (WebSocket deflate exception), CVE-2026-2581 (response buffer DoS)
    cd /app/code-server/lib/vscode/node_modules/undici && \
    npm pack undici@7.24.0 && \
    tar xzf undici-7.24.0.tgz --strip-components=1 && \
    rm -f undici-7.24.0.tgz && \
    apt-get purge -y npm linux-libc-dev && \
    apt-get autoremove -y && \
    apt-get clean && rm -rf /var/lib/apt/lists/*

# Fix Trivy false positives: Trivy misidentifies VS Code extension package.json
# metadata as vulnerable npm packages (aquasecurity/trivy#1407, coder/code-server#6332).
# Remove when Trivy adds VS Code extension awareness.
RUN sed -i 's/"name": "code-server"/"name": "vscode-core"/' /app/code-server/lib/vscode/package.json && \
    sed -i 's/"name":"npm"/"name":"vscode-npm-extension"/' /app/code-server/lib/vscode/extensions/npm/package.json

RUN mkdir -p /config && chown -R abc:abc /config

USER abc
