ARG SPARK_VERSION=3.3.2-scala2.12-java17-ubuntu24.04

# Stage 1: Build patched mpicbg (concurrent solver fix)
FROM azul/zulu-openjdk:8-latest AS mpicbg-builder
RUN apt-get update -y && apt-get install -y git maven

WORKDIR /build/mpicbg
ARG MPICBG_REPO=https://github.com/e11bio/mpicbg.git
ARG MPICBG_REF=prepipe
RUN git clone --depth 1 --branch ${MPICBG_REF} ${MPICBG_REPO} . \
    && mvn clean install -DskipTests -pl mpicbg

# Stage 2: Build patched multiview-reconstruction (re-enabled concurrent solver)
FROM mpicbg-builder AS mvrecon-builder
WORKDIR /build/mvrecon
ARG MVRECON_REPO=https://github.com/e11bio/multiview-reconstruction.git
ARG MVRECON_REF=prepipe
RUN git clone --depth 1 --branch ${MVRECON_REF} ${MVRECON_REPO} . \
    && mvn clean install -DskipTests

# Stage 3: Build BigStitcher-Spark fat jar (against released deps)
FROM mvrecon-builder AS spark-builder
RUN apt-get install -y curl && curl -sSL https://bit.ly/install-xq | bash
WORKDIR /build/bigstitcher-spark
ARG BIGSTITCHER_SPARK_REPO=https://github.com/e11bio/BigStitcher-Spark.git
ARG BIGSTITCHER_SPARK_REF=main
RUN git clone --depth 1 --branch ${BIGSTITCHER_SPARK_REF} ${BIGSTITCHER_SPARK_REPO} . \
    && mvn clean package -Pfatjar -DskipTests

# Find the built jar
RUN xq -x '/project/artifactId' pom.xml > target-name \
    && xq -x '/project/version' pom.xml > target-version \
    && mv target/$(cat target-name)-$(cat target-version).jar /build/app.jar

# Stage 4: Patch the fat jar with our fixed classes
RUN cd /tmp \
    && jar xf /root/.m2/repository/mpicbg/mpicbg/*/mpicbg-*.jar \
        mpicbg/models/TileUtil.class \
        mpicbg/models/TileUtil\$1.class \
        mpicbg/models/TileUtil\$1\$1.class \
    && jar xf /root/.m2/repository/net/preibisch/multiview-reconstruction/*/multiview-reconstruction-*.jar \
        net/preibisch/mvrecon/process/interestpointregistration/global/GlobalOptIterative.class \
    && jar uf /build/app.jar \
        mpicbg/models/TileUtil.class \
        mpicbg/models/TileUtil\$1.class \
        mpicbg/models/TileUtil\$1\$1.class \
        net/preibisch/mvrecon/process/interestpointregistration/global/GlobalOptIterative.class

# Stage 5: Final runtime image
FROM ghcr.io/janeliascicomp/spark:${SPARK_VERSION}

LABEL \
    org.opencontainers.image.title="BigStitcher Spark (E11 prepipe)" \
    org.opencontainers.image.description="BigStitcher-Spark with concurrent tile optimization fix" \
    org.opencontainers.image.authors="andrewc@e11.bio" \
    org.opencontainers.image.source="https://github.com/e11bio/prepipe-containers" \
    org.opencontainers.image.licenses="GPL-2.0-or-later"

USER root

RUN apt-get update -y || (sleep 30 && apt-get update -y) && \
    apt-get install -y --fix-missing \
        libblosc1 libblosc-dev \
        libzstd1 libzstd-dev libhdf5-dev && \
    rm -rf /var/lib/apt/lists/*

WORKDIR /app
COPY --from=spark-builder /build/bigstitcher-spark/LICENSE /app/LICENSE
COPY --from=spark-builder /build/app.jar /app/app.jar
