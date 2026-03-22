# ARG POSTGRES_VERSION=alpine
ARG POSTGRES_VERSION=18-bookworm
ARG PG_SEARCH_VERSION=0.22.2
FROM postgres:${POSTGRES_VERSION}

ARG PG_SEARCH_VERSION

RUN set -eux; \
    arch="$(dpkg --print-architecture)"; \
    case "${arch}" in \
      amd64|arm64) ;; \
      *) echo "Unsupported architecture for pg_search: ${arch}" >&2; exit 1 ;; \
    esac; \
    . /etc/os-release; \
    distro="${VERSION_CODENAME}"; \
    pg_major="${PG_MAJOR:-$(psql --version | awk '{print $3}' | cut -d. -f1)}"; \
    apt-get update; \
    apt-get install -y --no-install-recommends ca-certificates curl; \
    curl -fL "https://github.com/paradedb/paradedb/releases/download/v${PG_SEARCH_VERSION}/postgresql-${pg_major}-pg-search_${PG_SEARCH_VERSION}-1PARADEDB-${distro}_${arch}.deb" -o /tmp/pg_search.deb; \
    apt-get install -y --no-install-recommends /tmp/pg_search.deb; \
    rm -f /tmp/pg_search.deb; \
    rm -rf /var/lib/apt/lists/*

CMD ["postgres"]

EXPOSE 5432
