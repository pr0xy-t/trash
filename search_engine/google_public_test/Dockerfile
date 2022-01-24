FROM alpine:3.15

RUN apk update && \
	apk add --no-cache nginx
	

ADD public/* /var/www/html/
ADD conf/default.conf /etc/nginx/http.d/default.conf


EXPOSE 80


CMD nginx -g "daemon off;"
