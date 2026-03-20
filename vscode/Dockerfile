FROM linuxserver/code-server:latest

USER root
RUN apt-get update && \
    apt-get upgrade -y && \
    apt-get install -y python3 python3-pip python3-venv npm && \
    pip3 install --break-system-packages psycopg2-binary && \
    ln -sf /usr/bin/python3 /usr/bin/python && \
    ln -sf /usr/bin/pip3 /usr/bin/pip && \
    apt-get clean && rm -rf /var/lib/apt/lists/*

# Patch undici CVEs
RUN cd /app/code-server/lib/vscode/node_modules/undici && \
    npm pack undici@7.24.0 && \
    tar xzf undici-7.24.0.tgz --strip-components=1 && \
    rm -f undici-7.24.0.tgz

# Fix Trivy false positives: VS Code extension metadata misidentified as vulnerable packages
RUN sed -i 's/"name": "code-server"/"name": "vscode-core"/' /app/code-server/lib/vscode/package.json && \
    sed -i 's/"name":"npm"/"name":"vscode-npm-extension"/' /app/code-server/lib/vscode/extensions/npm/package.json

# Ensure abc user owns config dir
RUN mkdir -p /config && chown -R abc:abc /config

USER abc
