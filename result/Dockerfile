FROM node:23.11.0-alpine

ENV NODE_ENV=production

WORKDIR /app

COPY *.json .

COPY . . 

RUN npm install

USER node

ENTRYPOINT ["node", "server.js"]