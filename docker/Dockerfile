FROM alpine

ARG BUILD_DATE
ARG VCS_REF

LABEL maintainer="mel@hyena.network" \
    org.opencontainers.image.title="Akkoma" \
    org.opencontainers.image.description="Akkoma containerized" \
    org.opencontainers.image.authors="mel@hyena.network" \
    org.opencontainers.image.vendor="soc.hyena.network" \
    org.opencontainers.image.documentation="https://docs.akkoma.dev/stable" \
    org.opencontainers.image.licenses="AGPL-3.0" \
    org.opencontainers.image.url="https://akkoma.dev" \
    org.opencontainers.image.revision=$VCS_REF \
    org.opencontainers.image.created=$BUILD_DATE

# Set up environment
ENV LC_ALL=C.UTF-8
ENV LANG=C.UTF-8
ENV MIX_ENV=prod

# Prepare mounts
VOLUME /custom.d /uploads

# Expose HTTP, Gopher, and SSH ports, respectively
EXPOSE 4000 9999 2222

# Get dependencies
RUN \
    apk add --no-cache --virtual .tools \
        git curl rsync postgresql-client \
    && \
    apk add --no-cache --virtual .sdk \
        build-base cmake file-dev \
    && \
    apk add --no-cache --virtual .runtime \
        imagemagick ffmpeg exiftool \
        elixir erlang erlang-dev

# Add entrypoint
COPY ./entrypoint.sh /
RUN chmod a+x /entrypoint.sh
ENTRYPOINT ["/entrypoint.sh"]

# Limit permissions
RUN \
    # echo "#> akkoma user will be 1000:1000" 1>&2 && \
    addgroup -g 1000 akkoma && \
    adduser -S -s /bin/ash -G akkoma -u 1000 akkoma && \
    mkdir -p /custom.d /uploads && \
    chown -R akkoma:akkoma /custom.d /uploads

USER akkoma
WORKDIR /home/akkoma

# Get akkoma sources
#
# Also make sure that instance/static/emoji exists.
# akkoma does not ship with an `instance` folder by default, causing docker to create `instance/static` for us at launch.
# In itself that wouldn't be much of an issue, but said folder(s) are created as root:akkoma with 2755.
# This breaks the custom.d step in entrypoint.sh which relies on writing there (See #10).
#
ARG AKKOMA_VERSION=v3.10.3
RUN \
    # echo "#> Getting akkoma sources from $AKKOMA_GIT_REPO..." 1>&2 && \
    git clone --depth 1 --branch $AKKOMA_VERSION https://akkoma.dev/AkkomaGang/akkoma.git ./akkoma && \
    mkdir -p ./akkoma/instance/static/emoji

WORKDIR /home/akkoma/akkoma

# Precompile
RUN \
    cp ./config/dev.exs ./config/prod.secret.exs && \
    BUILDTIME=1 /entrypoint.sh

# Install default frontends
RUN \
    mix pleroma.frontend install pleroma-fe && \
    mix pleroma.frontend install admin-fe && \
    rm ./config/prod.secret.exs

# Register healthcheck
# You might need to change these values on slow or busy servers.
HEALTHCHECK \
    --interval=10s \
    --start-period=50s \
    --timeout=4s \
    --retries=3 \
    CMD curl -sSLf http://localhost:4000/api/pleroma/healthcheck || exit 1
