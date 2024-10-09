FROM mcr.microsoft.com/dotnet/sdk:8.0 AS dev

ARG PROJECT_PATH

WORKDIR /app

EXPOSE 8080

CMD dotnet watch run --project "$PROJECT_PATH" --urls "http://+:8080" --no-hot-reload