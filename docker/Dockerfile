# This Source Code Form is subject to the terms of the Mozilla Public
# License, v. 2.0. If a copy of the MPL was not distributed with this
# file, You can obtain one at http://mozilla.org/MPL/2.0/.
# Copyright (c) 2014 Mozilla Corporation
#
# Contributors:
# Jeff Bryner jbryner@mozilla.com
# Yohann Lepage yohann@lepage.info
# Anthony Verez averez@mozilla.com
# Charlie Lewis clewis@iqt.org
# Brandon Myers bmyers@mozilla.com

FROM centos:7

MAINTAINER mozdef@mozilla.com

ENV NODE_VERSION 4.7.0
ENV METEOR_VERSION 1.4.0.1
ENV METEOR_FILE_VERSION 1.4.0-1
ENV PYTHON_VERSION 2.7.11
ENV KIBANA_VERSION 4.5.4
ENV ES_VERSION 2.4.2
ENV ES_JAVA_VERSION 1.8.0
ENV RABBITMQ_VERSION 3.3.5

ENV MONGO_URL=mongodb://localhost:3002/meteor
ENV ROOT_URL=http://localhost
ENV PORT=3000

COPY docker/conf/mongodb.repo /etc/yum.repos.d/mongodb.repo

# Install ES, RabbitMQ, nginx, Kibana, python, Node, Meteor
RUN \
  yum clean all && \
  yum install -y epel-release && \
  yum install -y \
                    wget \
                    java-$ES_JAVA_VERSION \
                    glibc-devel \
                    gcc \
                    gcc-c++ \
                    libstdc++ \
                    supervisor \
                    libffi-devel \
                    zlib-devel \
                    cronie && \
  useradd -ms /bin/bash -d /opt/mozdef -m mozdef && \
  mkdir /opt/mozdef/envs && \
  curl -s -L https://download.elasticsearch.org/elasticsearch/elasticsearch/elasticsearch-$ES_VERSION.tar.gz  | tar -C /opt/mozdef -xz && \
  mv /opt/mozdef/elasticsearch-$ES_VERSION /opt/mozdef/envs/elasticsearch && \
  rpm --import https://www.rabbitmq.com/rabbitmq-release-signing-key.asc && \
  yum install -y rabbitmq-server-$RABBITMQ_VERSION && \
  yum install -y nginx && \
  mkdir /var/log/mozdef/ && \
  curl -s -L https://download.elastic.co/kibana/kibana/kibana-$KIBANA_VERSION-linux-x64.tar.gz | tar -C /opt/mozdef/ -xz && \
  mv /opt/mozdef/kibana-$KIBANA_VERSION-linux-x64 /opt/mozdef/envs/kibana && \
  yum install -y mongodb-org && \
  curl -s -L https://www.python.org/ftp/python/$PYTHON_VERSION/Python-$PYTHON_VERSION.tgz  | tar -C /opt/mozdef/ -xz && \
  cd /opt/mozdef/Python-$PYTHON_VERSION && \
  ./configure && \
  make && \
  make install && \
  rm -r /opt/mozdef/Python-$PYTHON_VERSION && \
  cd /opt/mozdef && \
  yum install -y mysql-devel \
                    python-devel \
                    python-pip && \
  chown -R mozdef:mozdef /opt/mozdef/ && \
  pip install virtualenv && \
  cd / && \
  curl -sL -o /opt/mozdef/nodesource.rpm https://rpm.nodesource.com/pub_4.x/el/7/x86_64/nodesource-release-el7-1.noarch.rpm && \
  rpm -i --nosignature --force /opt/mozdef/nodesource.rpm && \
  yum install -y nodejs-$NODE_VERSION && \
  npm install source-map-support@0.4.2 \
                 semver@5.3.0 \
                 fibers@1.0.13 \
                 amdefine@1.0.0 \
                 underscore@1.8.3 \
                 bcrypt && \
  mkdir /opt/mozdef/meteor && \
  curl -sL -o /opt/mozdef/meteor.tar.gz https://meteorinstall-4168.kxcdn.com/packages-bootstrap/$METEOR_VERSION/meteor-bootstrap-os.linux.x86_64.tar.gz && \
  tar -xzf /opt/mozdef/meteor.tar.gz -C /opt/mozdef/meteor && \
  mv /opt/mozdef/meteor/.meteor /opt/mozdef && \
  rm -r /opt/mozdef/meteor && \
  cp /opt/mozdef/.meteor/packages/meteor-tool/$METEOR_FILE_VERSION/mt-os.linux.x86_64/scripts/admin/launch-meteor /usr/bin/meteor

COPY meteor /opt/mozdef/envs/mozdef/meteor
RUN chown -R mozdef:mozdef /opt/mozdef/envs/mozdef/meteor
USER mozdef
RUN \
  cd /opt/mozdef/envs/mozdef/meteor && \
  meteor npm install --save babel-runtime && \
  meteor add accounts-password && \
  meteor add npm-bcrypt && \
  mkdir -p /opt/mozdef/envs/meteor/mozdef
USER root
COPY docker/conf/settings.js /opt/mozdef/envs/mozdef/meteor/app/lib/settings.js
RUN chown -R mozdef:mozdef /opt/mozdef/envs/mozdef/meteor/app/lib/settings.js
USER mozdef
RUN \
  cd /opt/mozdef/envs/mozdef/meteor && \
  meteor build --server localhost:3002 --directory /opt/mozdef/envs/meteor/mozdef/ --allow-incompatible-update

# Create python virtual environment and install dependencies
COPY requirements.txt /opt/mozdef/envs/mozdef/requirements.txt
RUN \
  virtualenv /opt/mozdef/envs/python && \
  source /opt/mozdef/envs/python/bin/activate && \
  pip install -r /opt/mozdef/envs/mozdef/requirements.txt

USER root
COPY docker/conf/elasticsearch.yml /opt/mozdef/envs/elasticsearch/config/
COPY docker/conf/supervisor.conf /etc/supervisor/conf.d/supervisor.conf
COPY docker/conf/mongod.conf /etc/mongod.conf
COPY docker/conf/rabbitmq.config /etc/rabbitmq/
COPY docker/conf/enabled_plugins /etc/rabbitmq/
COPY docker/conf/nginx.conf /etc/nginx/nginx.conf

COPY static /opt/mozdef/envs/mozdef/static
COPY rest /opt/mozdef/envs/mozdef/rest
COPY loginput /opt/mozdef/envs/mozdef/loginput
COPY bot /opt/mozdef/envs/mozdef/bot
COPY lib /opt/mozdef/envs/mozdef/lib
COPY cron /opt/mozdef/envs/mozdef/cron
COPY alerts /opt/mozdef/envs/mozdef/alerts
COPY mq /opt/mozdef/envs/mozdef/mq

COPY docker/conf/config.py /opt/mozdef/envs/mozdef/alerts/lib/config.py
COPY docker/conf/cron_entries.txt /cron_entries.txt

USER mozdef
RUN crontab /cron_entries.txt

USER root
RUN \
  mkdir /opt/mozdef/envs/mozdef/data && \
  mkdir /opt/mozdef/envs/mozdef/config && \
  mkdir -p /opt/mozdef/envs/mozdef/docker/conf

COPY docker/conf/initial_setup.py /opt/mozdef/envs/mozdef/docker/conf/initial_setup.py
COPY config/defaultMappingTemplate.json /opt/mozdef/envs/mozdef/config/defaultMappingTemplate.json

# We have to replace the python virtualenv path until
# https://github.com/mozilla/MozDef/issues/421 is fixed
RUN sed -i 's|/opt/mozdef/envs/mozdef/bin/activate|/opt/mozdef/envs/python/bin/activate|g' /opt/mozdef/envs/mozdef/cron/*.sh

RUN chown -R mozdef:mozdef /opt/mozdef/envs/mozdef

# VOLUMES
#  Elasticsearch
VOLUME /var/lib/elasticsearch
#  Mongodb
VOLUME /var/lib/mongo
# RabbitMQ
VOLUME /var/lib/rabbitmq
# MozDef data (geolite db for example)
VOLUME /opt/mozdef/envs/mozdef/data

# 80 = MozDef Web UI (meteor)
# 3002 = Mongodb
# 5672 = RabbitMQ
# 15672 = RabbitMQ Management
# 8080 = Loginput
# 8081 = RestAPI
# 9090 = Kibana Web UI "localhost:9090/app/kibana"
# 9200 = Elasticsearch
EXPOSE 80 3002 5672 15672 8080 8081 9090 9200

CMD supervisord -n -c /etc/supervisor/conf.d/supervisor.conf
