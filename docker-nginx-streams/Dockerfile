From nginx:latest

COPY nginx.conf /etc/nginx/nginx.conf
RUN mkdir -p /etc/nginx/streams
RUN mkdir -p /etc/nginx/templates
COPY stream.conf.template /etc/nginx/templates/stream.conf.template
