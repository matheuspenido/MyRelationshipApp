FROM mcr.microsoft.com/dotnet/sdk:8.0 AS dev

ENV PROJECT_PATH=./MyRelationshipApp.API

WORKDIR /app

EXPOSE 8080
EXPOSE 8081

ENV ASPNETCORE_Kestrel__Certificates__Default__Path=/app/https/localhost.pfx
ENV ASPNETCORE_Kestrel__Certificates__Default__Password=myrelationshipapp

CMD dotnet watch run --project "$PROJECT_PATH" --urls "http://+:8080;https://+:8081" --no-hot-reload