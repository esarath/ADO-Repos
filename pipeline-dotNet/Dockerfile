FROM mcr.microsoft.com/dotnet/sdk:6.0 AS build
WORKDIR /src
COPY . .
RUN dotnet publish pipelines-dotnet-core.csproj -c Release -o /app/publish

FROM mcr.microsoft.com/dotnet/aspnet:6.0 AS runtime
WORKDIR /app
COPY --from=build /app/publish .
RUN chmod -R g=u /app
EXPOSE 8080
ENV ASPNETCORE_URLS=http://+:8080
ENV HOME=/tmp
ENTRYPOINT ["dotnet", "pipelines-dotnet-core.dll"]
