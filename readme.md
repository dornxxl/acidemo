# Azure Container Instance Demo

## Azure Container Registry

1. Create Container Registry

   *Azure CLI*

   ```sh
   az group create --resource-group acidemo-rg --location eastasia
   az acr create --name acrth --resource-group acidemo-rg --sku Basic --admin-enabled
   ```

2. List scope-map for assign to Token

   *Azure CLI*

   ```sh
   az acr scope-map list --registry acrth --query '[].{Name:name,Description:description}' \
   --output table
   ```

3. Create Token (User) for docker-cli, Please remember Password for future use.

   *Azure CLI*

   ```sh
   az acr token create --name docker --registry acrth --expiration-in-days 90 \
   --scope-map _repositories_push_metadata_write
   ```

4. (Local Computer) Use Docker CLI to login to ACR

   ```sh
   docker login acrth.azurecr.io -u docker -p <PASSWORD_FROM_ACR>
   ```

## How to create docker image and upload to ACR (On Local)

1. Create sample dotnet Project

   ```sh
   dotnet new blazorserver --name example --framework net7.0 --no-https
   ```

2. Create Dockerfile in project directory

   ```docker
   FROM mcr.microsoft.com/dotnet/aspnet:7.0 AS base
   WORKDIR /app
   EXPOSE 80

   ENV ASPNETCORE_URLS=http://+:80

   FROM mcr.microsoft.com/dotnet/sdk:7.0 AS build
   ARG configuration=Release
   WORKDIR /src
   COPY ["example.csproj", "./"]
   RUN dotnet restore "example.csproj"
   COPY . .
   WORKDIR "/src/."
   RUN dotnet build "example.csproj" -c $configuration -o /app/build

   FROM build AS publish
   ARG configuration=Release
   RUN dotnet publish "example.csproj" -c $configuration -o /app/publish /p:UseAppHost=false

   FROM base AS final
   WORKDIR /app
   COPY --from=publish /app/publish .
   ENTRYPOINT ["dotnet", "example.dll"]
   ```

3. (Optional) Create .dockerignore in project directory

   ```text
   **/.classpath
   **/.dockerignore
   **/.env
   **/.git
   **/.gitignore
   **/.project
   **/.settings
   **/.toolstarget
   **/.vs
   **/.vscode
   **/*.*proj.user
   **/*.dbmdl
   **/*.jfm
   **/azds.yaml
   **/bin
   **/charts
   **/docker-compose*
   **/Dockerfile*
   **/node_modules
   **/npm-debug.log
   **/obj
   **/secrets.dev.yaml
   **/values.dev.yaml
   LICENSE
   README.md
   ```

4. Build Docker image

   ```sh
   cd example
   docker build -t example -t example:0.1 .
   ```

5. Test run built image

   ```sh
   docker run --rm -p 80:80 example
   ```

   Open Webbrowser and test. after finished use Crtl+C to terminate docker

6. Push docker image to ACR

   ```sh
   docker tag example acrth.azurecr.io/example
   docker tag example:0.1 acrth.azurecr.io/example:0.1
   docker push acrth.azurecr.io/example
   docker push acrth.azurecr.io/example:0.1
   ```

7. Check repository on ACR

   ```sh
   az acr repository list --name acrth
   az acr repository show-tags --name acrth --repository example
   ```

## How to run simple ACI from ACR Image

Create and Run Contairner with ACR Image

   ```sh
   az container create --resource-group acidemo-rg --name acithdemo --image acrth.azurecr.io/example --cpu 1 --memory 1 --ports 80 --dns-name-label acithdemo --registry-login-server acrth.azurecr.io --registry-username <acr_user> --registry-password <acr_password>
   ```
