FROM ubuntu:22.04

ENV DEBIAN_FRONTEND noninteractive

# Install dependencies.
RUN sudo apt-get update \
  && sudo apt-get install -y --no-install-recommends \
    make \
    build-essential \
    libssl-dev \
    zlib1g-dev \
    libbz2-dev \
    libreadline-dev \
    libsqlite3-dev \
    wget \
    curl \
    llvm \
    libncurses5-dev \
    xz-utils \
    tk-dev \
    libxml2-dev \
    libxmlsec1-dev \
    git \
    ca-certificates \
    libffi-dev \
  && apt-get clean autoclean \
  && apt-get autoremove -y \
  && rm -rf /var/lib/apt/lists/* \
  && rm -f /var/cache/apt/archives/*.deb

# Install ``pyenv``.
RUN git clone https://github.com/pyenv/pyenv /root/.pyenv

# Install the desired versions of Python.
#RUN for PYTHON_VERSION in 3.8.18 3.9.18 3.10.13 3.11.8 3.12.2 pypy3.10-7.3.15; do \
#  set -ex \
#    && /root/.pyenv/bin/pyenv install ${PYTHON_VERSION} \
#    && /root/.pyenv/versions/${PYTHON_VERSION}/bin/python -m pip install --upgrade pip \
#  ; done

# Add to PATH, in order of lowest precedence to highest.
ENV PATH /root/.pyenv/versions/pypy3.10-7.3.15/bin:${PATH}
ENV PATH /root/.pyenv/versions/3.8.18/bin:${PATH}
ENV PATH /root/.pyenv/versions/3.9.18/bin:${PATH}
ENV PATH /root/.pyenv/versions/3.10.13/bin:${PATH}
ENV PATH /root/.pyenv/versions/3.11.8/bin:${PATH}
ENV PATH /root/.pyenv/versions/3.12.2/bin:${PATH}

#CMD ["python3.12"]