# Base and setup configuration
FROM ubuntu:24.04
ARG DEBIAN_FRONTEND=noninteractive
LABEL maintainer="Target Brands, Inc. TTS-CFC-OpenSource@target.com"

# User configuration
ARG CONFIG_TESTS=false

ARG JOHN_VERSION=1.9.1-ce
ARG YARA_VERSION=4.6.0
ARG YARA_PYTHON_VERSION=4.5.4
ARG EXIFTOOL_VERSION=12.60
ARG ARCHIVE_ZIP_VERSION=1.68
ARG CAPA_VERSION=10.0.0

# Environment variables
ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONFAULTHANDLER=1
ENV PYTHONUNBUFFERED=1

# User configuration
ARG USERNAME=strelka
ARG USER_UID=1001
ARG USER_GID=$USER_UID
RUN groupadd --gid $USER_GID $USERNAME && \
    useradd --uid $USER_UID --gid $USER_GID --create-home --shell /bin/bash $USERNAME

# Install build and runtime packages
RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,target=/var/lib/apt,sharing=locked \
    apt-get update -qq && \
    apt-get install -qq -y --no-install-recommends \
    antiword \
    autoconf \
    automake \
    binwalk \
    build-essential \
    ca-certificates \
    cmake \
    curl \
    gcc \
    git \
    gpg \
    jq \
    libarchive-dev \
    libbz2-dev \
    libffi-dev \
    libfuzzy-dev \
    libgmp-dev \
    libgomp1 \
    libjansson-dev \
    liblzma-dev \
    libmagic-dev \
    libmagic-mgc \
    libncurses-dev \
    libpcap-dev \
    libreadline-dev \
    libsqlite3-dev \
    libssl-dev \
    libssl3 \
    libtool \
    libzbar0 \
    make \
    ncat \
    openssh-client \
    openssl \
    pkg-config \
    python-is-python3 \
    python3-dev \
    python3-pip \
    python3-venv \
    python3-pkg-resources \
    python3-setuptools \
    redis-server \
    supervisor \
    swig \
    tesseract-ocr \
    tini \
    unrar \
    unzip \
    upx \
    yasm \
    zlib1g-dev && \
    cpan -i Compress::Raw::Lzma && \
    rm -rf /var/lib/apt/lists/*

# Install libxar; this is a complicated process because Ubuntu pre-25.04 lacks
# libxar1... Essentially we pull the upstream source and patches from Ubuntu's archives,
# fudge the autoconf version (because we only have 2.71, not 2.72, neat) and patch out a
# kernel EXT2 flag that isn't present on 24.04, and only -then- can we build.
ADD build/python/backend/xar-ext2-no-ecompr.patch /tmp
RUN mkdir -p /tmp/build-xar && \
    cd /tmp/build-xar && \
    mv ../xar-ext2-no-ecompr.patch . && \
    curl -OL http://archive.ubuntu.com/ubuntu/pool/universe/x/xar/xar_1.8.0.498.orig.tar.gz && \
    curl -OL http://archive.ubuntu.com/ubuntu/pool/universe/x/xar/xar_1.8.0.498-2.debian.tar.xz && \
    tar -xf xar_1.8.0.498.orig.tar.gz && \
    tar -xf xar_1.8.0.498-2.debian.tar.xz && \
    cd xar-xar-498 && \
    for patchfile in $(cat ../debian/patches/series); do \
        patch -Np1 -i ../debian/patches/$patchfile; \
    done; \
    patch -Np1 -i ../xar-ext2-no-ecompr.patch && \
    cd xar && \
    (curl -fsSL --retry 3 --retry-delay 2 https://git.savannah.gnu.org/cgit/config.git/plain/config.guess -o config.guess || \
     curl -fsSL https://raw.githubusercontent.com/cgitmirror/config/master/config.guess -o config.guess) && \
    (curl -fsSL --retry 3 --retry-delay 2 https://git.savannah.gnu.org/cgit/config.git/plain/config.sub -o config.sub || \
     curl -fsSL https://raw.githubusercontent.com/cgitmirror/config/master/config.sub -o config.sub) && \
    chmod +x config.guess config.sub && \
    sed -i 's/AC_PREREQ(\[2.72\])/AC_PREREQ([2.71])/' configure.ac && \
    autoreconf -fi && \
    ./configure && \
    make -sj4 && \
    make -s install && \
    cd /tmp && \
    rm -rf /tmp/build-xar

# Install JTR; this is -very, very slow to build- so do it -as early as
# possible- so we hopefully don't ever have to redo it. Ever. Please. The
# maintainer isn't big on releases or tags so grab an arbitrary, but consistent,
# commit. Additionally jump through some extra hoops to get the single commit to
# save some download time.
RUN mkdir -p /tmp/build-jtr && \
    cd /tmp/build-jtr && \
    curl -OL https://github.com/openwall/john-packages/archive/refs/tags/v$JOHN_VERSION.tar.gz && \
    tar -xf v$JOHN_VERSION.tar.gz && \
    JOHN_COMMIT=$(sed -rne 's/ *source-commit: *(.+)/\1/p' \
            john-packages-$JOHN_VERSION/deploy/snap/snapcraft.yaml) && \
    [ -n "$JOHN_COMMIT" ] && \
    mkdir /jtr && \
    cd /jtr && \
    git init && \
    git remote add origin https://github.com/openwall/john.git && \
    git fetch --depth 1 origin $JOHN_COMMIT && \
    git reset --hard FETCH_HEAD && \
    rm -rf .git && \
    cd src && \
    ./configure && \
    make -s clean && \
    make -sj4 && \
    make install && \
    cp -Tr /jtr/run/ /jtr && \
    rm -rf /jtr/run && \
    chmod -R 777 /jtr && \
    chown -R $USER_UID:$USER_UID /jtr && \
    cd /tmp && \
    rm -rf /tmp/build-jtr

# Download and compile archive library and exiftool
RUN mkdir -p /tmp/archive-zip /tmp/build-exiftool && \
    cd /tmp/archive-zip && \
    curl -OL https://cpan.metacpan.org/authors/id/P/PH/PHRED/Archive-Zip-$ARCHIVE_ZIP_VERSION.tar.gz && \
    tar -xzf Archive-Zip-$ARCHIVE_ZIP_VERSION.tar.gz && \
    cd Archive-Zip-$ARCHIVE_ZIP_VERSION/ && \
    perl Makefile.PL && \
    make && \
    make install && \
    cd /tmp/build-exiftool && \
    curl -OL https://github.com/exiftool/exiftool/archive/refs/tags/$EXIFTOOL_VERSION.tar.gz && \
    tar -xf $EXIFTOOL_VERSION.tar.gz && \
    cd exiftool-$EXIFTOOL_VERSION && \
    perl Makefile.PL && \
    make && \
    make install && \
    cd /tmp && \
    rm -rf /tmp/archive-zip /tmp/build-exiftool

# Download and move binary for 7z 26.00
RUN mkdir -p /tmp/install-7z && \
    cd /tmp/install-7z && \
    curl -o 7z.tar.xz -fsSL https://7-zip.org/a/7z2600-linux-$(uname -m | sed 's/x86_64/x64/' | sed 's/aarch64/arm64/').tar.xz && \
    tar -xf 7z.tar.xz && \
    cp 7zz /usr/bin/ && \
    cd /tmp && \
    rm -rf /tmp/install-7z

# Install YARA and yara-python
RUN mkdir -p /tmp/build-yara /tmp/build-yara-python && \
    cd /tmp/build-yara && \
    curl -OL https://github.com/VirusTotal/yara/archive/v$YARA_VERSION.tar.gz && \
    tar -xf v$YARA_VERSION.tar.gz && \
    cd yara-$YARA_VERSION && \
    ./bootstrap.sh && \
    CC='gcc' CFLAGS='-O0 -fPIC -fno-optimize-sibling-calls' ./configure --with-crypto --enable-magic --enable-cuckoo && \
    make -j1 && \
    make install && \
    cd /tmp && \
    cd /tmp/build-yara-python && \
    curl -OL https://github.com/VirusTotal/yara-python/archive/v$YARA_PYTHON_VERSION.tar.gz && \
    tar -xf v$YARA_PYTHON_VERSION.tar.gz && \
    cd yara-python-$YARA_PYTHON_VERSION && \
    python3 setup.py build --dynamic-linking && \
    python3 setup.py install && \
    cd /tmp && \
    rm -rf /tmp/build-yara /tmp/build-yara-python

# install zeek
ARG ZEEK_INTEGRATION=1
RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,target=/var/lib/apt,sharing=locked \
    if [ "$ZEEK_INTEGRATION" = "1" ]; then \
        echo "Zeek enabled — installing…"; \
        echo 'deb http://download.opensuse.org/repositories/security:/zeek/xUbuntu_24.04/ /' \
            | tee /etc/apt/sources.list.d/security:zeek.list && \
        curl -fsSL https://download.opensuse.org/repositories/security:zeek/xUbuntu_24.04/Release.key | gpg --dearmor | tee /etc/apt/trusted.gpg.d/security_zeek.gpg > /dev/null && \
        apt-get -qq update && \
        apt-get install -qq -y --no-install-recommends zeek-lts && \
        apt-mark manual zeek-lts && \
        cp /opt/zeek/bin/zeek /usr/bin/ && \
        rm -rf /opt/zeek/share/btest /opt/zeek/bin && \
        rm -rf /opt/zeek/lib/*.a && \
        rm -rf /var/lib/apt/lists/*; \
    else \
        echo "Zeek disabled — skipping."; \
    fi
ENV PATH=$PATH:/opt/zeek/bin

# install suricata
ARG SURICATA_INTEGRATION=1
ARG SURICATA_VERSION_PATTERN="1:7.0.*"
RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,target=/var/lib/apt,sharing=locked \
    if [ "$SURICATA_INTEGRATION" = "1" ]; then \
        echo "Suricata enabled — installing…"; \
        apt-get -qq update && \
        apt-get install -qq -y --no-install-recommends \
            suricata=${SURICATA_VERSION_PATTERN} \
            suricata-update && \
        rm -rf /var/lib/apt/lists/*; \
    else \
        echo "Suricata disabled — skipping."; \
    fi

#Copy Suricata config to default location
COPY configs/python/backend/suricata/suricata.yaml /etc/suricata/suricata.yaml

# Build/install CAPA
RUN mkdir -p /tmp/build-capa && \
    python3 -m venv /tmp/build-capa && \
    cd /tmp/build-capa && \
    . ./bin/activate && \
    curl -fsSLo ./capa.zip https://github.com/mandiant/capa/archive/refs/tags/v$CAPA_VERSION.zip && \
    curl -fsSLo ./rules.zip https://github.com/mandiant/capa-rules/archive/refs/tags/v$CAPA_VERSION.zip && \
    unzip -q ./capa.zip && \
    unzip -q ./rules.zip && \
    cd capa-$CAPA_VERSION && \
    python3 -m pip install -e .[build] && \
    mv ../capa-rules-$CAPA_VERSION/* ./rules/ && \
    python3 ./scripts/cache-ruleset.py rules/ cache && \
    pyinstaller ./.github/pyinstaller/pyinstaller.spec && \
    chmod 755 ./dist/capa && \
    mv ./dist/capa /tmp/capa && \
    deactivate && \
    mv /tmp/capa /usr/bin/ && \
    chown root:root /usr/bin/capa && \
    rm -rf /tmp/build-capa

# Install ClamAV
RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,target=/var/lib/apt,sharing=locked \
    apt-get update -qq -qq && \
    apt-get install -qq -y --no-install-recommends \
    clamav \
    clamav-base \
    clamav-daemon \
    clamav-freshclam && \
    rm -rf /var/lib/apt/lists/*

# Update permissions for relevant fresclam log files
RUN touch /var/log/clamav/freshclam.log && \
    chmod 777 /var/log/clamav/freshclam.log && \
    chown clamav:clamav /var/log/clamav/freshclam.log && \
    chown 1001:1001 /var/lib/clamav

USER $USERNAME

# Run freshclam so that we have -something- for signatures in the image (fix this)
RUN freshclam

USER root

# Install Poetry globally and copy project files
ENV POETRY_HOME="/opt/poetry"
ENV POETRY_VIRTUALENVS_IN_PROJECT=false
ENV POETRY_NO_INTERACTION=1

# Ensure the poetry bin directory is added to the PATH
ENV PATH="$POETRY_HOME/bin:$PATH"

# Install Poetry using its official installation script
RUN curl -sSL https://install.python-poetry.org | python3 -

# Set the working directory and copy the project files
WORKDIR /strelka/
COPY pyproject.toml poetry.lock ./

# Use Poetry to install the project dependencies globally
# This step is after the COPY step because it is more likely to change,
# and therefore should not be included in earlier layers that can be cached.
RUN python3 -m pip install --break-system-packages --no-cache-dir --no-compile --upgrade \
    pyinstaller \
    ruamel.yaml \
    rich-argparse \
    semver \
    'ssdeep==3.4' && \
    poetry config virtualenvs.create false && \
    rm -f /usr/lib/python3.12/EXTERNALLY-MANAGED && \
    poetry install --only main && \
    python3 -c "import cv2, numpy; print('cv2', cv2.__version__, 'numpy', numpy.__version__)" && \
    rm -rf /root/.cache/pypoetry

# Copy Strelka files
COPY ./src/python/ /strelka/
COPY ./configs/python/backend/ /etc/strelka/

# Install Strelka
RUN cd /strelka/ && \
    rm -f pyproject.toml poetry.lock && \
    python3 -m pip install --break-system-packages --no-cache-dir --no-build-isolation . && \
    (cd /tmp && python3 -c "import strelka.backend; import importlib; [importlib.import_module(f'strelka.scanners.{m}') for m in ['scan_lsb','scan_nf']]; print('scanner imports OK'); import strelka.auxiliary.xar; assert hasattr(strelka.auxiliary.xar, 'XarArchive')") && \
    # Remove build artifacts and packages
    rm -rf build/ dist/ *.egg-info && \
    apt-get purge -y \
    automake \
    build-essential \
    cmake \
    curl \
    gcc \
    git \
    gpg \
    libtool \
    make \
    python3-dev \
    python3-pip \
    python3-wheel && \
    apt-get clean -qq && \
    rm -rf /var/lib/apt/lists/* && \
# Assign permissions to Strelka scan result logging directory
    mkdir /var/log/strelka/ && \
    chgrp -R 0 /var/log/strelka/ && \
    chmod -R g=u /var/log/strelka/

## FIXME: tests currently disabled while they're being fixed still, sorry!
# # Run tests as non-root user
# USER $USERNAME

# # Run build checks
# # Use --build-arg SCANNER_TEST=test_scan_<scannername>.py to ignore all but selected scanner
# # ex: docker-compose build --build-arg SCANNER_TEST=test_scan_yara.py
# ARG SCANNER_TEST=""

# # Run tests. Fail build if any tests fail.
# RUN echo '[+] Run build checks' && \
#     cd /strelka/strelka/ && \
#     if $SCANNER_TESTS; then python3 -m pytest -p no:cacheprovider -s tests/$SCANNER_TEST; fi && \
#     if $CONFIG_TESTS; then python3 -m pytest -s tests_configuration/; fi && \
#     echo '[+] Done'

# USER root

# Remove build directories and unused files
RUN cd /strelka/ && \
    rm -rf /strelka/ && \
    rm -rf /root/.cache && \
    rm -rf /tmp/*

# Remove config directory (will bind mount once built)
RUN rm -rf /etc/strelka/

# FIXME: use a script to drop permissions on entry, so that we can install plugins, etc.
USER $USERNAME

ENTRYPOINT ["/usr/bin/tini", "--"]
CMD ["strelka-backend"]

