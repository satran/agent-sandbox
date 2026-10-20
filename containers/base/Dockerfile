FROM ubuntu:26.04

# Avoid interactive prompts during build
ENV DEBIAN_FRONTEND=noninteractive

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
	sudo

# Set up a symbolic link for python (so 'python' works instead of 'python3')
RUN ln -s /usr/bin/python3 /usr/bin/python

# Setup local user
RUN useradd -ms /bin/bash agent
RUN usermod -G sudo agent
RUN echo '%sudo ALL=(ALL) NOPASSWD:ALL' >> /etc/sudoers

ENV PATH="/home/agent/.local/bin:$PATH"

WORKDIR /home/agent
USER agent

CMD ["/bin/bash"]
