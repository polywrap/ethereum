FROM rust:1.65.0 as base

# Install the wasm32 rust build target
RUN rustup target add wasm32-unknown-unknown

WORKDIR /build-deps

# Install curl
RUN apt-get update
RUN apt-get -y install curl clang llvm build-essential

# Install wasm-opt
RUN curl -L https://github.com/WebAssembly/binaryen/releases/download/version_101/binaryen-version_101-x86_64-linux.tar.gz | tar -xz \
    && chmod +x binaryen-version_101/bin/wasm-opt \
    && cp binaryen-version_101/bin/wasm-opt /usr/local/bin/ \
    && rm -rf binary-version_101

# Install the toml-cli
RUN cargo install -f toml-cli

# Install wasm-snip
RUN cargo install -f wasm-snip

# Install wasm-bindgen
RUN cargo install -f wasm-bindgen-cli

# Install cargo-build-deps
RUN cargo install -f cargo-build-deps



WORKDIR /
RUN USER=root cargo new --lib project
WORKDIR /project

# Copy all manifest files
COPY polywrap.yaml .
COPY polywrap.deploy.yaml .

# Copy Cargo manifest files
COPY .//Cargo.toml .//Cargo.toml


# Mock build for dependencies:

# Ensure the module at . has the crate-type = ["cdylib"]
RUN toml set ././Cargo.toml lib.crate-type ["cdylib"] > ././Cargo-local.toml && \
    rm -rf ././Cargo.toml && \
    mv ././Cargo-local.toml ././Cargo.toml

# Clean up artifacts left by the toml CLI program ("["cdylib"]" -> ["cdylib"])
RUN sed -i 's/"\[cdylib\]"/\["cdylib"\]/g' ././Cargo.toml

# Ensure the package name = "module"
RUN toml set ././Cargo.toml package.name "module" > ././Cargo-local.toml && \
    rm -rf ././Cargo.toml && \
    mv ././Cargo-local.toml ././Cargo.toml

# Prebuild all project dependencies, adding them to the cache
RUN cd ./. && cargo build-deps --release && cd /project

# Copy all source files
RUN mv ././Cargo.toml ././Cargo-deps.toml
COPY . .
RUN mv ././Cargo-deps.toml ././Cargo.toml

# Actual build:

# Ensure the Wasm module is configured to use imported memory
ENV RUSTFLAGS="-C link-arg=-z -C link-arg=stack-size=65536 -C link-arg=--import-memory"

# Build the module at .
RUN cargo build --manifest-path ././Cargo.toml \
    --target wasm32-unknown-unknown --release

# Make the build directory
RUN rm -rf ./build
RUN mkdir ./build

# Enable the "WASM_INTERFACE_TYPES" feature, which will remove the __wbindgen_throw import.
# See: https://github.com/rustwasm/wasm-bindgen/blob/7f4663b70bd492278bf0e7bba4eeddb3d840c868/crates/cli-support/src/lib.rs#L397-L403
ENV WASM_INTERFACE_TYPES=1

# Run wasm-bindgen over the module, replacing all placeholder __wbindgen_... imports
RUN wasm-bindgen ././target/wasm32-unknown-unknown/release/module.wasm --out-dir ./build --out-name bg_module.wasm

RUN wasm-snip ./build/bg_module.wasm -o ./build/snipped_module.wasm && \
    rm -rf ./build/bg_module.wasm

# Use wasm-opt to perform the "asyncify" post-processing step over all modules
RUN wasm-opt --asyncify -Os ./build/snipped_module.wasm -o ./build/wrap.wasm && \
    rm -rf ./build/snipped_module.wasm
