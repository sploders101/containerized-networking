FROM debian:bookworm

RUN apt update \
  && apt install -y kea \
  && rm -rf /var/lib/apt/lists/*

RUN install -do _kea -g _kea /run/kea

CMD ["/usr/sbin/kea-dhcp4", "-c", "/etc/kea/kea-dhcp4.conf"]
