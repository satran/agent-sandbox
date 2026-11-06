FROM sandbox-base

# Installs the agy binary into ~/.local/bin.
RUN curl -fsSL https://antigravity.google/cli/install.sh | bash

# Starts a D-Bus session and keyring, then runs agy.
COPY --chmod=755 agy-start /usr/local/bin/agy-start
