FROM node:22-slim

WORKDIR /usr/src/app

# backend
COPY package*.json ./
RUN npm install

# frontend
COPY client ./client
RUN cd client && npm install && npm run build

# copia resto
COPY . .

EXPOSE 8080

CMD ["node", "server.js"]