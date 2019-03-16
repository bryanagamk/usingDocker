FROM debian:wheezy

MAINTAINER Bryan Agam K <bryanagamk@gmail.com>
RUN apt-get update && apt-get install -y cowsay fortune
COPY entrypoint.sh /

ENTRYPOINT ["/entrypoint.sh"]
