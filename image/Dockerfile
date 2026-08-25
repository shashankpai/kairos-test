# syntax=docker/dockerfile:1
# Custom Kairos image for the Raspberry Pi 4 edge node.
#
# Per https://kairos.io/docs/advanced/customizing/, extending an existing
# Kairos image only requires FROM + adding packages. Do NOT re-run kairos-init
# here; it is only needed when starting from a raw base distro (ubuntu, alpine,
# etc.). The hadron image below is already a valid Kairos artifact.
#
# Build for arm64 (the Pi's architecture):
#   docker build --platform linux/arm64 -t ghcr.io/shashankpai/kairos-pi:0.1.0 .
#
# The GitHub Actions workflow in .github/workflows/build-and-publish.yaml
# handles platform, tagging, and pushing to GHCR.

FROM quay.io/kairos/hadron:v0.5.1-standard-arm64-rpi4-v4.2.0-k3s-v1.36.3-k3s1

# Add customizations below. Hadron is Alpine-based, so use apk.
# Example:
#   RUN apk add --no-cache figlet
# Or copy in config files:
#   COPY my-config.toml /etc/foo/config.toml

# Stamp a custom version so `kairos-agent upgrade list-releases` and
# /etc/os-release identify this as our image, not stock hadron.
ARG KAIROS_TEST_VERSION=0.1.0
RUN echo "KAIROS_TEST_VERSION=${KAIROS_TEST_VERSION}" >> /etc/os-release
