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

# Add customizations below. Hadron is a musl-based immutable system; it does
# NOT ship apk. To add packages, you must use the Kairos bundles mechanism or
# build from a base distro with kairos-init. For now, we only stamp a version.
#
# NOTE: Hadron v0.5.1 is a "nogit" build — git is not installed and the
# kairos-agent git stage plugin is unavailable. Our cloud-config uses
# curl+tar instead (see cloud-config/10_git-pull.yaml). If we ever need
# the native git stage module, we'd need to build from a base that includes
# git or add it via a bundle.

# Stamp a custom version so `kairos-agent upgrade list-releases` and
# /etc/os-release identify this as our image, not stock hadron.
ARG KAIROS_TEST_VERSION=0.1.0
RUN echo "KAIROS_TEST_VERSION=${KAIROS_TEST_VERSION}" >> /etc/os-release
