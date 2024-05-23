ARG PROFILE=release
ARG BUILD_DATE
ARG GIT_REVISION
# ARG RUST_VERSION=1.76.0

FROM stagex/busybox:sx2024.05.0@sha256:8cb9360041cd17e8df33c5cbc6c223875045c0c249254367ed7e0eb445720757 AS busybox
FROM stagex/musl:sx2024.05.0@sha256:f888fcf45fabaaae3d0268bcec902ceb94edba7bf8d09ef6966ebb20e00b7127 AS musl
FROM stagex/rust:sx2024.05.0@sha256:b982614e41a163f0b4222a7472030e30d466a8a605a1ff41e9731e9921e90f0e AS rust
FROM stagex/gcc:sx2024.05.0@sha256:ce77c0d1576d3c9d7905edd438fc58d2c51c0340a996c09a93c4602534dc2e26 AS gcc
FROM stagex/llvm:sx2024.05.0@sha256:c43abe585e2867d2ef2fad0902f2a264a3c4a429f95dddd00320de59dcf66210 AS llvm
FROM stagex/libunwind:sx2024.05.0@sha256:18d3b25f523d83aec9db229528d83068a5e289cc6dd27c85ab6ed0f0a55bc9a9 AS libunwind
FROM stagex/openssl:sx2024.05.0@sha256:9bd55ed05263a538e6a23c0262edc356c998a24674f3b8ad008a4b117a4cdf3b AS openssl
FROM stagex/zlib:sx2024.05.0@sha256:d0d6eef463a410191e086448c710441109ae72693cb074fe2b795ee033aa6c9d AS zlib
FROM stagex/ca-certificates:sx2024.05.0@sha256:76b232139c838fad3cdc693f839384141c2a5bf6e7f390294a133be1392a9b7a AS ca-certificates

FROM stagex/binutils:sx2024.05.0@sha256:823ad20a58696435f4afd61aadbe7d9e18afde676a94b59a932126fc16ba0761 AS binutils
FROM stagex/make:sx2024.05.0@sha256:8357ff7a8afa260ae3cc8e8993d80bce524d9802b2033020f7ea7f8f85133634 AS make
FROM stagex/clang:sx2024.05.0@sha256:0021ac32c35197d8bba0ae6a27104da7dd7c63535b0f3e9bfe812e55b9e97b9d AS clang
FROM stagex/linux-headers:sx2024.05.0@sha256:fe366787ecaf36393b17ede6108161af4136bf5b7521e49f0a005a6ef68ef8db AS linux-headers

FROM scratch AS base

FROM base AS fetch

COPY --from=busybox . /
COPY --from=musl . /
COPY --from=rust . /

COPY --from=gcc . /
COPY --from=llvm . /
COPY --from=libunwind . /
COPY --from=openssl . /
COPY --from=zlib . /

# NOTE: Necessary for `cargo fetch`, but CA trust is not relied upon
COPY --from=ca-certificates . /

COPY . /sui

WORKDIR sui

RUN cargo fetch

FROM fetch AS build

# Rust build deps

COPY --from=binutils . /
COPY --from=gcc . /
COPY --from=llvm . /
COPY --from=make . /
COPY --from=musl . /

# Sui build deps

COPY --from=clang . /
COPY --from=linux-headers . /

ARG PROFILE
ARG GIT_REVISION

ENV RUST_BACKTRACE=1
ENV RUSTFLAGS='-C target-feature=-crt-static -C codegen-units=1'
ENV GIT_REVISION=${GIT_REVISION}
ENV PROFILE=${PROFILE}

RUN --network=none cargo build --frozen --profile ${PROFILE} --bin sui-node

FROM scratch AS install

COPY --from=busybox . /

COPY --from=busybox . /rootfs
COPY --from=libunwind . /rootfs
COPY --from=gcc . /rootfs
COPY --from=musl . /rootfs

# support current + legacy paths
RUN mkdir -p /rootfs/opt/sui/bin
RUN mkdir -p /rootfs/usr/local/bin
COPY --from=build sui/target/release/sui-node /rootfs/opt/sui/bin/sui-node


RUN --network=none find /rootfs -exec touch -hcd "@0" "{}" +

FROM scratch AS package

ARG PROFILE
ARG GIT_REVISION

LABEL build-date=${BUILD_DATE}
LABEL git-revision=${GIT_REVISION}

COPY --from=install /rootfs /

RUN ln -s /opt/sui/bin/sui-node /usr/local/bin/sui-node

