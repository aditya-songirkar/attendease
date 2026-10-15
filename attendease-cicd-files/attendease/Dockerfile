FROM node:18-alpine

WORKDIR /app

# Install only production dependencies (better layer caching)
COPY package*.json ./
RUN npm ci --omit=dev

# Copy application source (server/ and public/)
COPY server ./server
COPY public ./public

ENV NODE_ENV=production
ENV PORT=3000
EXPOSE 3000

# Run as non-root user
USER node

CMD ["npm", "start"]
