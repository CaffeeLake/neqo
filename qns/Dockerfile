FROM lukemathwalker/cargo-chef@sha256:ca4454a0d7f0a78d1052295414692136d54986104eab9b2477d9df3cb466ed29 AS chef

WORKDIR /app

FROM chef AS planner

COPY . .
RUN cargo chef prepare --recipe-path recipe.json

FROM chef AS builder
RUN apt-get update && apt-get install -y --no-install-recommends \
    libclang-dev gyp ninja-build python-is-python3 \
    && apt-get autoremove -y && apt-get clean -y \
    && rm -rf /var/lib/apt/lists/*

# We unfortunately need to build NSS from source, because the Debian package is
# not compiled with support for SSLKEYLOGFILE.
# See https://bugs.debian.org/cgi-bin/bugreport.cgi?bug=842292
ENV NSS_DIR=/nss \
    NSS_TARGET=Release \
    NSS_PREBUILT=1 \
    NSPR_DIR=/nspr \
    LD_LIBRARY_PATH=/dist/Release/lib

RUN set -eux; \
    git clone --depth=1 https://github.com/nss-dev/nspr "$NSPR_DIR"; \
    git clone --depth=1 https://github.com/nss-dev/nss "$NSS_DIR"

RUN "$NSS_DIR"/build.sh --static -Ddisable_tests=1 -o

COPY --from=planner /app/recipe.json recipe.json
# Build dependencies
RUN cargo chef cook --release --recipe-path recipe.json
# Build application
ADD . /neqo
RUN set -eux; \
    cd /neqo; \
    CARGO_PROFILE_RELEASE_DEBUG=true cargo build --locked --release --bin neqo-client --bin neqo-server

# Copy only binaries to the final image to keep it small.

FROM martenseemann/quic-network-simulator-endpoint@sha256:3c373d0bac88ac0a005c56628e551fe980e44bf2dd50b9c6904d58d7f8d54089

ENV LD_LIBRARY_PATH=/neqo/lib
COPY --from=builder /neqo/target/release/neqo-client /neqo/target/release/neqo-server /neqo/bin/
COPY --from=builder /dist/Release/lib/*.so /neqo/lib/
COPY --from=builder /dist/Release/bin/certutil /dist/Release/bin/pk12util /neqo/bin/

COPY qns/interop.sh /neqo/
RUN chmod +x /neqo/interop.sh
ENTRYPOINT [ "/neqo/interop.sh" ]
