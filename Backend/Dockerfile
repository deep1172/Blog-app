# Stage 1: Build
FROM node:18-alpine as builder

WORKDIR /app
COPY backend/package*.json ./
RUN npm install
COPY backend/ .
RUN npm run build

# Stage 2: Run
FROM node:18-alpine

WORKDIR /app
COPY --from=builder /app /app
EXPOSE 5000
CMD ["node", "server.js"]
