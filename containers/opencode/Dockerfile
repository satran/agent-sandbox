FROM sandbox-base

RUN curl -fsSL https://opencode.ai/v2/install | bash
# The installer puts the binary in ~/.opencode/bin, which isn't on PATH.
RUN mkdir -p ~/.local/bin && ln -s ~/.opencode/bin/opencode ~/.local/bin/opencode
