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
docker build -t my-relationship-app .
build the image, passing the executable project to env variable PROJECT_PATH

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