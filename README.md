# Azure DevOps Self-Hosted Agent
[Reference](https://docs.microsoft.com/en-us/azure/devops/pipelines/agents/docker?view=azure-devops)

## Environment variables

| Environment variable | Description                                                  |
| :------------------- | :----------------------------------------------------------- |
| AZP_URL              | The URL of the Azure DevOps or Azure DevOps Server instance. (https://dev.azure.com/yourOrg) |
| AZP_TOKEN            | [Personal Access Token (PAT)](https://docs.microsoft.com/en-us/azure/devops/organizations/accounts/use-personal-access-tokens-to-authenticate?view=azure-devops) with **Agent Pools (read, manage)** scope, created by a user who has permission to [configure agents](https://docs.microsoft.com/en-us/azure/devops/pipelines/agents/pools-queues?view=azure-devops#creating-agent-pools), at `AZP_URL`. (https://dev.azure.com/yourOrg/_usersSettings/tokens) |
| AZP_AGENT_NAME       | Agent name (default value: the container hostname).          |
| AZP_POOL             | Agent pool name (default value: `Default`). (https://dev.azure.com/yourOrg/projectName/_settings/agentqueues) |
| AZP_WORK             | Work directory (default value: `_work`).                     |

## Dockerfile

1. Save the following content to `~/Dockerfile`:

   ```dockerfile
   FROM ubuntu:18.04
   
   # To make it easier for build and release pipelines to run apt-get,
   # configure apt to not require confirmation (assume the -y argument by default)
   ENV DEBIAN_FRONTEND=noninteractive
   RUN echo "APT::Get::Assume-Yes \"true\";" > /etc/apt/apt.conf.d/90assumeyes
   
   RUN apt-get update && apt-get install -y --no-install-recommends \
       ca-certificates \
       curl \
       jq \
       git \
       iputils-ping \
       libcurl4 \
       libicu60 \
       libunwind8 \
       netcat \
       libssl1.0 \
       zip \
       unzip \
     && rm -rf /var/lib/apt/lists/*
   
   RUN curl -LsS https://aka.ms/InstallAzureCLIDeb | bash \
     && rm -rf /var/lib/apt/lists/*
   
   ARG TARGETARCH=amd64
   ARG AGENT_VERSION=2.185.1
   
   WORKDIR /azp
   RUN if [ "$TARGETARCH" = "amd64" ]; then \
         AZP_AGENTPACKAGE_URL=https://vstsagentpackage.azureedge.net/agent/${AGENT_VERSION}/vsts-agent-linux-x64-${AGENT_VERSION}.tar.gz; \
       else \
         AZP_AGENTPACKAGE_URL=https://vstsagentpackage.azureedge.net/agent/${AGENT_VERSION}/vsts-agent-linux-${TARGETARCH}-${AGENT_VERSION}.tar.gz; \
       fi; \
       curl -LsS "$AZP_AGENTPACKAGE_URL" | tar -xz
   
   COPY ./start.sh .
   RUN chmod +x start.sh
   
   ENTRYPOINT [ "./start.sh" ]
   ```

2. Save the following content to `~/start.sh`, making sure to use Unix-style (LF) line endings:

   ```shell
   #!/bin/bash
   set -e
   
   if [ -z "$AZP_URL" ]; then
     echo 1>&2 "error: missing AZP_URL environment variable"
     exit 1
   fi
   
   if [ -z "$AZP_TOKEN_FILE" ]; then
     if [ -z "$AZP_TOKEN" ]; then
       echo 1>&2 "error: missing AZP_TOKEN environment variable"
       exit 1
     fi
   
     AZP_TOKEN_FILE=/azp/.token
     echo -n $AZP_TOKEN > "$AZP_TOKEN_FILE"
   fi
   
   unset AZP_TOKEN
   
   if [ -n "$AZP_WORK" ]; then
     mkdir -p "$AZP_WORK"
   fi
   
   export AGENT_ALLOW_RUNASROOT="1"
   
   cleanup() {
     if [ -e config.sh ]; then
       print_header "Cleanup. Removing Azure Pipelines agent..."
   
       # If the agent has some running jobs, the configuration removal process will fail.
       # So, give it some time to finish the job.
       while true; do
         ./config.sh remove --unattended --auth PAT --token $(cat "$AZP_TOKEN_FILE") && break
   
         echo "Retrying in 30 seconds..."
         sleep 30
       done
     fi
   }
   
   print_header() {
     lightcyan='\033[1;36m'
     nocolor='\033[0m'
     echo -e "${lightcyan}$1${nocolor}"
   }
   
   # Let the agent ignore the token env variables
   export VSO_AGENT_IGNORE=AZP_TOKEN,AZP_TOKEN_FILE
   
   source ./env.sh
   
   print_header "1. Configuring Azure Pipelines agent..."
   
   ./config.sh --unattended \
     --agent "${AZP_AGENT_NAME:-$(hostname)}" \
     --url "$AZP_URL" \
     --auth PAT \
     --token $(cat "$AZP_TOKEN_FILE") \
     --pool "${AZP_POOL:-Default}" \
     --work "${AZP_WORK:-_work}" \
     --replace \
     --acceptTeeEula & wait $!
   
   print_header "2. Running Azure Pipelines agent..."
   
   trap 'cleanup; exit 0' EXIT
   trap 'cleanup; exit 130' INT
   trap 'cleanup; exit 143' TERM
   
   # To be aware of TERM and INT signals call run.sh
   # Running it with the --once flag at the end will shut down the agent after the build is executed
   ./run.sh "$@"
   ```

## Docker Compose

1. Save the following content to `~/docker-compose.yml`:

   ```yaml
   version: '3.7'
   services:
     azdevops-agent:
       build: .
       image: azdevops-agent
       container_name: azdevops-agent
       environment:
         - AZP_URL=https://dev.azure.com/{your-organization}
           AZP_TOKEN={pat-token}
           AZP_POOL={agent-pool}
       volumes:
       - /var/run/docker.sock:/var/run/docker.sock #Docker within a Docker container
       restart: always
   ```

2. Build and run with:

   ```
   docker-compose up -d --build
   ```

## Kubernetes - AKS

1. Build image:

   ```shell
   docker build -t azdevops-agent:latest .
   docker image tag azdevops-agent <acr-server>/azdevops-agent
   ```
2. Create the secrets on the AKS cluster.

   ```shell
   # Azure DevOps pat user needs Manage permissions for the agent pool
   kubectl create secret generic azdevops-agent \
     --from-literal=AZP_URL=https://dev.azure.com/yourOrg \
     --from-literal=AZP_TOKEN=YourPAT \
     --from-literal=AZP_POOL=NameOfYourPool
   ```

2. Run this command to push your container to Container Registry:

   ```shell
   docker login <acr-server> --username <acr-username> --password-stdin
   docker push <acr-server>/azdevops-agent:latest
   ```

3. Configure Container Registry integration for existing AKS clusters. 

   ```shell
   az aks update -n myAKSCluster -g myResourceGroup --attach-acr <acr-name>
   ```

4. Save the following content to `~/deployment.yml`:

   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: azdevops-agent
     labels:
       app: azdevops-agent
   spec:
     replicas: 1 #here is the configuration for the actual agent always running
     selector:
       matchLabels:
         app: azdevops-agent
     template:
       metadata:
         labels:
           app: azdevops-agent
       spec:
         containers:
         - name: azdevops-agent
           image: {your-acr-server}.azurecr.io/azdevops-agent
           imagePullPolicy: Always
           env:
             - name: AZP_URL
               valueFrom:
                 secretKeyRef:
                   name: azdevops-agent
                   key: AZP_URL
             - name: AZP_TOKEN
               valueFrom:
                 secretKeyRef:
                   name: azdevops-agent
                   key: AZP_TOKEN
             - name: AZP_POOL
               valueFrom:
                 secretKeyRef:
                   name: azdevops-agent
                   key: AZP_POOL
           volumeMounts:
           - mountPath: /var/run/docker.sock #Docker within a Docker container
             name: docker-volume
         volumes:
         - name: docker-volume
           hostPath:
             path: /var/run/docker.sock
   ```

   This Kubernetes YAML creates a replica set and a deployment, where `replicas: 1` indicates the number or the agents that are running on the cluster.

5. Run this command:

   ```shell
   kubectl apply -f deployment.yml
   ```

Now your agents will run the AKS cluster.
