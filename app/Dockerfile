
# ============================================================
# Dockerfile — DevOps Observability Project
# Packages Nginx + custom config + dashboard into one image
# Base: nginx:alpine (official, minimal, ~5MB)
# ============================================================

# -----------------------------------------------------------
# FROM — Layer 1: Start from official Nginx on Alpine Linux
# alpine = tiny Linux distro, only 5MB, fewer attack surfaces
# than full Ubuntu (which is 70MB+)
# nginx:alpine already has Nginx installed and ready
# -----------------------------------------------------------
FROM nginx:1.27.5-alpine




# -----------------------------------------------------------
# LABEL — image metadata
# Documents who built this, what it does, which version
# Visible via: docker inspect devops-observability:v1
# -----------------------------------------------------------
LABEL maintainer="Muskan <amity-mca-2026>" \
      project="devops-observability" \
      version="5.0" \
      description="Nginx observability dashboard for MTTR research — exact version pinned"



     

# -----------------------------------------------------------
# RUN — Build time: Delete default Nginx config
# nginx:alpine ships with a default server config at this path
# It conflicts with our custom nginx.conf
# We delete it so only OUR config exists
# -----------------------------------------------------------
RUN rm /etc/nginx/conf.d/default.conf

RUN apk update && apk upgrade --no-cache && rm -rf /var/cache/apk/*
# -----------------------------------------------------------
# COPY — our nginx.conf replaces the default main config
# Source: app/nginx.conf (on your WSL machine)
# Destination: /etc/nginx/nginx.conf (inside the image)
# This is where Nginx always looks for its main config
# -----------------------------------------------------------
COPY nginx.conf /etc/nginx/nginx.conf

# -----------------------------------------------------------
# COPY — our dashboard HTML into Nginx's serving directory
# Source: app/index.html (on your WSL machine)
# Destination: /usr/share/nginx/html/index.html (inside image)
# This matches the root directive in our nginx.conf exactly
# -----------------------------------------------------------
COPY index.html /usr/share/nginx/html/index.html

# -----------------------------------------------------------
# EXPOSE — document that this container uses port 80
# This is metadata only — does NOT open any firewall port
# Actual port mapping happens in docker run -p 80:80
# AWS Security Group handles the real firewall rule
# -----------------------------------------------------------
EXPOSE 80

# -----------------------------------------------------------
# HEALTHCHECK — Docker's built-in container health monitor
# Every 30s: curl http://localhost/health
# If no 200 response within 5s = unhealthy
# After 3 consecutive failures = container marked UNHEALTHY
# This shows in docker ps output — visible evidence for report
# Directly uses the /health endpoint we built in nginx.conf
# -----------------------------------------------------------
HEALTHCHECK --interval=30s --timeout=5s --start-period=5s --retries=3 \
    CMD wget --no-verbose --tries=1 --spider http://127.0.0.1/health || exit 1
# -----------------------------------------------------------
# CMD — what runs when container starts
# nginx = start nginx
# -g = pass a global directive
# daemon off = stay in foreground so Docker watches PID 1
# Array format ["",""] is preferred over string format
# -----------------------------------------------------------
CMD ["nginx", "-g", "daemon off;"]