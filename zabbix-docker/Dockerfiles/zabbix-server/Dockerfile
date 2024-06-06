ARG ZABBIX_SERVER_IMAGE
ARG ZABBIX_IMAGE_TAG
ARG ZABBIX_IMAGE_TAG_POSTFIX

FROM ${ZABBIX_SERVER_IMAGE}:${ZABBIX_IMAGE_TAG}${ZABBIX_IMAGE_TAG_POSTFIX}

USER root

RUN apt-get update && \
    apt-get install -y --no-install-recommends odbc-postgresql && \
    rm -rf /var/lib/apt/lists

USER zabbix