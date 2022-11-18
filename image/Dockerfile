FROM ubuntu:16.04

ARG KSI_TOOL_TAG=2.9.1374
ARG EXTRA_CERT_DIR=/usr/local/share/ca-certificates/extra/
ARG RSYSLOG_CONF_DIR=/etc
ARG LOGROTATE_SCRIPT_DIR=/etc/logrotate.d
ARG LOGROTATE_CONF_DIR=/etc


RUN mkdir -p /var/log-ksi



COPY globalsign.crt ${EXTRA_CERT_DIR}
COPY logrotate-rsyslog-ksi-integrate.conf ${LOGROTATE_CONF_DIR}
RUN chmod 644 ${LOGROTATE_CONF_DIR}/logrotate-rsyslog-ksi-integrate.conf


# Install c build tools.
# Install certificate for publications file verification.
# Clone and build KSI tool ubuntu package.
# Install KSI tool.
# Clean build tools.
RUN apt-get -y update \
	&& apt-get -y upgrade \
	&& apt-get -y install software-properties-common curl \
	&& add-apt-repository -y ppa:adiscon/v8-stable \
	&& apt-get -y update \
	&& apt-get -y install libfastjson4 libfastjson-dev libestr0 libestr-dev uuid-dev libgcrypt11-dev libsystemd-dev \
	&& apt-get -y install apt-transport-https ca-certificates \
	&& curl https://download.guardtime.com/ksi/GUARDTIME-GPG-KEY | apt-key add - \
	&& curl -o /etc/apt/sources.list.d/guardtime.list https://download.guardtime.com/ksi/configuration/guardtime.xenial.list \
	&& apt-get -y update \
	&& apt-get -y install git gcc make autoconf automake libtool pkg-config openssl libcurl4-gnutls-dev libssl-dev debmake \
	&& apt-get -y install libksi libksi-dev logksi gttlvutil ksi-tools\
	&& apt-get -y install flex bison python-docutils \
	&& apt-get -y install logrotate lsof\
	&& cd /tmp \
	&& git clone https://github.com/rsyslog/rsyslog.git \
	&& cd rsyslog \
	&& git checkout v8.2208.0 \
	&& autoreconf -fi \
	&& ./configure CFLAGS=-Og --enable-imptcp  --enable-ksi-ls12 \
	&& make install \
	&& dpkg -r software-properties-common \
	&& dpkg -r libfastjson-dev libestr-dev uuid-dev libgcrypt11-dev libsystemd-dev libksi-dev \
	&& dpkg -r git autoconf automake libcurl4-gnutls-dev libssl-dev debmake \
	&& dpkg -r flex bison python-docutils \
	&& cd .. \
	&& rm -rf /tmp/rsyslog


EXPOSE	10514

CMD	["rsyslogd", "-n", "-f", "/etc/rsyslog.conf"]
