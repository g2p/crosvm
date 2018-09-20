FROM debian:stretch
LABEL description="Test crosvm using a command like the following: \
docker run --privileged -v /dev/log:/dev/log -v <path to crosvm>:/src:ro <crosvm base image>"

RUN apt-get update && apt-get install -y \
    autoconf \
    automake \
    curl \
    gcc \
    git \
    libcap-dev \
    libdrm-dev \
    libegl1-mesa-dev \
    libgl1-mesa-dev \
    libgles1-mesa-dev \
    libgles2-mesa-dev \
    libtool \
    libwayland-dev \
    make \
    nasm \
    ninja-build \
    pkg-config \
    protobuf-compiler \
    python3

ENV RUSTUP_HOME=/usr/local/rustup \
    CARGO_HOME=/usr/local/cargo \
    PATH=/usr/local/cargo/bin:$PATH \
    RUST_VERSION=1.29.0

# Debian usually has an old rust version in the repository. Instead of using that, we use rustup to
# pull in a toolchain versions of our choosing.
RUN curl -LO "https://static.rust-lang.org/rustup/archive/1.13.0/x86_64-unknown-linux-gnu/rustup-init" \
    && echo "f69dafcca62fe70d7882113e21bb96a2cbdf4fc4636d25337d6de9191bdec8da *rustup-init" | sha256sum -c - \
    && chmod +x rustup-init \
    && ./rustup-init -y --no-modify-path --default-toolchain $RUST_VERSION \
    && rm rustup-init \
    && chmod -R a+w $RUSTUP_HOME $CARGO_HOME \
    && rustup --version \
    && cargo --version \
    && rustc --version

# Warms up the cargo registry cache for future cargo runs. Cargo will still update the cache using a
# git pull, but it only needs to download files that were changed since this image was built.
RUN cargo install thisiznotarealpackage -q || true

# Used /scratch for building dependencies which are too new or don't exist on Debian stretch.
WORKDIR /scratch

# minijail does not exist in upstream linux distros.
RUN git clone https://android.googlesource.com/platform/external/minijail \
    && cd minijail \
    && make -j24 \
    && cp libminijail.so /usr/lib/x86_64-linux-gnu/

# The gbm used by upstream linux distros is not compatible with crosvm, which must use Chrome OS's
# minigbm.
RUN dpkg --force-depends -r libgbm1
RUN git clone https://chromium.googlesource.com/chromiumos/platform/minigbm \
    && cd minigbm \
    && sed 's/-Wall/-Wno-maybe-uninitialized/g' -i Makefile \
    && make install -j24

# New libepoxy requires newer meson than is in Debian stretch.
RUN git clone https://github.com/mesonbuild/meson \
    && cd meson \
    && git checkout 0a5ff338012a00f32c3aa9d8773835accc3e4e5b \
    && ln -s $PWD/meson.py /usr/bin/meson

# New libepoxy has EGL_KHR_DEBUG entry points needed by crosvm.
RUN git clone https://github.com/anholt/libepoxy.git \
    && cd libepoxy \
    && git checkout 707f50e680ab4f1861b1e54ca6e2907aaca56c12 \
    && mkdir build \
    && cd build \
    && meson \
    && ninja install

# virglrenderer is under heavy development on master and we want the very latest.
RUN git clone https://gitlab.freedesktop.org/virgl/virglrenderer.git \
    && cd virglrenderer \
    && ./autogen.sh \
    && make install -j24

# Reduces image size and prevents accidentally using /scratch files
RUN rm -r /scratch /usr/bin/meson

# The manual installation of shared objects requires an ld.so.cache refresh.
RUN ldconfig

# The /build directory is used so that the bind mounted /src volume does not get scribbled on.
ENV CARGO_TARGET_DIR=/build
RUN mkdir -p $CARGO_TARGET_DIR
WORKDIR /src
CMD cargo test --no-fail-fast --all-features --all --exclude aarch64 $TEST_FLAGS -- \
    --test-threads=1 $TEST_RUNNER_FLAGS