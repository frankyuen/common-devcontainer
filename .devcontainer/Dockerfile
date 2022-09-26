# If you prefer to use Microsoft's dev container images directly, see:
# - https://github.com/microsoft/vscode-dev-containers/tree/main/containers
# - https://mcr.microsoft.com/v2/vscode/devcontainers/python/tags/list (tag list)
# - https://mcr.microsoft.com/v2/vscode/devcontainers/go/tags/list (tag list)

FROM buildpack-deps:bullseye

# *** Install Python: https://github.com/docker-library/python/

# ensure local python is preferred over distribution python
ENV PATH /usr/local/bin:$PATH

# http://bugs.python.org/issue19846
# > At the moment, setting "LANG=C" on a Linux system *fundamentally breaks Python 3*, and that's not OK.
ENV LANG C.UTF-8

# runtime dependencies
RUN set -eux; \
    apt-get update; \
    apt-get install -y --no-install-recommends \
        libbluetooth-dev \
        tk-dev \
        uuid-dev \
    ; \
    rm -rf /var/lib/apt/lists/*

ENV GPG_KEY A035C8C19219BA821ECEA86B64E628F8D684696D
ENV PYTHON_VERSION 3.10.7

RUN set -eux; \
    \
    wget -O python.tar.xz "https://www.python.org/ftp/python/${PYTHON_VERSION%%[a-z]*}/Python-$PYTHON_VERSION.tar.xz"; \
    wget -O python.tar.xz.asc "https://www.python.org/ftp/python/${PYTHON_VERSION%%[a-z]*}/Python-$PYTHON_VERSION.tar.xz.asc"; \
    GNUPGHOME="$(mktemp -d)"; export GNUPGHOME; \
    gpg --batch --keyserver hkps://keys.openpgp.org --recv-keys "$GPG_KEY"; \
    gpg --batch --verify python.tar.xz.asc python.tar.xz; \
    command -v gpgconf > /dev/null && gpgconf --kill all || :; \
    rm -rf "$GNUPGHOME" python.tar.xz.asc; \
    mkdir -p /usr/src/python; \
    tar --extract --directory /usr/src/python --strip-components=1 --file python.tar.xz; \
    rm python.tar.xz; \
    \
    cd /usr/src/python; \
    gnuArch="$(dpkg-architecture --query DEB_BUILD_GNU_TYPE)"; \
    ./configure \
        --build="$gnuArch" \
        --enable-loadable-sqlite-extensions \
        --enable-optimizations \
        --enable-option-checking=fatal \
        --enable-shared \
        --with-lto \
        --with-system-expat \
        --without-ensurepip \
    ; \
    nproc="$(nproc)"; \
    make -j "$nproc" \
    ; \
    make install; \
    \
# enable GDB to load debugging data: https://github.com/docker-library/python/pull/701
    bin="$(readlink -ve /usr/local/bin/python3)"; \
    dir="$(dirname "$bin")"; \
    mkdir -p "/usr/share/gdb/auto-load/$dir"; \
    cp -vL Tools/gdb/libpython.py "/usr/share/gdb/auto-load/$bin-gdb.py"; \
    \
    cd /; \
    rm -rf /usr/src/python; \
    \
    find /usr/local -depth \
        \( \
        \( -type d -a \( -name test -o -name tests -o -name idle_test \) \) \
        -o \( -type f -a \( -name '*.pyc' -o -name '*.pyo' -o -name 'libpython*.a' \) \) \
        \) -exec rm -rf '{}' + \
    ; \
    \
    ldconfig; \
    \
    python3 --version

# make some useful symlinks that are expected to exist ("/usr/local/bin/python" and friends)
RUN set -eux; \
    for src in idle3 pydoc3 python3 python3-config; do \
        dst="$(echo "$src" | tr -d 3)"; \
        [ -s "/usr/local/bin/$src" ]; \
        [ ! -e "/usr/local/bin/$dst" ]; \
        ln -svT "$src" "/usr/local/bin/$dst"; \
    done

# if this is called "PIP_VERSION", pip explodes with "ValueError: invalid truth value '<VERSION>'"
ENV PYTHON_PIP_VERSION 22.2.2
# https://github.com/docker-library/python/issues/365
ENV PYTHON_SETUPTOOLS_VERSION 63.2.0
# https://github.com/pypa/get-pip
ENV PYTHON_GET_PIP_URL https://github.com/pypa/get-pip/raw/5eaac1050023df1f5c98b173b248c260023f2278/public/get-pip.py
ENV PYTHON_GET_PIP_SHA256 5aefe6ade911d997af080b315ebcb7f882212d070465df544e1175ac2be519b4

RUN set -eux; \
    \
    wget -O get-pip.py "$PYTHON_GET_PIP_URL"; \
    echo "$PYTHON_GET_PIP_SHA256 *get-pip.py" | sha256sum -c -; \
    \
    export PYTHONDONTWRITEBYTECODE=1; \
    \
    python get-pip.py \
        --disable-pip-version-check \
        --no-cache-dir \
        --no-compile \
        "pip==$PYTHON_PIP_VERSION" \
        "setuptools==$PYTHON_SETUPTOOLS_VERSION" \
    ; \
    rm -f get-pip.py

# *** Install Go: https://github.com/docker-library/golang

# install cgo-related dependencies
RUN set -eux; \
    apt-get update; \
    apt-get install -y --no-install-recommends \
        g++ \
        gcc \
        libc6-dev \
        make \
        pkg-config \
    ; \
    rm -rf /var/lib/apt/lists/*

ENV PATH /usr/local/go/bin:$PATH

ENV GOLANG_VERSION 1.18.6

RUN set -eux; \
    arch="$(dpkg --print-architecture)"; arch="${arch##*-}"; \
    url=; \
    case "$arch" in \
        'amd64') \
            url='https://dl.google.com/go/go1.18.6.linux-amd64.tar.gz'; \
            sha256='bb05f179a773fed60c6a454a24141aaa7e71edfd0f2d465ad610a3b8f1dc7fe8'; \
            ;; \
        'armel') \
            export GOARCH='arm' GOARM='5' GOOS='linux'; \
            ;; \
        'armhf') \
            url='https://dl.google.com/go/go1.18.6.linux-armv6l.tar.gz'; \
            sha256='fca2a46dfdab541f63afaa04029c0d75e934e05464bf8c4f636c9d9856dfdaf2'; \
            ;; \
        'arm64') \
            url='https://dl.google.com/go/go1.18.6.linux-arm64.tar.gz'; \
            sha256='838ffa94158125f16e4aa667ee4f6b499ea57e3e35a7e2517ad357ea06714691'; \
            ;; \
        'i386') \
            url='https://dl.google.com/go/go1.18.6.linux-386.tar.gz'; \
            sha256='52a8ec92dc1c192fbbfc3c9583ded388edf14ee8b687b034e3c1034024bf2970'; \
            ;; \
        'mips64el') \
            export GOARCH='mips64le' GOOS='linux'; \
            ;; \
        'ppc64el') \
            url='https://dl.google.com/go/go1.18.6.linux-ppc64le.tar.gz'; \
            sha256='bcec49f08bb67ae2821ece8fecbc2ba678d54ce6f8cfaa572b86448aa09ca816'; \
            ;; \
        's390x') \
            url='https://dl.google.com/go/go1.18.6.linux-s390x.tar.gz'; \
            sha256='661af75e03cb8effcf90705dc0a12875efdb653ad8a0ca434905ef665189350a'; \
            ;; \
        *) echo >&2 "error: unsupported architecture '$arch' (likely packaging update needed)"; exit 1 ;; \
    esac; \
    build=; \
    if [ -z "$url" ]; then \
# https://github.com/golang/go/issues/38536#issuecomment-616897960
        build=1; \
        url='https://dl.google.com/go/go1.18.6.src.tar.gz'; \
        sha256='a7f1d50424355dabce66d1112b1cae439b6ee5e4f15edba6f104c0a4b173e895'; \
        echo >&2; \
        echo >&2 "warning: current architecture ($arch) does not have a compatible Go binary release; will be building from source"; \
        echo >&2; \
    fi; \
    \
    wget -O go.tgz.asc "$url.asc"; \
    wget -O go.tgz "$url" --progress=dot:giga; \
    echo "$sha256 *go.tgz" | sha256sum -c -; \
    \
# https://github.com/golang/go/issues/14739#issuecomment-324767697
    GNUPGHOME="$(mktemp -d)"; export GNUPGHOME; \
# https://www.google.com/linuxrepositories/
    gpg --batch --keyserver keyserver.ubuntu.com --recv-keys 'EB4C 1BFD 4F04 2F6D DDCC  EC91 7721 F63B D38B 4796'; \
# let's also fetch the specific subkey of that key explicitly that we expect "go.tgz.asc" to be signed by, just to make sure we definitely have it
    gpg --batch --keyserver keyserver.ubuntu.com --recv-keys '2F52 8D36 D67B 69ED F998  D857 78BD 6547 3CB3 BD13'; \
    gpg --batch --verify go.tgz.asc go.tgz; \
    gpgconf --kill all; \
    rm -rf "$GNUPGHOME" go.tgz.asc; \
    \
    tar -C /usr/local -xzf go.tgz; \
    rm go.tgz; \
    \
    if [ -n "$build" ]; then \
        savedAptMark="$(apt-mark showmanual)"; \
        apt-get update; \
        apt-get install -y --no-install-recommends golang-go; \
        \
        export GOCACHE='/tmp/gocache'; \
        \
        ( \
            cd /usr/local/go/src; \
# set GOROOT_BOOTSTRAP + GOHOST* such that we can build Go successfully
            export GOROOT_BOOTSTRAP="$(go env GOROOT)" GOHOSTOS="$GOOS" GOHOSTARCH="$GOARCH"; \
            ./make.bash; \
        ); \
        \
        apt-mark auto '.*' > /dev/null; \
        apt-mark manual $savedAptMark > /dev/null; \
        apt-get purge -y --auto-remove -o APT::AutoRemove::RecommendsImportant=false; \
        rm -rf /var/lib/apt/lists/*; \
        \
# remove a few intermediate / bootstrapping files the official binary release tarballs do not contain
        rm -rf \
            /usr/local/go/pkg/*/cmd \
            /usr/local/go/pkg/bootstrap \
            /usr/local/go/pkg/obj \
            /usr/local/go/pkg/tool/*/api \
            /usr/local/go/pkg/tool/*/go_bootstrap \
            /usr/local/go/src/cmd/dist/dist \
            "$GOCACHE" \
        ; \
    fi; \
    \
    go version

ENV GOPATH /go
ENV PATH $GOPATH/bin:$PATH
RUN mkdir -p "$GOPATH/src" "$GOPATH/bin" && chmod -R 777 "$GOPATH"

# *** Install libary-scripts (need to manually download them first) for vscode dev container:
# *** https://github.com/microsoft/vscode-dev-containers/tree/main/containers/python-3/.devcontainer/library-scripts
# *** https://github.com/microsoft/vscode-dev-containers/tree/main/containers/go/.devcontainer/library-scripts

# Copy library scripts to execute
COPY library-scripts/*.sh library-scripts/*.env /tmp/library-scripts/

# [Option] Install zsh
ARG INSTALL_ZSH="true"
# [Option] Upgrade OS packages to their latest versions
ARG UPGRADE_PACKAGES="true"
# Install needed packages and setup non-root user. Use a separate RUN statement to add your own dependencies.
ARG USERNAME=vscode
ARG USER_UID=1000
ARG USER_GID=$USER_UID
RUN apt-get update && export DEBIAN_FRONTEND=noninteractive \
    # Remove imagemagick due to https://security-tracker.debian.org/tracker/CVE-2019-10131
    && apt-get purge -y imagemagick imagemagick-6-common \
    # Install common packages, non-root user
    && bash /tmp/library-scripts/common-debian.sh "${INSTALL_ZSH}" "${USERNAME}" "${USER_UID}" "${USER_GID}" "${UPGRADE_PACKAGES}" "true" "true" \
    && apt-get autoremove -y && apt-get clean -y && rm -rf /var/lib/apt/lists/*

# Setup default python tools in a venv via pipx to avoid conflicts
ENV PIPX_HOME=/usr/local/py-utils \
    PIPX_BIN_DIR=/usr/local/py-utils/bin
ENV PATH=${PATH}:${PIPX_BIN_DIR}
RUN bash /tmp/library-scripts/python-debian.sh "none" "/usr/local" "${PIPX_HOME}" "${USERNAME}" \ 
    && apt-get clean -y && rm -rf /var/lib/apt/lists/*

# Install Go tools
ENV GO111MODULE=auto
RUN bash /tmp/library-scripts/go-debian.sh "none" "/usr/local/go" "${GOPATH}" "${USERNAME}" "false" \
    && apt-get clean -y && rm -rf /var/lib/apt/lists/*

# [Choice] Node.js version: none, lts/*, 16, 14, 12, 10
ARG NODE_VERSION="lts/*"
ENV NVM_DIR=/usr/local/share/nvm
ENV NVM_SYMLINK_CURRENT=true \
    PATH=${NVM_DIR}/current/bin:${PATH}
RUN bash /tmp/library-scripts/node-debian.sh "${NVM_DIR}" "${NODE_VERSION}" "${USERNAME}" \
    && apt-get clean -y && rm -rf /var/lib/apt/lists/*

# Remove library scripts for final image
RUN rm -rf /tmp/library-scripts

# [Optional] If your pip requirements rarely change, uncomment this section to add them to the image.
# COPY requirements.txt /tmp/pip-tmp/
# RUN pip3 --disable-pip-version-check --no-cache-dir install -r /tmp/pip-tmp/requirements.txt \
#    && rm -rf /tmp/pip-tmp

# [Optional] Uncomment the next line to use go get to install anything else you need
# RUN go get -x <your-dependency-or-tool>

# [Optional] Uncomment this section to install additional OS packages.
# RUN apt-get update && export DEBIAN_FRONTEND=noninteractive \
#     && apt-get -y install --no-install-recommends <your-package-list-here>

# [Optional] Uncomment this line to install global node packages.
# RUN su vscode -c "source /usr/local/share/nvm/nvm.sh && npm install -g <your-package-here>" 2>&1

# Set some common alias
COPY --chown=${USER_UID}:${USER_GID} scripts/* /home/${USERNAME}/

# To keep the container running in detached mode
CMD tail -f /dev/null
