FROM node:18-slim

WORKDIR /app

# Install dependencies first (layer caching)
COPY package*.json ./
RUN npm ci --omit=dev

# Copy application code
COPY . .

EXPOSE 4000

CMD ["node", "server.js"]
