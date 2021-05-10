FROM nginx:alpine
RUN apk --no-cache add shadow && \
    usermod -u 1001 nginx && \
    mkdir /app && \
    chown nginx:nginx /app /var/cache/nginx /var/run
COPY app.conf /etc/nginx/conf.d/
COPY app /app
USER nginx
EXPOSE 8000
ENTRYPOINT ["nginx"]
CMD ["-g", "daemon off;"]