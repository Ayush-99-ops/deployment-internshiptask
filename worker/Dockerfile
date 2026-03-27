# --- Build Stage ---
FROM mcr.microsoft.com/dotnet/sdk:7.0 AS build
WORKDIR /source

# Copy project file and restore dependencies
COPY *.csproj ./
RUN dotnet restore

# Copy source code and publish
COPY . .
RUN dotnet publish -c Release -o /app --no-restore

# --- Runtime Stage ---
FROM mcr.microsoft.com/dotnet/runtime:7.0 AS runtime
WORKDIR /app

COPY --from=build /app ./

ENTRYPOINT ["dotnet", "Worker.dll"]
