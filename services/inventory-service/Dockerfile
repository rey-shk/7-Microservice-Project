FROM node:20-alpine

WORKDIR /app

ENV NODE_ENV=production

# Install dependencies
COPY package*.json ./
RUN npm ci --omit=dev

# Copy application source
COPY . .

# Run as non-root user
USER node

EXPOSE 8083

CMD ["node", "src/index.js"]
