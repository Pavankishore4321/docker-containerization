# ─────────────────────────────────────────────────────────────────────────────
# Dockerfile: Angular Frontend — Multi-Stage Build
# Project:    AspTax, P-Collab (Incresol Software Services)
#
# Stage 1 (builder): Installs Node.js, runs ng build → produces dist/
# Stage 2 (runtime): Copies dist/ into Nginx — no Node.js in final image
#
# Result: Very small production image (~25MB vs ~500MB with Node.js)
# Build:  docker build -t asptax-web:latest .
# Run:    docker run -d -p 80:80 asptax-web:latest
# ─────────────────────────────────────────────────────────────────────────────

# ── STAGE 1: BUILD ────────────────────────────────────────────────────────────
# Use Node.js to install dependencies and build Angular app
FROM node:18-alpine AS builder

# Set working directory inside container
WORKDIR /app

# Copy package files first (Docker cache optimization)
# If package.json didn't change, npm install is skipped on rebuild
COPY package.json package-lock.json ./

# Install all dependencies
RUN npm install --force

# Copy rest of the Angular source code
COPY . .

# Build Angular for production
# Output goes to /app/dist/<app-name>/
RUN npx ng build --configuration=production

# ── STAGE 2: RUNTIME ──────────────────────────────────────────────────────────
# Use lightweight Nginx — no Node.js, no npm, no source code in final image
FROM nginx:1.25-alpine

# Remove default Nginx welcome page
RUN rm -rf /usr/share/nginx/html/*

# Copy Angular build output from Stage 1 into Nginx web root
# Replace 'asptax-web' with your actual Angular app name from angular.json
COPY --from=builder /app/dist/asptax-web /usr/share/nginx/html

# Copy custom Nginx config for Angular (handles routing, gzip, headers)
COPY nginx.conf /etc/nginx/conf.d/default.conf

# Expose port 80
EXPOSE 80

# Health check — verify Nginx is serving
HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
  CMD wget --quiet --tries=1 --spider http://localhost/health || exit 1

# Start Nginx
CMD ["nginx", "-g", "daemon off;"]
