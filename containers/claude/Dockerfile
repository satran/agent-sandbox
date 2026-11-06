FROM sandbox-base

# Install leaf for better markdown viewing
RUN curl -fsSL https://leaf.rivolink.mg/install.sh | sh
RUN curl -fsSL https://claude.ai/install.sh | bash

# Installs the plugins listed in it on first start, then runs claude.
COPY --chmod=755 claude-start /usr/local/bin/claude-start
