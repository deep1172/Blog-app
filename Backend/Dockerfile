# Stage 1: Install deps
FROM node:22-alpine AS deps
WORKDIR /app

# Install only production deps first
COPY package*.json ./
RUN npm install --production

# Stage 2: Copy and run
FROM node:22-alpine
WORKDIR /app

# Copy installed deps and source code
COPY --from=deps /app/node_modules ./node_modules
COPY . .

# Expose backend port
EXPOSE 5000

# Start the app
CMD ["node", "index.js"]
