# ---------- Stage 1: Build ----------
FROM node:20-alpine AS build

WORKDIR /app

COPY package.json package-lock.json ./
RUN npm ci

COPY . .
RUN npm run build -- --configuration production


# ---------- Stage 2: Serve ----------
FROM nginxinc/nginx-unprivileged:1.25-alpine AS runtime

COPY --from=build /app/dist/angular-conduit /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=3s CMD wget -qO- http://127.0.0.1:8080/ || exit 1