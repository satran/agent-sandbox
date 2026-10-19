FROM ubuntu:26.04

# Avoid interactive prompts during build
ENV DEBIAN_FRONTEND=noninteractive

RUN apt-get update && apt-get install -y \
	curl \
	vim \
	emacs-nox \
	git \
	python3 \
	python3-pip \
	python3-venv \
	nodejs \
	npm

RUN npm install -g @google/gemini-cli

# Set up a symbolic link for python (so 'python' works instead of 'python3')
RUN ln -s /usr/bin/python3 /usr/bin/python

# Setup local user
RUN apt-get install -y sudo
RUN useradd -ms /bin/bash satran
RUN usermod -G sudo satran
RUN echo '%sudo ALL=(ALL) NOPASSWD:ALL' >> /etc/sudoers

WORKDIR /home/satran
USER satran

CMD ["/bin/bash"]
