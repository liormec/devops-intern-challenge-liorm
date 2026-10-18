# Proxy image inherited from the previous team.

FROM nginx:alpine

COPY nginx.conf /etc/nginx/nginx.conf

# Give nginx user permissions for runtime cache and PID
RUN touch /tmp/nginx.pid && \
    chown -R nginx:nginx /var/cache/nginx /tmp/nginx.pid

# Run as non-root user (UID 101 in Alpine)
USER nginx

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
