FROM node:18-alpine AS base

WORKDIR /app

COPY package*.json ./
RUN apk add --no-cache python3 make g++
RUN npm install --production

COPY . .

ENV NODE_ENV=production
EXPOSE 3001

CMD ["npm", "run", "start"]
