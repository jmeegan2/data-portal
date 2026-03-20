FROM rocker/rstudio:4.5.3

RUN apt-get update && \
    apt-get upgrade -y && \
    apt-get install -y libpq-dev && \
    R -e "install.packages('RPostgres', repos='https://cloud.r-project.org')" && \
    dpkg --force-depends --purge linux-libc-dev && \
    apt-get clean && rm -rf /var/lib/apt/lists/*

# Fix permissions so rstudio user (UID 1000) can run rserver without root
RUN chown -R rstudio:rstudio /etc/rstudio && \
    chmod 664 /etc/rstudio/database.conf && \
    chmod 666 /usr/local/lib/R/etc/Renviron.site && \
    mkdir -p /var/lib/rstudio-server /var/run/rstudio-server && \
    chown -R rstudio:rstudio /var/lib/rstudio-server /var/run/rstudio-server

USER rstudio
