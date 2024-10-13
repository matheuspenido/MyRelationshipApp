Documented steps to reproduce the creation of this project structure:

folders:

MyRelationshipApp/
backend/
frontend/
Create the app:
docker run -it -v $(pwd)/backend:/app mcr.microsoft.com/dotnet/sdk:8.0 bash
inside the container:
cd app
dotnet new sln -n MyRelationshipApp
dotnet new webapi -n MyRelationshipApp.API
dotnet sln MyRelationshipApp.sln add MyRelationshipApp.API/MyRelationshipApp.API.csproj
exit -> back to host

cd ..
sudo chown -R Matheus:Matheus .

Verify if the aspnet runtime is installed
dotnet watch run

remove the extra sln file:
rm backend/MyRelationshipApp.API/MyRelationshipApp.API.sln

Create certificate:
openssl req -x509 -nodes -days 3650 -newkey rsa:2048 -keyout localhost.key -out localhost.crt -subj "/CN=localhost"

Create pfx from certificate and key:
openssl pkcs12 -export -out localhost.pfx -inkey localhost.key -in localhost.crt -passout pass:myrelationshipapp
put the pfx on certs folder, inside the root folder

Add Dockerfile on backend folder:

FROM mcr.microsoft.com/dotnet/sdk:8.0 AS dev
ENV PROJECT_PATH=./MyRelationshipApp.API
WORKDIR /app
EXPOSE 8080
EXPOSE 8081
ENV ASPNETCORE_Kestrel__Certificates__Default__Path=/app/https/localhost.pfx
ENV ASPNETCORE_Kestrel__Certificates__Default__Password=myrelationshipapp
CMD dotnet watch run --project "$PROJECT_PATH" --urls "http://+:8080;https://+:8081" --no-hot-reload

build the image, passing the executable project to env variable PROJECT_PATH
docker build --build-arg PROJECT_PATH=./MyRelationshipApp.API -t my-relationship-app .

Run the build image, listening from 8080 to 5000
inside backend folder:
docker run -e PROJECT_PATH=./MyRelationshipApp.API -p 5000:8080 -p 5001:8081 -v $(pwd):/app -v $(pwd)/certs:/app/https:ro my-relationship-app

Open the url: http://localhost:5000/swagger
Create the dockercompose file in docker folder (inside the root folder)
services:
  backend:
    image: my-relationship-app
    build:
      context: ../backend
      dockerfile: Dockerfile
    environment:
      - PROJECT_PATH=./backend/MyRelationshipApp/API
      - ASPNETCORE_ENVIRONMENT=Development
      - ASPNETCORE_URLS=https://+:8081;http://+:8080
      - ASPNETCORE_Kestrel__Certificates__Default__Path=/app/https/localhost.pfx
      - ASPNETCORE_Kestrel__Certificates__Default__Password=myrelationshipapp
    ports:
      - "5000:8080"
      - "5001:8081"
    volumes:
      - ../backend/:/app
      - ../backend/certs:/https:ro

after that: 

docker-compose -f docker-compose.backend.yml up --build
should start the app.

install the front-end project:

on the root folder of the project:
docker run -it -v $(pwd)/frontend:/app node:20.11.1 bash

inside the container:
install angular:
npm install -g @angular/cli@17
npm install nodemon --save-dev

create the project:
ng new client --skip-git --directory=.
This command will create the angular inside the /app folder and the app folder is mapped to frontend in out host project.

To come back to host and leave the iterative image of node:
exit

change the files permission:
cd ..
sudo chown -R Matheus:Matheus ./frontend

create the nodemon inside the root folder for the frontend, call the file nodemon.js:
{
    "watch": ["src/", "angular.json", "package.json", "tsconfig.json"],
    "ext": "ts, html,css,scss,json",
    "execMap": {
        "json": "npm install"
    },
    "exec": "ng serve --host ${HOST}"
}

create the dockerfile inside the frontend folder using nodemon to run and execute the changes and reload the project:

FROM node:20.11.1 AS dev

WORKDIR /app

COPY ./package.json ./package-lock.json ./

RUN npm install

COPY . .

EXPOSE 4200

ENV PATH="/app/node_modules/.bin:$PATH"
ENV HOST=0.0.0.0

CMD npx nodemon

run the docker commands to test the project:

docker build -t client .
docker run -it -v $(pwd):/app -p 4200:4200 client

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
docker run -e PROJECT_PATH=./MyRelationshipApp.API -p 5000:8080 -p 5001:8443 -it my-relationship-app

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

docker run --rm -p 4280:80 -p 443:443 -e ANGULAR_PROJECT_NAME=client my-angular-app-test 

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