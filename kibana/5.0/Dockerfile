FROM kibana:5

RUN apt-get update && apt-get install -y netcat bzip2

# Copy logstash config files (often only logstash.conf) into logstash config dir in the container
COPY config/* /opt/kibana/config/

COPY entrypoint.sh /tmp/entrypoint.sh
RUN chmod +x /tmp/entrypoint.sh

CMD ["/tmp/entrypoint.sh"]
