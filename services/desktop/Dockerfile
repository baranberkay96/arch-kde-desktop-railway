# syntax=docker/dockerfile:1
#
# Arch Linux with KDE Plasma (Wayland), in the browser, for Railway.
#
# Base: linuxserver.io webtop, release arch-kde-2025-07-12-ls271 (built
# 2026-11-18 from the arch-kde branch of linuxserver/docker-webtop, on
# baseimage-selkies:arch). Arch is a rolling release: the image is a snapshot
# of Arch on its build day. The digest is the multi-arch index, identical on
# lscr.io, ghcr.io and Docker Hub. To upgrade, replace tag and digest together
# with a newer "arch-kde-<version>-ls<n>" release and redeploy.
FROM lscr.io/linuxserver/webtop:arch-kde-2025-07-12-ls271@sha256:326c66c83c3c6723fe758687a11ac9f7e798845899c2664df8a416ac617090a7

# Image defaults for Railway. Variables set on the Railway service win.
#  - PUID/PGID 1000: fixed owner of the /config volume. Do not change them
#    after the first boot, because files already on the volume keep the old ids.
#  - START_DOCKER=false: Railway gives no privileged containers, so the
#    bundled Docker daemon can never run there.
#  - SELKIES_ALLOWED_ORIGINS empty: the image ships "*", which lets any
#    website open the desktop's WebSocket with the browser's cached login.
#    Empty is Selkies' own default: same-origin only.
#  - SELKIES_ENABLE_BASIC_AUTH=false: nginx does the login. Selkies listens
#    on loopback only.
#  - PIXELFLUX_WAYLAND=true: KDE runs as a Wayland session, as upstream.
ENV PUID=1000 \
    PGID=1000 \
    START_DOCKER=false \
    SELKIES_ALLOWED_ORIGINS="" \
    SELKIES_ENABLE_BASIC_AUTH=false \
    PIXELFLUX_WAYLAND=true \
    TITLE="Arch Linux KDE Plasma Desktop" \
    CUSTOM_USER=arch \
    PORT=3000

COPY patch-image.sh /tmp/railway-desktop/patch-image.sh
COPY nginx-origin-map.conf nginx-server-hardening.conf /tmp/railway-desktop/
COPY railway-start.sh /usr/local/bin/railway-start
COPY pkg-restore.sh /usr/local/bin/railway-pkg-restore

# Build-time edits of the image's nginx files. The script checks every
# anchor it edits and fails the build if the upstream files have changed.
RUN bash /tmp/railway-desktop/patch-image.sh \
 && rm -rf /tmp/railway-desktop \
 && chmod 0755 /usr/local/bin/railway-start /usr/local/bin/railway-pkg-restore

# A pacman configuration that installs from the Arch Linux Archive snapshot of
# the day after the image's newest package was installed, so packages listed
# in PACMAN_PACKAGES match the libraries of this pinned image (no partial
# upgrade). The regular /etc/pacman.conf is left as it is: "sudo pacman -Syu"
# in the desktop still updates from the live mirrors.
RUN set -eux; \
    newest="$(sed -n '/^%INSTALLDATE%$/{n;p;}' /var/lib/pacman/local/*/desc | sort -n | tail -n 1)"; \
    [ -n "$newest" ]; \
    day="$(date -u -d "@$(( newest + 86400 ))" +%Y/%m/%d)"; \
    printf '# Arch Linux Archive snapshot matching this image (written at build time).\nServer = https://archive.archlinux.org/repos/%s/$repo/os/$arch\n' "$day" \
      > /etc/pacman.d/mirrorlist-railway-snapshot; \
    sed -E 's|^Include *= */etc/pacman\.d/mirrorlist$|Include = /etc/pacman.d/mirrorlist-railway-snapshot|' \
      /etc/pacman.conf > /etc/pacman-railway-snapshot.conf; \
    grep -q '^Include = /etc/pacman.d/mirrorlist-railway-snapshot$' /etc/pacman-railway-snapshot.conf; \
    cat /etc/pacman.d/mirrorlist-railway-snapshot

EXPOSE 3000

# Railway runs its own init (podman-init) as PID 1, and the image's /init
# (s6-overlay 3.2.1.0) refuses to start unless it is PID 1. railway-start
# brings up the same s6 supervision tree without that check. catatonit
# (part of the linuxserver.io base image) reaps zombies and forwards signals,
# as a child subreaper when it is not PID 1 itself.
ENTRYPOINT ["/usr/bin/catatonit", "--", "/usr/local/bin/railway-start"]
