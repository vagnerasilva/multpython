FROM public.ecr.aws/sagemaker/sagemaker-distribution:2.3-cpu

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
    libffi-dev 

# Install ``pyenv``.
RUN sudo git clone https://github.com/pyenv/pyenv /root/.pyenv

# Install the desired versions of Python.
RUN for PYTHON_VERSION in 3.8.18 3.9.5 3.12.2 ; do \
  set -ex \
    && sudo /root/.pyenv/bin/pyenv install ${PYTHON_VERSION} \
    && sudo /root/.pyenv/versions/${PYTHON_VERSION}/bin/python -m pip install --upgrade pip \
  ; done

# Add to PATH, in order of lowest precedence to highest.
ENV sudo PATH /root/.pyenv/versions/3.8.18/bin:${PATH}
ENV sudo PATH /root/.pyenv/versions/3.9.5/bin:${PATH}
ENV sudo PATH /root/.pyenv/versions/3.12.2/bin:${PATH}

CMD ["python3.12.2"]