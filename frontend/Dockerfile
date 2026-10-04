FROM nginxinc/nginx-unprivileged:alpine

COPY frontend/nginx.conf /etc/nginx/nginx.conf
COPY frontend/index.html /usr/share/nginx/html/index.html
COPY frontend/images/ /usr/share/nginx/html/images/

EXPOSE 8080
