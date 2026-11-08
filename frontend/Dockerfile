FROM node:25-alpine AS build
WORKDIR /app
COPY frontend/app_frontend/package*.json ./
RUN npm ci
COPY frontend/app_frontend/ ./
RUN npm run build


FROM nginx:alpine
COPY frontend/nginx.conf /etc/nginx/conf.d/default.conf
COPY --from=build /app/dist /usr/share/nginx/html
EXPOSE 80