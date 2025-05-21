## Mongo FDW
FROM postgres:16.9 as mongo-fdw-builder
ARG MONGO_FDW_VERSION=5_5_2
ARG MONGO_FDW_URL=https://github.com/EnterpriseDB/mongo_fdw/archive/REL-${MONGO_FDW_VERSION}.tar.gz
ARG MONGOC_VERSION=1.17.3
ARG MONGOC_URL=https://github.com/mongodb/mongo-c-driver/releases/download/${MONGOC_VERSION}/mongo-c-driver-${MONGOC_VERSION}.tar.gz
ARG JSONC_VERSION=0.15-20200726
ARG JSONC_URL=https://github.com/json-c/json-c/archive/json-c-${JSONC_VERSION}.tar.gz
ARG SOURCE_FILES=/tmp/mongo_fdw
RUN apt-get update \
    && apt-get install -y --no-install-recommends wget ca-certificates make gcc cmake pkg-config postgresql-server-dev-$PG_MAJOR libssl-dev libzstd-dev \
    && export PKG_CONFIG_PATH=/tmp/mongo-c-driver/src/libmongoc/src:/tmp/mongo-c-driver/src/libbson/src \
    && export LD_LIBRARY_PATH=/lib \
    && mkdir -p ${SOURCE_FILES} \
    && wget -O - ${MONGO_FDW_URL} | tar -zx --strip-components=1 -C ${SOURCE_FILES} \
    && cd ${SOURCE_FILES} \
    # mongo_fdw install
    && ./autogen.sh \
    && make USE_PGXS=1 \
    && make USE_PGXS=1 install \
    && cd - && rm -rf ${SOURCE_FILES}

## jdbc fdw
FROM postgres:16.9 as jdbc-fdw-builder
ARG JDBC_FDW_VERSION=v0.5.0
ARG JDBC_FDW_URL=https://github.com/pgspider/jdbc_fdw/archive/refs/tags/${JDBC_FDW_VERSION}.tar.gz
RUN apt-get update \
    && apt-get install -y --no-install-recommends default-jdk wget ca-certificates make gcc cmake pkg-config postgresql-server-dev-$PG_MAJOR \
    && mkdir -p /tmp/jdbc_fdw \
    && wget -O - ${JDBC_FDW_URL} | tar -zx --strip-components=1 -C /tmp/jdbc_fdw \
    && cd /tmp/jdbc_fdw \
    && ln -s /usr/lib/jvm/java-17-openjdk-amd64/lib/server/libjvm.so /usr/lib/libjvm.so \
    && USE_PGXS=1 LIBDIR=/usr/lib make clean \
    && USE_PGXS=1 LIBDIR=/usr/lib make install

## pgai
FROM postgres:16.9 as pgai-builder
ARG PGAI_VERSION=extension-0.10.5
ARG PGAI_URL=https://github.com/timescale/pgai/archive/refs/tags/${PGAI_VERSION}.tar.gz
RUN apt-get update \
    && apt-get install -y --no-install-recommends wget ca-certificates make gcc cmake pkg-config git curl vim \
        postgresql-plpython3-$PG_MAJOR \
        postgresql-$PG_MAJOR-pgextwlist \
        postgresql-server-dev-$PG_MAJOR \
        postgresql-$PG_MAJOR-pgvector \
        python3-pip \
    && mkdir -p /tmp/pgai \
    && wget -O - ${PGAI_URL} | tar -zx --strip-components=1 -C /tmp/pgai \
    && cd /tmp/pgai \
    && USE_PGXS=1 LIBDIR=/usr/lib PG_BIN=/usr/lib/postgresql/16/bin projects/extension/build.py install

## pg_duckdb
FROM postgres:16.9 as duckdb-builder
ARG PG_DUCKDB_VERSION=v0.3.1
ARG PG_DUCKDB_URL=https://github.com/duckdb/pg_duckdb/archive/refs/tags/${PG_DUCKDB_VERSION}.tar.gz
RUN apt-get update \
    && apt-get install -y --no-install-recommends liblz4-dev wget ca-certificates make pkg-config git g++ cmake ninja-build libssl-dev postgresql-server-dev-$PG_MAJOR \
    && cd /tmp && git clone --depth 1 --branch ${PG_DUCKDB_VERSION} https://github.com/duckdb/pg_duckdb.git \
    && cd /tmp/pg_duckdb \
    && USE_PGXS=1 LIBDIR=/usr/lib make clean \
    && USE_PGXS=1 LIBDIR=/usr/lib make install

## Citus
FROM postgres:16.9
ARG VERSION=12.1.6
LABEL maintainer="Citus Data https://citusdata.com" \
      org.label-schema.name="Citus" \
      org.label-schema.description="Scalable PostgreSQL for multi-tenant and real-time workloads" \
      org.label-schema.url="https://www.citusdata.com" \
      org.label-schema.vcs-url="https://github.com/citusdata/citus" \
      org.label-schema.vendor="Citus Data, Inc." \
      org.label-schema.version=${VERSION} \
      org.label-schema.schema-version="1.0"

ENV CITUS_VERSION ${VERSION}.citus-1
ENV POSTGRES_DB citus
ENV PGDATA /data

## mongo fdw
COPY --from=mongo-fdw-builder /lib/ /lib/
COPY --from=mongo-fdw-builder /usr/lib/postgresql/16/lib/ /usr/lib/postgresql/16/lib/
COPY --from=mongo-fdw-builder /usr/share/postgresql/16/extension/ /usr/share/postgresql/16/extension/
## jdbc-fdw
COPY --from=jdbc-fdw-builder /usr/lib/postgresql/16/lib/ /usr/lib/postgresql/16/lib/
COPY --from=jdbc-fdw-builder /usr/share/postgresql/16/extension/ /usr/share/postgresql/16/extension/
## pgai
COPY --from=pgai-builder /usr/lib/postgresql/16/lib/ /usr/lib/postgresql/16/lib/
COPY --from=pgai-builder /usr/share/postgresql/16/extension/ /usr/share/postgresql/16/extension/
COPY --from=pgai-builder /usr/local/lib/ /usr/local/lib/
COPY --from=pgai-builder /usr/local/bin/ /usr/local/bin/
## pg_duckdb
COPY --from=duckdb-builder /usr/lib/postgresql/16/lib/ /usr/lib/postgresql/16/lib/
COPY --from=duckdb-builder /usr/share/postgresql/16/extension/ /usr/share/postgresql/16/extension/

# install Citus and needed extension on hyperscale
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
       ca-certificates \
       curl \
       libssl-dev libglib2.0-dev liblz4-dev \
    && curl -s https://install.citusdata.com/community/deb.sh | bash \
    && apt-get install -y postgresql-$PG_MAJOR-citus-12.1=$CITUS_VERSION \
                          postgresql-$PG_MAJOR-hll=2.18.citus-1 \
                          postgresql-$PG_MAJOR-topn=2.7.0.citus-1 \
                          postgresql-$PG_MAJOR-postgis-3 \
                          postgresql-plperl-$PG_MAJOR \
                          postgresql-contrib \
                          postgresql-$PG_MAJOR-partman \
                          postgresql-$PG_MAJOR-tdigest \
                          postgresql-$PG_MAJOR-mysql-fdw \
                          postgresql-$PG_MAJOR-cron \
                          default-jdk \
                          postgresql-plpython3-$PG_MAJOR \
                          postgresql-$PG_MAJOR-pgextwlist \
                          postgresql-$PG_MAJOR-pgvector \
                          python3-pip \
    && ln -s /usr/lib/jvm/java-17-openjdk-amd64/lib/server/libjvm.so /usr/lib/libjvm.so \
    && apt-get purge -y --auto-remove curl \
    && rm -rf /var/lib/apt/lists/*

## Using PostgreSQL Extension Repo by Pigsty
RUN apt-get update \
    && apt-get install -y --no-install-recommends curl ca-certificates libssl-dev libglib2.0-dev \
    && echo "deb [trusted=yes] https://repo.pigsty.io/apt/infra generic main" > /etc/apt/sources.list.d/pigsty.list \
    && apt-get update \
    && apt-get install -y pig \
    && pig repo add pigsty -u && pig repo update \
    && apt-get install -y postgresql-$PG_MAJOR-http \
                          postgresql-$PG_MAJOR-sqlite-fdw postgresql-$PG_MAJOR-wrappers \
                          postgresql-$PG_MAJOR-h3 postgresql-$PG_MAJOR-pg-polyline postgresql-$PG_MAJOR-pg-geohash \
                          postgresql-$PG_MAJOR-pg-jsonschema postgresql-$PG_MAJOR-pg-parquet \
    && apt-get purge -y --auto-remove curl \
    && rm -rf /var/lib/apt/lists/*

RUN localedef -i zh_CN -c -f UTF-8 -A /usr/share/locale/locale.alias zh_CN.UTF-8 \
    && ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
ENV LANG zh_CN.UTF-8
ENV LC_COLLATE 'zh_CN.UTF-8'
ENV LC_CTYPE 'zh_CN.UTF-8'

# set shared_preload_libraries to the list of extensions that needs to be loaded when postgresql starts
RUN echo "shared_preload_libraries='citus,pg_cron,pg_stat_statements,pg_duckdb'" >> /usr/share/postgresql/postgresql.conf.sample

# set default database name for pg_cron extension
RUN echo "cron.database_name = 'citus'" >> /usr/share/postgresql/postgresql.conf.sample

# add scripts to run after initdb
COPY hyperscale/001-db-user-creation.sql /docker-entrypoint-initdb.d/
COPY hyperscale/002-create_extension_functions.sql /docker-entrypoint-initdb.d/
COPY hyperscale/003-create_citus_extension.sql /docker-entrypoint-initdb.d/
COPY hyperscale/004-postgres_replication_origins.sql /docker-entrypoint-initdb.d/
COPY hyperscale/005-other_extensions.sql /docker-entrypoint-initdb.d/
COPY hyperscale/006-pg_cron_pg_partman_setup.sql /docker-entrypoint-initdb.d/
COPY hyperscale/007-grant_usage_fdw.sql /docker-entrypoint-initdb.d/
COPY hyperscale/008-other_grants.sql /docker-entrypoint-initdb.d/
COPY hyperscale/009-stat_statements_reset.sql /docker-entrypoint-initdb.d/

# copy access rules for postgres
COPY hyperscale/pg_hba.conf /pg_hba.conf

# add health check script
COPY pg_healthcheck wait-for-manager.sh /

# required for to replace citus user password in the execution of docker-entrypoint.sh
RUN chown -R postgres:postgres /docker-entrypoint-initdb.d

COPY hyperscale/docker-entrypoint.sh /usr/local/bin/docker-entrypoint.sh

HEALTHCHECK --interval=4s --start-period=6s CMD ./pg_healthcheck

