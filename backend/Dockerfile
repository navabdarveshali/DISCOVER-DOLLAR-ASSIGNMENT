FROM node:22.2.0 AS base

WORKDIR /app

COPY package.json ./

RUN npm install 

COPY . .


FROM node:22.2.0 AS final

WORKDIR /app

COPY --from=base /app/node_modules ./node_modules

COPY --from=base /app .

EXPOSE 8080

CMD ["node", "server.js"]