FROM node:20-alpine

WORKDIR /app

COPY app/package*.json ./
RUN npm ci --omit=dev

COPY app/ ./

RUN chown -R node:node /app

USER root

EXPOSE 3000

ENV NODE_ENV=production

CMD ["sh", "-c", "echo 'no server on 3000' && sleep 3600"]

