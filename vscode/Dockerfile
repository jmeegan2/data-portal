FROM codercom/code-server:latest

USER root
RUN apt-get update && \
    apt-get install -y python3 python3-pip python3-venv && \
    pip3 install --break-system-packages psycopg2-binary && \
    ln -sf /usr/bin/python3 /usr/bin/python && \
    ln -sf /usr/bin/pip3 /usr/bin/pip && \
    apt-get clean && rm -rf /var/lib/apt/lists/*
USER coder
