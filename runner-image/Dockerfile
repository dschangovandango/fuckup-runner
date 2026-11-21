# Prebuilt audit toolchain image.
#
# This image contains the toolchains, not the FUCKUP source. The audit
# workflow mounts the per-job runner bundle at /opt/runner and sets
# PYTHONPATH=/opt/runner, so pipeline changes do not require an image rebuild.
FROM ubuntu:24.04

ENV DEBIAN_FRONTEND=noninteractive
ENV PYTHONUNBUFFERED=1
ENV PIP_DISABLE_PIP_VERSION_CHECK=1
ENV PATH=/root/.foundry/bin:/root/.local/bin:/usr/local/go/bin:/usr/local/bin:$PATH

RUN apt-get update && apt-get install -y --no-install-recommends \
    ca-certificates curl wget git jq unzip xz-utils tar gnupg lsb-release \
    build-essential cmake ninja-build pkg-config clang llvm libclang-dev \
    libssl-dev libffi-dev libgmp-dev libxml2-dev libxslt1-dev libpq-dev \
    python3 python3-pip python3-venv python3-dev \
    golang-go \
    nmap openssl dnsutils netcat-openbsd socat lsof procps \
    ripgrep fd-find vim less \
    && rm -rf /var/lib/apt/lists/*

# Node 24 + pnpm/yarn. Ubuntu's nodejs is too old for several Solidity/JS repos.
RUN curl -fsSL https://deb.nodesource.com/setup_24.x | bash - && \
    apt-get install -y --no-install-recommends nodejs && \
    corepack enable && \
    corepack prepare pnpm@latest --activate && \
    npm install -g yarn && \
    node --version && npm --version && pnpm --version

# Rust stable (cargo, rustc, rustfmt, clippy).
RUN curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs -o /tmp/rustup.sh && \
    sh /tmp/rustup.sh -y --default-toolchain stable --profile default && \
    /root/.cargo/bin/rustc --version && /root/.cargo/bin/cargo --version

# Foundry (forge, cast, anvil, chisel).
RUN curl -fsSL https://foundry.paradigm.xyz | bash && \
    /root/.foundry/bin/foundryup && \
    /root/.foundry/bin/forge --version

# Sui / Move.
RUN curl -fsSL https://raw.githubusercontent.com/MystenLabs/suiup/main/install.sh | bash && \
    for i in 1 2 3; do \
        /root/.local/bin/suiup install sui@1.84.0 && break; \
        echo "suiup install attempt $i failed; retrying" >&2; \
        sleep 10; \
    done && \
    /root/.local/bin/sui --version

# Python audit stack: runner client, EVM tooling, fuzzers, Vyper, Halmos.
RUN python3 -m pip install --break-system-packages --no-cache-dir \
    openai requests python-json-logger httpx \
    web3 eth-utils eth-abi eth-account eth-typing \
    py-solc-x solc-select slither-analyzer crytic-compile \
    vyper halmos \
    beautifulsoup4 lxml html5lib numpy scipy \
    opentelemetry-api opentelemetry-sdk \
    opentelemetry-exporter-otlp-proto-grpc && \
    solc-select install 0.8.28 && solc-select use 0.8.28 && \
    solc --version && slither --version

# EVM fuzzers used by the audit toolset.
RUN mkdir -p /opt/fuzz /usr/local/bin && \
    curl -fsSL -o /tmp/echidna.zip \
      https://github.com/crytic/echidna/releases/latest/download/echidna-x86_64-linux.zip && \
    unzip -o /tmp/echidna.zip echidna -d /opt/fuzz >/dev/null && \
    chmod +x /opt/fuzz/echidna && ln -sf /opt/fuzz/echidna /usr/local/bin/echidna && \
    curl -fsSL -o /tmp/medusa.tar.gz \
      https://github.com/crytic/medusa/releases/latest/download/medusa-linux-x86_64.tar.gz && \
    tar -xzf /tmp/medusa.tar.gz -C /opt/fuzz && \
    chmod +x /opt/fuzz/medusa && ln -sf /opt/fuzz/medusa /usr/local/bin/medusa && \
    echidna --version && medusa --version

# Cairo / Starknet: Scarb, Starknet Foundry, Clarinet.
RUN curl --proto '=https' --tlsv1.2 -sSf \
      https://docs.swmansion.com/scarb/install.sh | sh && \
    curl -sL https://raw.githubusercontent.com/foundry-rs/starknet-foundry/master/scripts/install.sh | sh && \
    curl -fsSL -o /tmp/clarinet.tar.gz \
      https://github.com/hirosystems/clarinet/releases/latest/download/clarinet-linux-x64.tar.gz && \
    tar -xzf /tmp/clarinet.tar.gz -C /usr/local/bin && \
    chmod +x /usr/local/bin/clarinet && clarinet --version

# Aptos CLI (latest release asset).
RUN set -eux; \
    url="$(curl -fsSL https://api.github.com/repos/aptos-labs/aptos-core/releases/latest \
      | jq -r '[.assets[] | select(.name | test("aptos-cli.*(Ubuntu|ubuntu).*x86_64.*zip$"))][0].browser_download_url')"; \
    test -n "$url"; \
    curl -fsSL "$url" -o /tmp/aptos.zip; \
    unzip -q /tmp/aptos.zip -d /usr/local/bin; \
    chmod +x /usr/local/bin/aptos; \
    /usr/local/bin/aptos --version

# Iota CLI (latest release asset).
RUN set -eux; \
    url="$(curl -fsSL https://api.github.com/repos/iotaledger/iota/releases/latest \
      | jq -r '[.assets[] | select(.name | test(".*(ubuntu|linux).*x86_64.*tar.gz$"))][0].browser_download_url')"; \
    test -n "$url"; \
    curl -fsSL "$url" -o /tmp/iota.tar.gz; \
    tar -xzf /tmp/iota.tar.gz -C /usr/local/bin --strip-components=1; \
    chmod +x /usr/local/bin/iota; \
    /usr/local/bin/iota --version

# Cosmos: gaiad + wasmd (latest release assets).
RUN set -eux; \
    url="$(curl -fsSL https://api.github.com/repos/cosmos/gaia/releases/latest \
      | jq -r '[.assets[] | select(.name | test("gaiad.*linux-amd64$"))][0].browser_download_url')"; \
    test -n "$url"; \
    curl -fsSL "$url" -o /usr/local/bin/gaiad; \
    chmod +x /usr/local/bin/gaiad; \
    url="$(curl -fsSL https://api.github.com/repos/CosmWasm/wasmd/releases/latest \
      | jq -r '[.assets[] | select(.name | test("wasmd.*(Linux|linux).*x86_64.*tar.gz$"))][0].browser_download_url')"; \
    test -n "$url"; \
    curl -fsSL "$url" -o /tmp/wasmd.tar.gz; \
    tar -xzf /tmp/wasmd.tar.gz -C /usr/local/bin wasmd; \
    chmod +x /usr/local/bin/wasmd; \
    gaiad version --long | head -1; wasmd version

# Solana / Agave toolchain (solana, cargo-build-sbf, solana-test-validator).
RUN sh -c "$(curl -sSfL https://release.anza.xyz/stable/install)" && \
    /root/.local/share/solana/install/active_release/bin/solana --version

# Staging directories used by the runner entrypoint.
RUN mkdir -p /workspace /tests/run /opt/runner

ENV PATH=/root/.local/share/solana/install/active_release/bin:$PATH
WORKDIR /workspace
ENTRYPOINT ["python3", "-m", "pipeline.runner.gh_entrypoint"]
