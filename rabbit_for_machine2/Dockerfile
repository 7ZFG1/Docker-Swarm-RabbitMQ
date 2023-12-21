FROM rabbitmq:3.11-management

ADD rabbitmq.config /etc/rabbitmq/
ADD definitions.json /etc/rabbitmq/

RUN rabbitmq-plugins enable rabbitmq_prometheus
