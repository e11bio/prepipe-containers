ARG SPARK_VERSION=3.3.2-scala2.12-java17-ubuntu24.04

# Stage 1: Build patched mpicbg (concurrent solver fix + deterministic tile order)
FROM azul/zulu-openjdk:8-latest AS mpicbg-builder
RUN apt-get update -y && apt-get install -y git maven curl

# multiview-reconstruction has one Swing panel that imports javafx, and its pom leaves the
# openjfx dependencies commented out: upstream builds it with a JDK that bundles JavaFX
# (their CI asks setup-java for zulu 8 'jdk+fx'). This image's JDK does not, so fetch that
# one and build with it. Pinned by exact version; the URL is Azul's public CDN.
ARG ZULU_FX_URL=https://cdn.azul.com/zulu/bin/zulu8.96.0.19-ca-fx-jdk8.0.502-linux_x64.tar.gz
RUN mkdir -p /opt/zulu-fx \
    && curl -sSL ${ZULU_FX_URL} | tar -xz -C /opt/zulu-fx --strip-components=1 \
    && test -f /opt/zulu-fx/jre/lib/ext/jfxrt.jar \
    && /opt/zulu-fx/bin/java -version
ENV JAVA_HOME=/opt/zulu-fx
ENV PATH=/opt/zulu-fx/bin:${PATH}

WORKDIR /build/mpicbg
ARG MPICBG_REPO=https://github.com/e11bio/mpicbg.git
# Pin to a specific commit (not a branch tip) so the container build is
# reproducible — branch-pinning bit us when an upstream fix wasn't picked up
# without a manual rebuild trigger.
ARG MPICBG_REF=7356b7bcc315f388f6243f68779f4fd880ef7ce6
RUN git init . \
    && git fetch --depth 1 ${MPICBG_REPO} ${MPICBG_REF} \
    && git checkout FETCH_HEAD \
    && mvn clean install -DskipTests -pl mpicbg

# Stage 2: Build patched multiview-reconstruction against that mpicbg
# (e11bio fork = the 8.0.0 release tag, the version BigStitcher-Spark builds against,
# plus the one commit that re-enables the concurrent tile optimizer)
FROM mpicbg-builder AS mvrecon-builder
WORKDIR /build/mvrecon
ARG MVRECON_REPO=https://github.com/e11bio/multiview-reconstruction.git
ARG MVRECON_REF=9f7b4d12ecc1fca7987f1a0bf38a466573e26b6a
ARG MPICBG_VERSION=1.6.7-SNAPSHOT
RUN git init . \
    && git fetch --depth 1 ${MVRECON_REPO} ${MVRECON_REF} \
    && git checkout FETCH_HEAD \
    && mvn clean install -DskipTests -Dmpicbg.mpicbg.version=${MPICBG_VERSION}

# Stage 3: Build BigStitcher-Spark fat jar against our mvrecon and mpicbg
FROM mvrecon-builder AS spark-builder
RUN curl -sSL https://bit.ly/install-xq | bash
WORKDIR /build/bigstitcher-spark
ARG BIGSTITCHER_SPARK_REPO=https://github.com/e11bio/BigStitcher-Spark.git
ARG BIGSTITCHER_SPARK_REF=5e1efb15b75096d988b4ed52dfb3854b9e071619
ARG MVRECON_VERSION=8.0.0
ARG MPICBG_VERSION=1.6.7-SNAPSHOT
RUN git init . \
    && git fetch --depth 1 ${BIGSTITCHER_SPARK_REPO} ${BIGSTITCHER_SPARK_REF} \
    && git checkout FETCH_HEAD \
    && mvn clean package -Pfatjar -DskipTests \
        -Dmultiview-reconstruction.version=${MVRECON_VERSION} \
        -Dmpicbg.mpicbg.version=${MPICBG_VERSION}

# Find the built jar
RUN xq -x '/project/artifactId' pom.xml > target-name \
    && xq -x '/project/version' pom.xml > target-version \
    && mv target/$(cat target-name)-$(cat target-version).jar /build/app.jar

# The fat jar must contain OUR mvrecon and OUR mpicbg, not the released ones. Both
# resolve from the local repository this image populated, which is easy to get wrong
# silently, so assert it rather than trusting the resolution order.
RUN javap -c -cp /build/app.jar net.preibisch.mvrecon.process.interestpointregistration.global.GlobalOptIterative \
        | grep -q "optimizeConcurrently" \
    && javap -p -cp /build/app.jar mpicbg.models.TileUtil | grep -q "optimizeColored" \
    && javap -p -cp /build/app.jar mpicbg.models.TileUtil | grep -q "defaultSeed" \
    && echo "fat jar carries the e11bio mvrecon and mpicbg"

# Stage 4: Final runtime image
FROM ghcr.io/janeliascicomp/spark:${SPARK_VERSION}

LABEL \
    org.opencontainers.image.title="BigStitcher Spark (E11 prepipe)" \
    org.opencontainers.image.description="BigStitcher-Spark built against the e11bio mpicbg and multiview-reconstruction forks" \
    org.opencontainers.image.authors="andrewc@e11.bio" \
    org.opencontainers.image.source="https://github.com/e11bio/prepipe-containers" \
    org.opencontainers.image.licenses="GPL-2.0-or-later"

USER root

RUN apt-get update -y || (sleep 30 && apt-get update -y) && \
    apt-get install -y --fix-missing \
        libblosc1 libblosc-dev \
        libzstd1 libzstd-dev libhdf5-dev && \
    rm -rf /var/lib/apt/lists/*

# /opt/scripts/runapp.sh (from the Janelia base image) expands
# ${additional_spark_config[@]} unquoted, so any --spark-conf value containing a space -
# two -D java options, say - is word-split and reaches spark-submit as stray arguments,
# which makes it print its usage and exit 1. That is how the solver's schedule/seed
# properties are passed, so quote it. The grep fails the build if the script changes
# shape and the sed silently stops matching.
RUN sed -i 's|^\( *\)${additional_spark_config\[@\]}$|\1"${additional_spark_config[@]}"|' /opt/scripts/runapp.sh \
    && grep -qF '"${additional_spark_config[@]}"' /opt/scripts/runapp.sh

WORKDIR /app
COPY --from=spark-builder /build/bigstitcher-spark/LICENSE /app/LICENSE
COPY --from=spark-builder /build/app.jar /app/app.jar
