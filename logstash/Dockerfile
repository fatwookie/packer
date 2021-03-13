FROM docker.elastic.co/logstash/logstash:7.11.1

MAINTAINER fatwookie

COPY config/pipelines.yml   /usr/share/logstash/config/pipelines.yml
COPY config/logstash.yml    /usr/share/logstash/config/logstash.yml
COPY config/pipeline        /usr/share/logstash/pipeline
