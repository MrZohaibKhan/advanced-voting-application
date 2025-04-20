FROM  mcr.microsoft.com/dotnet/sdk:7.0.409-bookworm-slim@sha256:08009349c7f0ad79898f5de5bcf70a3bddcca82dcf736052bd9c02066f042200 AS build

WORKDIR /app

COPY Worker.csproj .

RUN dotnet restore

COPY . .

RUN dotnet publish -c Release -o out 

FROM  mcr.microsoft.com/dotnet/runtime:7.0

ARG UID=10001
RUN adduser \
    --disabled-password \
    --gecos "" \
    --home "/nonexistent" \
    --shell "/sbin/nologin" \
    --no-create-home \
    --uid "${UID}" \
    dotnetuser

WORKDIR /app

COPY --from=build /app/out .

USER dotnetuser

ENTRYPOINT ["dotnet", "Worker.dll"]