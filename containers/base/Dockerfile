FROM ubuntu:26.04

# Avoid interactive prompts during build
ENV DEBIAN_FRONTEND=noninteractive

# No sudo: the agent must not be able to become root in here. Add anything it
# needs system-wide to this list and rebuild with `sb <agent> --rebuild`.
RUN apt-get update && apt-get install -y \
	curl \
	vim \
	emacs-nox \
	tmux \
	fish \
	python3 \
	python3-pip \
	python3-venv \
	nodejs \
	npm \
	git \
	ranger \
	iptables \
	iproute2 \
	&& rm -rf /var/lib/apt/lists/*

# Set up a symbolic link for python (so 'python' works instead of 'python3')
RUN ln -s /usr/bin/python3 /usr/bin/python

# Setup local user
RUN useradd -ms /bin/bash agent

# Installs the egress firewall and idles; see the script. Runs as root, which
# is why the agent is attached as 'agent' instead of being this process.
COPY sandbox-init /usr/local/bin/sandbox-init
RUN chmod 755 /usr/local/bin/sandbox-init

# Drop setuid/setgid bits so there is nothing left for the agent to escalate
# through. Combined with --security-opt no-new-privileges this is belt and
# braces, but it also catches a container started without that flag.
RUN find / -xdev -type f -perm /6000 -exec chmod a-s {} + || true

ENV PATH="/home/agent/.local/bin:$PATH"

WORKDIR /home/agent
USER agent

CMD ["/bin/bash"]
