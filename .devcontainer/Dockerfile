FROM public.ecr.aws/l8e3r2a5/rmx-ci-rcm-arm64

USER root

ARG TZ
ENV TZ="$TZ"

# Install basic development tools and iptables/ipset
RUN apt-get update && apt-get install -y --no-install-recommends \
  less \
  fzf \
  man-db \
  gnupg2 \
  gh \
  iptables \
  ipset \
  iproute2 \
  dnsutils \
  aggregate \
  vim \
  zsh \
  && apt-get clean && rm -rf /var/lib/apt/lists/*

# Python dependencies
RUN pip install cfn-lint

ARG USERNAME=circleci
ARG GROUP=circleci

# Ensure default user has access to /usr/local/share
RUN mkdir -p /usr/local/share/npm-global && \
  chown -R $USERNAME:$GROUP /usr/local/share

# Persist bash history.
RUN SNIPPET="export PROMPT_COMMAND='history -a' && export HISTFILE=/commandhistory/.bash_history" \
  && mkdir /commandhistory \
  && touch /commandhistory/.bash_history \
  && chown -R $USERNAME /commandhistory

# Set `DEVCONTAINER` environment variable to help with orientation
ENV DEVCONTAINER=true

# Create workspace and config directories and set permissions
RUN mkdir -p /workspace /home/$USERNAME/.claude && \
  chown -R $USERNAME:$GROUP /workspace /home/$USERNAME/.claude

WORKDIR /workspace

ARG GIT_DELTA_VERSION=0.18.2
RUN ARCH=$(dpkg --print-architecture) && \
  curl "https://github.com/dandavison/delta/releases/download/${GIT_DELTA_VERSION}/git-delta_${GIT_DELTA_VERSION}_${ARCH}.deb" -L -o "git-delta_${GIT_DELTA_VERSION}_${ARCH}.deb" && \
  sudo dpkg -i "git-delta_${GIT_DELTA_VERSION}_${ARCH}.deb" && \
  rm "git-delta_${GIT_DELTA_VERSION}_${ARCH}.deb"

USER $USERNAME

# Install global packages
ENV NPM_CONFIG_PREFIX=/usr/local/share/npm-global
ENV PATH=$PATH:/usr/local/share/npm-global/bin

# Set the default shell to bash rather than sh
ENV SHELL=/bin/bash

# Set the default editor and visual
ENV EDITOR=vim
ENV VISUAL=vim

# Default powerline10k theme
ARG ZSH_IN_DOCKER_VERSION=1.2.0
RUN sh -c "$(wget -O- https://github.com/deluan/zsh-in-docker/releases/download/v${ZSH_IN_DOCKER_VERSION}/zsh-in-docker.sh)" -- \
  -p git \
  -p fzf \
  -a "source /usr/share/doc/fzf/examples/key-bindings.zsh" \
  -a "source /usr/share/doc/fzf/examples/completion.zsh" \
  -a "export PROMPT_COMMAND='history -a' && export HISTFILE=/commandhistory/.bash_history" \
  -x

# AWS CLI
RUN curl -fsSL https://awscli.amazonaws.com/v2/install.sh | bash

# Install Claude
RUN curl -fsSL https://claude.ai/install.sh | bash
ENV PATH=$PATH:/home/${USERNAME}/.local/bin

# check that worked
RUN claude --version

# Copy and set up firewall script
COPY init-firewall.sh /usr/local/bin/
USER root
RUN chmod +x /usr/local/bin/init-firewall.sh && \
  echo "${USERNAME} ALL=(root) NOPASSWD: /usr/local/bin/init-firewall.sh" > /etc/sudoers.d/${USERNAME}-firewall && \
  chmod 0440 /etc/sudoers.d/${USERNAME}-firewall
USER $USERNAME

# Set up the same PATH for login shells as non-login
RUN echo "export PATH=${PATH}" >> ~/.bash_profile

RUN echo "alias ll='ls -lah'" >> ~/.bash_profile

