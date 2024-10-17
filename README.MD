Documented steps to reproduce the creation of this project structure:

Create the folder structure:

MyRelationshipApp/
backend/
frontend/

===========================================

Create the backend app:
docker run --rm -it -v $(pwd)/backend:/app mcr.microsoft.com/dotnet/sdk:8.0 bash

inside the container, do:
cd app
dotnet new sln -n MyRelationshipApp
dotnet new webapi -n MyRelationshipApp.API
dotnet sln MyRelationshipApp.sln add MyRelationshipApp.API/MyRelationshipApp.API.csproj
exit

exit command will return to host.
change the files permission, to do that, go to the root folder and do:
sudo chown -R Matheus:Matheus ./backend

Verify if the aspnet runtime is installed
dotnet watch run

remove the extra sln file:
rm backend/MyRelationshipApp.API/MyRelationshipApp.API.sln

Create certificate:
openssl req -x509 -nodes -days 3650 -newkey rsa:2048 -keyout localhost.key -out localhost.crt -subj "/CN=localhost"

Create pfx from certificate and key:
openssl pkcs12 -export -out localhost.pfx -inkey localhost.key -in localhost.crt -passout pass:myrelationshipapp
put the pfx on certs folder, inside the root backend folder.

Add Dockerfile on backend folder:

FROM mcr.microsoft.com/dotnet/sdk:8.0 AS dev

ENV PROJECT_PATH=./MyRelationshipApp.API

WORKDIR /app

EXPOSE 8080
EXPOSE 8081

ENV ASPNETCORE_Kestrel__Certificates__Default__Path=/app/https/localhost.pfx
ENV ASPNETCORE_Kestrel__Certificates__Default__Password=myrelationshipapp

CMD dotnet watch run --project "$PROJECT_PATH" --urls "http://+:8080;https://+:8081" --no-hot-reload

build the image, passing the executable project to env variable PROJECT_PATH, inside backend folder do:
docker build --build-arg PROJECT_PATH=./MyRelationshipApp.API -t my-relationship-app .

Run the build image, listening from 8080 to 5000
inside backend folder, do:
docker run --rm -e PROJECT_PATH=./MyRelationshipApp.API -p 5000:8080 -p 5001:8081 -v $(pwd):/app -v $(pwd)/certs:/app/https:ro my-relationship-app

Open the url: http://localhost:5000/swagger and https://localhost:5001/swagger to verify if everything is right.

Try to change some code inside the program.cs to verify is the auto reaload will work, after the file was saved.

Create the docker-compose.dev.backend.yml file in docker folder (docker fold will be created inside the root folder)

services:
  backend:
    image: my-relationship-app-backend-dev
    build:
      context: ../backend
      dockerfile: Dockerfile.dev
    environment:
      - PROJECT_PATH=/app/MyRelationshipApp.API/
      - ASPNETCORE_ENVIRONMENT=Development
      - ASPNETCORE_URLS=https://+:8081;http://+:8080
      - ASPNETCORE_Kestrel__Certificates__Default__Path=/app/https/localhost.pfx
      - ASPNETCORE_Kestrel__Certificates__Default__Password=myrelationshipapp
    ports:
      - "5000:8080"
      - "5001:8081"
    volumes:
      - ../backend/:/app
      - ../backend/certs:/app/https:ro

after that: 

docker-compose -f docker-compose.backend.yml up --build
should start the app.
Test it again for https and https

===============================================
install the front-end project:

on the frontend folder, do:
docker run --rm -it -v $(pwd):/app node:20.11.1 bash

inside the container:
install angular:
npm install -g @angular/cli@17
npm install nodemon --save-dev

create the project:
ng new client --skip-git --directory=.
This command will create the angular inside the /app folder and the app folder is mapped to frontend in out host project.

To come back to host and leave the iterative image of node:
exit

change the files permission, to do that, go to the root folder and do:
sudo chown -R Matheus:Matheus ./frontend

create the dockerfile inside the frontend folder using nodemon to run and execute the changes and reload the project:

FROM node:20.11.1 AS dev

WORKDIR /app

COPY ./package.json ./package-lock.json ./

RUN npm install

COPY . .

EXPOSE 4200
EXPOSE 4201

ENV PATH="/app/node_modules/.bin:$PATH"
ENV HOST=0.0.0.0
ENV HTTP_PORT=4200
ENV HTTPS_PORT=4201
ENV SSL_CERT_PATH=/app/certs/localhost.crt
ENV SSL_KEY_PATH=/app/certs/localhost.key

CMD npx nodemon

create the nodemon inside the root folder for the frontend, call the file nodemon.js:
{
    "watch": ["src/", "angular.json", "package.json", "tsconfig.json"],
    "ext": "ts, html,css,scss,json",
    "execMap": {
        "json": "npm install"
    },
    "exec": "ng serve --host ${HOST} --port ${HTTP_PORT} & ng serve --host ${HOST} --port ${HTTPS_PORT} --ssl true --ssl-cert ${SSL_CERT_PATH} --ssl-key ${SSL_KEY_PATH}"
}

run the docker commands to test the project:

docker build -t client .
docker run --rm -it -v $(pwd):/app -p 4200:4200 -p 4201:4201 client

Open the url: http://localhost:4200 and https://localhost:4201 to verify if everything is right.

Try to change some code inside the index.html page to verify is the auto reload will work, after the file was saved.

Create the docker-compose.dev.frontend.yml file in docker folder (docker fold will be created inside the root folder):

services:
  frontend:
    image: my-relationship-app-client-dev
    build:
      context: ../frontend
      dockerfile: Dockerfile.dev
    environment:
      - NODE_ENV=development
      - HOST=0.0.0.0
    ports:
      - "4200:4200"
      - "4201:4201"
    volumes:
      - ../frontend:/app

========================================

Backend prod env:

Rename the previous Dockerfile inside backend folder to Dockerfile.dev
Create a second Dockerfile.prod file, next to the Dockerfile.dev with the following content:

#Stage 1: Build
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /source

# Argument to accept the project path (Looking for a way to be automated soon)
ARG PROJECT_PATH

# Copy the files
COPY . .

# Restore dependencies
RUN dotnet restore 

# Publish the project defined by PROJECT_PATH arg
RUN dotnet publish ${PROJECT_PATH} -c Release -o /app/publish

# Stage 2: Runtime
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS runtime
WORKDIR /app

# Copy the published files from the build stage
COPY --from=build /app/publish .

# Expose the default ports for .NET 8 (8080 for HTTP and 8443 for HTTPS)
EXPOSE 8080
EXPOSE 8443

ENV ASPNETCORE_URLS="https://+:8443;http://+:8080"
ENV ASPNETCORE_Kestrel__Certificates__Default__Path=/app/https/localhost.pfx
ENV ASPNETCORE_Kestrel__Certificates__Default__Password=myrelationshipapp

ENTRYPOINT ["sh", "-c", "dotnet /app/${PROJECT_PATH}.dll"]

To build this image:
docker build -f Dockerfile.prod --build-arg PROJECT_PATH=./MyRelationshipApp.API -t my-relationship-app .

To run this image:
docker run --rm -e PROJECT_PATH=./MyRelationshipApp.API -p 5000:8080 -p 5001:8443 -it my-relationship-app

Create the docker compose file docker-compose.prod.backend.yml:

services:
  backend:
    image: my-relationship-app-backend-prod
    build:
      context: ../backend
      dockerfile: Dockerfile.prod
      args:
        PROJECT_PATH: ./MyRelationshipApp.API
    environment:
      - PROJECT_PATH=./MyRelationshipApp.API
      - ASPNETCORE_ENVIRONMENT=Production
      - ASPNETCORE_URLS=https://+:8443;http://+:8080
      - ASPNETCORE_Kestrel__Certificates__Default__Path=/app/https/localhost.pfx
      - ASPNETCORE_Kestrel__Certificates__Default__Password=myrelationshipapp
    ports:
      - "5000:8080"
      - "5001:8443"
    volumes:
      - ../backend/certs:/app/https:ro

ps: the volumes for cert is essential to allow the developer update the certificate without copying it directly to the image.

==============================================================

Nginx to prod env for Angular:

Create the nginx folder inside the frontend folder
Create the nginx.conf file inside the nginx folder
Create the default.conf file inside the nginx folder

copy certs folder from the backend folder to frontend folder, we can keep the same
cert for a while.

The content for nginx.conf file:

user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log warn;
pid /var/run/nginx.pid;

events {
    worker_connections 1024;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for"';
    
    access_log /var/log/nginx/access.log main;

    sendfile on;
    keepalive_timeout 65;

    include /etc/nginx/conf.d/*.conf;
}

content for default.conf file:

server {
    listen 80;
    server_name my-relationship-app.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name my-relationship-app.com;

    # SSL Configuration
    ssl_certificate /etc/nginx/ssl/localhost.crt;
    ssl_certificate_key /etc/nginx/ssl/localhost.key;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;
    ssl_ciphers HIGH:!aNULL:!MD5;

    root /usr/share/nginx/html;

    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }

    gzip on;
    gzip_types text/plain application/javascript application/x-javascript text/javascript text/css application/xml text/xml application/json image/svg+xml;
    gzip_proxied any;
    gzip_vary on;
}

The Dockerfile.prod for the frontend app:

# Stage 1: Build Angular Application
FROM node:20.11.1 AS build
WORKDIR /app

# Copy package.json and package-lock.json to install dependencies
COPY ./package.json ./package-lock.json ./
RUN npm install

# Copy the rest of the application code
COPY . .

# Build the Angular app for production
RUN npm run build --prod

# Stage 2: Serve with Nginx
FROM nginx:alpine AS prod-stage

ARG ANGULAR_PROJECT_NAME

# Copy the built Angular app from the build stage
COPY --from=build /app/dist/${ANGULAR_PROJECT_NAME}/browser /usr/share/nginx/html

# Copy custom Nginx configuration
COPY ./nginx/nginx.conf /etc/nginx/nginx.conf
COPY ./nginx/default.conf /etc/nginx/conf.d/default.conf

# Copy SSL certificates
COPY certs/localhost.key /etc/nginx/ssl/localhost.key
COPY certs/localhost.crt /etc/nginx/ssl/localhost.crt

# Expose port 80 for HTTP and HTTPS traffic
EXPOSE 80
EXPOSE 443

# Start Nginx server
CMD ["nginx", "-g", "daemon off;"]

to build this image:

docker build -f Dockerfile.prod --build-arg ANGULAR_PROJECT_NAME=client -t my-angular-app-test .

to run this image:

docker run --rm -p 80:80 -p 443:443 -e ANGULAR_PROJECT_NAME=client my-angular-app-test 

Create the docker-compose.prod.frontend.yml inside root/docker:

services:
  frontend:
    image: my-relationship-app-client-prod
    build:
      context: ../frontend
      dockerfile: Dockerfile.prod
      args:
        ANGULAR_PROJECT_NAME: client
    ports:
      - "80:80"
      - "443:443"

To run this docker compose file, go to docker folder and do:

docker-compose -f docker-compose.dev.frontend.yml up --build

The app should work for http://localhost:4200 and https://localhost:4201 and display angular information








inside the backend folder:

docker build -f Dockerfile.dev --build-arg PROJECT_NAME=MyRelationshipApp.API --build-arg ASPNETCORE_ENVIRONMENT=Development --build-arg ASPNETCORE_URLS="http://+:8080;https://+:8081" --build-arg CERT_PATH=https/localhost.pfx --build-arg CERT_KEY=myrelationshipapp -t my-relationship-app-backend-dev .

docker run --rm -v $(pwd):/app -v $(pwd)/certs:/app/https -p 5000:8080 -p 5001:8081 -it my-relationship-app-backend-dev

docker-compose -f ./docker/docker-compose.dev.backend.yml --env-file .env.dev up --build

inside backend folder:

docker build -f Dockerfile.prod --build-arg PROJECT_NAME=MyRelationshipApp.API --build-arg ASPNETCORE_ENVIRONMENT=Production --build-arg ASPNETCORE_URLS="http://+:8080;https://+:8443" --build-arg CERT_PATH=certs/localhost.pfx --build-arg CERT_KEY=myrelationshipapp -t my-relationship-app-backend-prod .

docker run --rm -it -v $(pwd)/certs:/app/https -p 5000:8080 -p 5001:8443  my-relationship-app-backend-prod


To build prod for frontend with .env.prod file:
The same structure can be created for other envs but dev, ok?

dockebuild -f ./frontend/Dockerfile.prod --build-arg ANGULAR_PROJECT_NAME=client -t test-angular-prod-envvariables ./frontend

docker run --rm --env-file .env.prod -v $(pwd)/frontend/certs/:/etc/nginx/ssl -v $(pwd)/frontend/nginx/nginx.conf:/etc/nginx/nginx.conf -v $(pwd)/frontend/nginx/default.conf.template:/etc/nginx/conf.d/default.conf.template -p 80:80 -p 443:443 test-angular-prod-envvariables






=======================================
on the root folder:

To create NGINX RP for backend:
docker build -f ./backend-server/Dockerfile.prod ./backend-server -t my-backend-server

To create the API container:
docker build -f ./backend/Dockerfile.prod --build-arg PROJECT_NAME=MyRelationshipApp.API -t my-backend-api ./backend

To create the network to allow communication between the backend and RP server:
docker network create my-network

To run the API (addressed by backend-api)on my-network:
docker run -d --name backend-api --network my-network -e PROJECT_NAME=MyRelationshipApp.API -e ASPNETCORE_ENVIRONMENT=Production -e ASPNETCORE_URLS="http://+:8080" my-backend-api

To run the server on my-network
docker run -d --name backend-server --network my-network -v $(pwd)/backend-server/certs:/etc/nginx/ssl/ -v $(pwd)/backend-server/nginx:/etc/nginx/conf.d/ -p 443:443 -p 80:80 -e BACKEND_DOMAIN=localhost -e BACKEND_API_URL=backend-api:8080 -e SSL_CERT_FILE=localhost.crt -e SSL_KEY_FILE=localhost.key my-backend-server