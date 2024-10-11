FROM node:20.11.1 AS dev

WORKDIR /app

COPY ./package.json ./package-lock.json ./

RUN npm install

COPY . .

EXPOSE 4200

ENV PATH="/app/node_modules/.bin:$PATH"
ENV HOST=0.0.0.0

CMD npx nodemon