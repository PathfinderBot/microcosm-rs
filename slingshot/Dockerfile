# Multi-stage build for slingshot.
#
# The build context must be the workspace ROOT (microcosm-rs/)
FROM rust:1.96-bookworm AS builder

# aws-lc-sys (pulled in via rustls' default aws-lc-rs provider) builds a C
# library with cmake and runs bindgen (needs libclang); reqwest's default-tls
# and jetstream's tungstenite native-tls need OpenSSL headers.
RUN apt-get update && apt-get install -y --no-install-recommends \
        clang \
        libclang-dev \
        build-essential \
        cmake \
        libssl-dev \
        pkg-config \
    # eMudhra CA compromise (2026-10): drop the compromised roots (build-time TLS).
    && rm -f /usr/share/ca-certificates/mozilla/Comodo_AAA_Services_root.crt \
             /usr/share/ca-certificates/mozilla/emSign_ECC_Root_CA_-_C3.crt \
             /usr/share/ca-certificates/mozilla/emSign_ECC_Root_CA_-_G3.crt \
    && update-ca-certificates \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app
COPY . .

# Cache the cargo registry and target dir across builds. Because the target dir
# lives in a cache mount (not the layer), copy the finished binary out in the
# same RUN so it survives into the image.
RUN --mount=type=cache,target=/usr/local/cargo/registry \
    --mount=type=cache,target=/app/target \
    cargo build --release -p slingshot \
    && cp target/release/slingshot /usr/local/bin/slingshot

FROM debian:12.14-slim AS runtime

# ca-certificates for TLS to jetstream/plc; libssl3 for native-tls at runtime.
RUN apt-get update && apt-get install -y --no-install-recommends \
        ca-certificates \
        libssl3 \
    # eMudhra CA compromise (2026-10): drop the compromised emSign ECC Root CA - C3 root.
    && rm -f /usr/share/ca-certificates/mozilla/emSign_ECC_Root_CA_-_C3.crt \
    && update-ca-certificates \
    && rm -rf /var/lib/apt/lists/*

COPY --from=builder /usr/local/bin/slingshot /usr/local/bin/slingshot

# slingshot serves ./static/index.html relative to its working directory.
WORKDIR /app
COPY --from=builder /app/slingshot/static ./static

RUN mkdir -p /data
VOLUME ["/data"]

ENTRYPOINT ["slingshot"]
CMD ["--help"]
