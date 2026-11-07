FROM node:24-alpine

WORKDIR /app

# no npm install: stats has no external packages
COPY package.json server.js ./

EXPOSE 3000

CMD ["node", "server.js"]
