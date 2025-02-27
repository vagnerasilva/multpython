

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
#  && apt-get clean autoclean \
#  && apt-get autoremove -y \
#  && rm -rf /var/lib/apt/lists/* \
#  #&& rm -f /var/cache/apt/archives/*.deb

sudo /root/.pyenv/bin/pyenv


pyenv exec python3.8.10 test.py

sudo /root/.pyenv/bin/pyenv exec python3.12.2 test.py

pyenv exec python3.8.10 test.py

sudo /root/.pyenv/bin/pyenv exec  

sudo /root/.pyenv/bin/pyenv install --list

sudo /root/.pyenv/bin/pyenv install --list


sudo /root/.pyenv/bin/pyenv whence --path python3.12.2