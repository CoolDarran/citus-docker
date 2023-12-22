## Mongo FDW
FROM postgres:15.0 as mongo-fdw-builder
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

## Clickhouse FDW
FROM postgres:15.0 as clickhouse-fdw-builder
ARG CLICKHOUSE_FDW_COMMIT_HASH=1c1235e
RUN apt-get update \
    # clickhouse_fdw build
    && apt-get install -y --no-install-recommends openssl ca-certificates libssl-dev postgresql-server-dev-15 libcurl4-openssl-dev automake make gcc cmake autoconf pkg-config libtool uuid-dev git build-essential \
    && cd /tmp \
    && git clone https://github.com/adjust/clickhouse_fdw.git clickhouse_fdw \
    && cd clickhouse_fdw \
    && git reset --hard ${CLICKHOUSE_FDW_COMMIT_HASH} \
    && mkdir build && cd build \
    && cmake .. \
    && make && make install \
    && cd - && rm -rf /tmp/clickhouse_fdw

## HDFS FDW
FROM postgres:15.0 as hdfs-fdw-builder
ARG HDFS_FDW_VERSION=v2.2.0
ARG HDFS_FDW_URL=https://github.com/EnterpriseDB/hdfs_fdw/archive/refs/tags/${HDFS_FDW_VERSION}.tar.gz
ARG THRIFT_VERSION=0.16.0
RUN apt-get update \
    && apt-get install -y --no-install-recommends default-jdk ant wget ca-certificates openssl libssl-dev make gcc g++ cmake autoconf automake pkg-config file postgresql-server-dev-15 libssl-dev libzstd-dev libboost-all-dev libtool \
    && mkdir -p /tmp/hdfs_fdw \
    && wget -O - ${HDFS_FDW_URL} | tar -zx --strip-components=1 -C /tmp/hdfs_fdw \
    && cd /tmp/hdfs_fdw \
    && export JDK_INCLUDE=/usr/lib/jvm/java-11-openjdk-amd64/include \
    ## thrift
    && rm -rf thrift-${THRIFT_VERSION}.tar.gz && wget https://archive.apache.org/dist/thrift/${THRIFT_VERSION}/thrift-${THRIFT_VERSION}.tar.gz \
    && tar -zxvf thrift-${THRIFT_VERSION}.tar.gz && cd thrift-${THRIFT_VERSION} \
    && export CXXFLAGS="-fPIC" \
    && mkdir -p /opt/packages/thrift \
    && ./configure --prefix="/opt/packages/thrift" -with-qt4=no --with-c_glib=no --with-csharp=no --with-java=no --with-erlang=no --with-nodejs=no --with-lua=no --with-python=no --with-perl=no --with-php=no --with-php_extension=no --with-ruby=no --with-haskell=no --with-go=no --with-d=no \
    && make && make install \
    && cd .. \
    ## libhive
    # C++
    && cd libhive \
    && mkdir -p /opt/packages/libhive \
    && export INSTALL_DIR=/opt/packages/libhive \
    && make \
    && make install \
    # Java
    && cd jdbc \
    && javac MsgBuf.java \
    && javac HiveJdbcClient.java \
    && jar cf HiveJdbcClient-1.0.jar *.class \
    && cp HiveJdbcClient-1.0.jar /opt/packages/libhive \
    ## all
    && cd ../.. \
    && make USE_PGXS=1 \
    && make USE_PGXS=1 install \
    && cd - && rm -rf /tmp/hdfs_fdw \
    ## hadoop jar
    && mkdir -p /opt/packages/hadoop \
    && cd /opt/packages/hadoop && wget https://github.com/timveil/hive-jdbc-uber-jar/releases/download/v1.9-2.6.5/hive-jdbc-uber-2.6.5.0-292.jar

## jdbc fdw
FROM postgres:15.0 as jdbc-fdw-builder
ARG JDBC_FDW_VERSION=v0.2.0
ARG JDBC_FDW_URL=https://github.com/pgspider/jdbc_fdw/archive/refs/tags/${JDBC_FDW_VERSION}.tar.gz
RUN apt-get update \
    && apt-get install -y --no-install-recommends default-jdk wget ca-certificates make gcc cmake pkg-config postgresql-server-dev-15 \
    && mkdir -p /tmp/jdbc_fdw \
    && wget -O - ${JDBC_FDW_URL} | tar -zx --strip-components=1 -C /tmp/jdbc_fdw \
    && cd /tmp/jdbc_fdw \
    && ln -s /usr/lib/jvm/java-11-openjdk-amd64/lib/server/libjvm.so /usr/lib/libjvm.so \
    && USE_PGXS=1 LIBDIR=/usr/lib make clean \
    && USE_PGXS=1 LIBDIR=/usr/lib make install

## Citus
FROM postgres:15.0
ARG VERSION=11.1.2
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
## clickhouse fdw
COPY --from=clickhouse-fdw-builder /usr/lib/postgresql/15/lib/ /usr/lib/postgresql/15/lib/
COPY --from=clickhouse-fdw-builder /usr/share/postgresql/15/extension/ /usr/share/postgresql/15/extension/
## hdfs-fdw
COPY --from=hdfs-fdw-builder /usr/lib/postgresql/15/lib/ /usr/lib/postgresql/15/lib/
COPY --from=hdfs-fdw-builder /usr/share/postgresql/15/extension/ /usr/share/postgresql/15/extension/
# thrift
COPY --from=hdfs-fdw-builder /opt/packages/thrift/bin/ /usr/local/bin/
COPY --from=hdfs-fdw-builder /opt/packages/thrift/include/ /usr/local/include/
COPY --from=hdfs-fdw-builder /opt/packages/thrift/lib/ /usr/local/lib/
# libhive
COPY --from=hdfs-fdw-builder /opt/packages/libhive/libhive.so /lib/libhive.so
COPY --from=hdfs-fdw-builder /opt/packages/libhive/HiveJdbcClient-1.0.jar /opt/packages/libhive/HiveJdbcClient-1.0.jar
# hadoop uber jar
COPY --from=hdfs-fdw-builder /opt/packages/hadoop /opt/packages/hadoop
## jdbc-fdw
# TODO 是否需要内置JDBC Driver Jar?
COPY --from=jdbc-fdw-builder /usr/lib/postgresql/15/lib/ /usr/lib/postgresql/15/lib/
COPY --from=jdbc-fdw-builder /usr/share/postgresql/15/extension/ /usr/share/postgresql/15/extension/

RUN localedef -i zh_CN -c -f UTF-8 -A /usr/share/locale/locale.alias zh_CN.UTF-8 \
    && ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
ENV LANG zh_CN.UTF-8
ENV LC_COLLATE 'zh_CN.UTF-8'
ENV LC_CTYPE 'zh_CN.UTF-8'

# install Citus and needed extension on hyperscale
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
       ca-certificates \
       curl \
    && curl -s https://install.citusdata.com/community/deb.sh | bash \
    && apt-get install -y postgresql-$PG_MAJOR-citus-11.1=$CITUS_VERSION \
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
    && apt-get purge -y --auto-remove curl \
    && rm -rf /var/lib/apt/lists/*

# set shared_preload_libraries to the list of extensions that needs to be loaded when postgresql starts
RUN echo "shared_preload_libraries='citus,pg_cron,pg_stat_statements'" >> /usr/share/postgresql/postgresql.conf.sample

# set default database name for pg_cron extension
RUN echo "cron.database_name = 'citus'" >> /usr/share/postgresql/postgresql.conf.sample

# hdfs_fwd
RUN echo "hdfs_fdw.jvmpath='/usr/lib/jvm/java-11-openjdk-amd64/lib/server/'" >> /usr/share/postgresql/postgresql.conf.sample
RUN echo "hdfs_fdw.classpath='/opt/packages/libhive/HiveJdbcClient-1.0.jar:/opt/packages/hadoop/hive-jdbc-uber-2.6.5.0-292.jar'" >> /usr/share/postgresql/postgresql.conf.sample

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

