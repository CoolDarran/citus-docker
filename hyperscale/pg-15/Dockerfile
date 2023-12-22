## Mongo FDW
FROM postgres:15.3 as mongo-fdw-builder
ARG MONGO_FDW_VERSION=5_4_0
ARG MONGO_FDW_URL=https://github.com/EnterpriseDB/mongo_fdw/archive/REL-${MONGO_FDW_VERSION}.tar.gz
ARG SOURCE_FILES=/tmp/mongo_fdw
RUN apt-get update \
    && apt-get install -y --no-install-recommends wget ca-certificates make gcc cmake pkg-config postgresql-server-dev-15 libssl-dev libzstd-dev \
    # mongo_fdw build
    && export PKG_CONFIG_PATH=${SOURCE_FILES}/mongo-c-driver/src/libmongoc/src:${SOURCE_FILES}/mongo-c-driver/src/libbson/src \
    && export LD_LIBRARY_PATH=/lib \
    && mkdir -p ${SOURCE_FILES} \
    && wget -O - ${MONGO_FDW_URL} | tar -zx --strip-components=1 -C ${SOURCE_FILES} \
    && cd ${SOURCE_FILES} \
    # mongo_fdw install
    && ./autogen.sh --with-master \
    && make \
    && make install \
    && cd - && rm -rf ${SOURCE_FILES}

## jdbc fdw
FROM postgres:15.3 as jdbc-fdw-builder
ARG JDBC_FDW_VERSION=v0.3.1
ARG JDBC_FDW_URL=https://github.com/pgspider/jdbc_fdw/archive/refs/tags/${JDBC_FDW_VERSION}.tar.gz
RUN apt-get update \
    && apt-get install -y --no-install-recommends default-jdk wget ca-certificates make gcc cmake pkg-config postgresql-server-dev-15 \
    && mkdir -p /tmp/jdbc_fdw \
    && wget -O - ${JDBC_FDW_URL} | tar -zx --strip-components=1 -C /tmp/jdbc_fdw \
    && cd /tmp/jdbc_fdw \
    && ln -s /usr/lib/jvm/java-17-openjdk-amd64/lib/server/libjvm.so /usr/lib/libjvm.so \
    && USE_PGXS=1 LIBDIR=/usr/lib make clean \
    && USE_PGXS=1 LIBDIR=/usr/lib make install

## pgsql-http fdw
FROM postgres:15.3 as http-fdw-builder
ARG HTTP_FDW_VERSION=v1.6.0
ARG HTTP_FDW_URL=https://github.com/pramsey/pgsql-http/archive/refs/tags/${HTTP_FDW_VERSION}.tar.gz
RUN apt-get update \
    && apt-get install -y --no-install-recommends wget ca-certificates make g++ cmake pkg-config postgresql-server-dev-15 libcurl4-openssl-dev \
    && mkdir -p /tmp/http_fdw \
    && wget -O - ${HTTP_FDW_URL} | tar -zx --strip-components=1 -C /tmp/http_fdw \
    && cd /tmp/http_fdw \
    && USE_PGXS=1 LIBDIR=/usr/lib make \
    && USE_PGXS=1 LIBDIR=/usr/lib make install

## sqlite fdw
FROM postgres:15.3 as sqlite-fdw-builder
ARG SQLITE_FDW_VERSION=v2.3.0
ARG SQLITE_FDW_URL=https://github.com/pgspider/sqlite_fdw/archive/refs/tags/${SQLITE_FDW_VERSION}.tar.gz
RUN apt-get update \
    && apt-get install -y --no-install-recommends pgxnclient wget ca-certificates make g++ cmake pkg-config postgresql-server-dev-15 libsqlite3-dev \
    && mkdir -p /tmp/sqlite_fdw \
    && wget -O - ${SQLITE_FDW_URL} | tar -zx --strip-components=1 -C /tmp/sqlite_fdw \
    && cd /tmp/sqlite_fdw \
    && USE_PGXS=1 LIBDIR=/usr/lib make \
    && USE_PGXS=1 LIBDIR=/usr/lib make install

## Citus
FROM postgres:15.3
ARG VERSION=12.0.0
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
COPY --from=mongo-fdw-builder /usr/lib/postgresql/15/lib/ /usr/lib/postgresql/15/lib/
COPY --from=mongo-fdw-builder /usr/share/postgresql/15/extension/ /usr/share/postgresql/15/extension/
## jdbc-fdw
COPY --from=jdbc-fdw-builder /usr/lib/postgresql/15/lib/ /usr/lib/postgresql/15/lib/
COPY --from=jdbc-fdw-builder /usr/share/postgresql/15/extension/ /usr/share/postgresql/15/extension/
## http-fdw
COPY --from=http-fdw-builder /usr/lib/postgresql/15/lib/ /usr/lib/postgresql/15/lib/
COPY --from=http-fdw-builder /usr/share/postgresql/15/extension/ /usr/share/postgresql/15/extension/
## sqlite-fdw
COPY --from=sqlite-fdw-builder /usr/lib/postgresql/15/lib/ /usr/lib/postgresql/15/lib/
COPY --from=sqlite-fdw-builder /usr/share/postgresql/15/extension/ /usr/share/postgresql/15/extension/

# install Citus and needed extension on hyperscale
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
       ca-certificates \
       curl \
       libssl-dev libglib2.0-dev \
    && curl -s https://install.citusdata.com/community/deb.sh | bash \
    && apt-get install -y postgresql-$PG_MAJOR-citus-12.0=$CITUS_VERSION \
                          postgresql-$PG_MAJOR-hll=2.17.citus-1 \
                          postgresql-$PG_MAJOR-topn=2.5.0.citus-1 \
                          postgresql-15-postgis-3 \
                          postgresql-plperl-14 \
                          postgresql-contrib \
                          postgresql-15-partman \
                          postgresql-15-tdigest \
                          postgresql-15-mysql-fdw \
                          postgresql-15-cron \
                          default-jdk \
                          postgresql-15-pgvector \
    && ln -s /usr/lib/jvm/java-17-openjdk-amd64/lib/server/libjvm.so /usr/lib/libjvm.so \
    && apt-get purge -y --auto-remove curl \
    && rm -rf /var/lib/apt/lists/*

RUN localedef -i zh_CN -c -f UTF-8 -A /usr/share/locale/locale.alias zh_CN.UTF-8 \
    && ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
ENV LANG zh_CN.UTF-8
ENV LC_COLLATE 'zh_CN.UTF-8'
ENV LC_CTYPE 'zh_CN.UTF-8'

# set shared_preload_libraries to the list of extensions that needs to be loaded when postgresql starts
RUN echo "shared_preload_libraries='citus,pg_cron,pg_stat_statements'" >> /usr/share/postgresql/postgresql.conf.sample

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

