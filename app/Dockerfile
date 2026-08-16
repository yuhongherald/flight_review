# Self-hosted PX4 Flight Review, forked to render UAV-SEAD annotations.
# Multi-stage: the venv is built and slimmed in one image and copied into a clean one.
# Measured 624 MB, against 1.23 GB for the single-stage ubuntu:noble build this replaces.

FROM python:3.12-slim AS builder

RUN apt-get update \
	&& apt-get install -y --no-install-recommends binutils \
	&& rm -rf /var/lib/apt/lists/*

# Drop dev/notebook-only deps (jupyter's whole tree + pylint). tr first: a Windows checkout
# hands over CRLF, and the trailing \r would put itself between the name and the anchor, so
# every line survives the filter and the image gains ~250 MB of dev tree.
COPY requirements.txt /tmp/requirements.txt
RUN tr -d '\r' < /tmp/requirements.txt | grep -viE '^(jupyter|pylint)([<>=~ ]|$)' > /tmp/req.txt
RUN python3 -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"
# --no-compile skips ~110 MB of .pyc
RUN pip install --no-cache-dir --no-compile -r /tmp/req.txt

# Slim the venv: drop bundled test suites (never imported at runtime), pip/wheel, and
# debug symbols in the native shared libs (strip --strip-unneeded is safe for .so).
RUN find /opt/venv -type d -name tests -prune -exec rm -rf {} + \
	&& find /opt/venv -type d -name __pycache__ -prune -exec rm -rf {} + \
	&& rm -rf /opt/venv/lib/python*/site-packages/pip \
	          /opt/venv/lib/python*/site-packages/wheel \
	&& find /opt/venv -name '*.so*' -type f -exec strip --strip-unneeded {} + 2>/dev/null || true


FROM python:3.12-slim

RUN apt-get update \
	&& apt-get install -y --no-install-recommends sqlite3 libfftw3-bin \
	&& rm -rf /var/lib/apt/lists/*

WORKDIR /opt/service
COPY --from=builder /opt/venv /opt/venv
COPY . /opt/service/
ENV PATH="/opt/venv/bin:$PATH"
RUN mkdir -p /opt/service/data
# run.sh is exec'd, so a CRLF checkout makes the kernel look for an interpreter called
# `/bin/bash\r` and the container crash-loops on "no such file or directory".
RUN sed -i 's/\r$//' /opt/service/run.sh && chmod +x /opt/service/run.sh

# plot_app/libevents is a submodule: an uninitialized one copies as an empty directory and
# the server starts, then fails on import. Fail the build instead.
RUN test -f /opt/service/plot_app/libevents/libs/python/libevents_parse/parser.py \
	|| (echo 'libevents submodule missing: git submodule update --init --recursive' && false)

EXPOSE 5006
CMD ["/opt/service/run.sh"]

## USAGE
## ## ## ## ## ## ## ## ## ## ## ##
## BUILD  docker build -t yuhongherald/px4-flight-review:$(date +%Y%m%d) app
## RUN    docker compose up -d
