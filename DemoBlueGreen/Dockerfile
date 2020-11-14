FROM mcr.microsoft.com/dotnet/core/sdk:3.1-buster AS build-env
WORKDIR /src

# Copy csproj and restore as distinct layers
COPY *.sln ./
COPY DemoBlueGreen/DemoBlueGreen.csproj DemoBlueGreen/
RUN dotnet restore

# Copy everything else and build
COPY . .
WORKDIR "DemoBlueGreen"
#RUN mv appsettings.Development.json appsettings.json
RUN dotnet publish "DemoBlueGreen.csproj" -c Release -o /app

# Build runtime image
FROM mcr.microsoft.com/dotnet/core/aspnet:3.1-buster-slim
WORKDIR /app
COPY --from=build-env /app .
ENTRYPOINT ["dotnet", "DemoBlueGreen.dll"]

EXPOSE 80
