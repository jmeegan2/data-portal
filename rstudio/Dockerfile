FROM rocker/rstudio:latest

RUN apt-get update && \
    apt-get install -y libpq-dev && \
    R -e "install.packages('RPostgres', repos='https://cloud.r-project.org')" && \
    apt-get clean && rm -rf /var/lib/apt/lists/*
