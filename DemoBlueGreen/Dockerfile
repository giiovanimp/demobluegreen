FROM mcr.microsoft.com/dotnet/core/sdk:3.1-buster AS build-env
WORKDIR /src

COPY *.sln ./
COPY DemoBlueGreen/DemoBlueGreen.csproj DemoBlueGreen/
RUN dotnet restore

COPY . .
WORKDIR "DemoBlueGreen"
RUN dotnet publish "DemoBlueGreen.csproj" -c Release -o /app

FROM mcr.microsoft.com/dotnet/core/aspnet:3.1-buster-slim
WORKDIR /app
COPY --from=build-env /app .
ENTRYPOINT ["dotnet", "DemoBlueGreen.dll"]

EXPOSE 80
