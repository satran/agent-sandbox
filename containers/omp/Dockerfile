FROM ubuntu:26.04

# Avoid interactive prompts during build
ENV DEBIAN_FRONTEND=noninteractive

RUN apt-get update && apt-get install -y \
	curl \
	vim \
	fish \
	git

# Setup local user
RUN apt-get install -y sudo
RUN useradd -ms /bin/bash satran
RUN usermod -G sudo satran
RUN echo '%sudo ALL=(ALL) NOPASSWD:ALL' >> /etc/sudoers

WORKDIR /home/satran
USER satran

RUN curl -fsSL https://omp.sh/install | sh
RUN echo 'export PATH="$HOME/.local/bin:$PATH"' >> /home/satran/.bashrc

CMD ["/bin/bash"]

