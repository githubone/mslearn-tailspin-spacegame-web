az vm create --name MyLinuxAgen --resource-group learn-6b752ab0-0f28-4a0f-9b9e-3e91a2c781d1 --image Canonical:UbuntuServer:18.04-LTS:latest --location eastus --size Standard_DS2_v2 --admin-username azureuser --generate-ssh-keys

{- Finished ..
  "fqdns": "",
  "id": "/subscriptions/a1ccbcf7-9d27-4cf3-aeff-4dca9c6f4c01/resourceGroups/learn-6b752ab0-0f28-4a0f-9b9e-3e91a2c781d1/providers/Microsoft.Compute/virtualMachines/MyLinuxAgen",
  "location": "eastus",
  "macAddress": "00-22-48-1F-AA-0C",
  "powerState": "VM running",
  "privateIpAddress": "10.0.0.4",
  "publicIpAddress": "40.76.30.3",
  "resourceGroup": "learn-6b752ab0-0f28-4a0f-9b9e-3e91a2c781d1",
  "zones": ""
}



IPADDRESS=$(az vm show --name MyLinuxAgent --resource-group learn-6b752ab0-0f28-4a0f-9b9e-3e91a2c781d1 --show-details --query [publicIps] --output tsv)


export AZP_TOKEN=sqws43qovfug2zopgfe2dhnxhr23ugcjnzfsfboac6jp47hqoh7a


>cat build-agent.sh

azureuser@MyLinuxAgen:~$ cat build-agent.sh
#!/bin/bash
set -e

# Select a default agent version if one is not specified
if [ -z "$AZP_AGENT_VERSION" ]; then
  AZP_AGENT_VERSION=2.164.1
fi

# Verify Azure Pipelines token is set
if [ -z "$AZP_TOKEN" ]; then
  echo 1>&2 "error: missing AZP_TOKEN environment variable"
  exit 1
fi

# Verify Azure DevOps URL is set
if [ -z "$AZP_URL" ]; then
  echo 1>&2 "error: missing AZP_URL environment variable"
  exit 1
fi

# If a working directory was specified, create that directory
if [ -n "$AZP_WORK" ]; then
  mkdir -p "$AZP_WORK"
fi

# Create the Downloads directory under the user's home directory
if [ -n "$HOME/Downloads" ]; then
  mkdir -p "$HOME/Downloads"
fi

# Download the agent package
curl https://vstsagentpackage.azureedge.net/agent/$AZP_AGENT_VERSION/vsts-agent-linux-x64-$AZP_AGENT_VERSION.tar.gz> $HOME/Downloads/vsts-agent-linux-x64-$AZP_AGENT_VERSION.tar.gz

# Create the working directory for the agent service to run jobs under
if [ -n "$AZP_WORK" ]; then
  mkdir -p "$AZP_WORK"
fi

# Create a working directory to extract the agent packageto
mkdir -p $HOME/azp/agent

# Move to the working directory
cd $HOME/azp/agent

# Extract the agent package to the working directory
tar zxvf $HOME/Downloads/vsts-agent-linux-x64-$AZP_AGENT_VERSION.tar.gz

# Install the agent software
./bin/installdependencies.sh

# Configure the agent as the sudo (non-root) user
chown $SUDO_USER $HOME/azp/agent
sudo -u $SUDO_USER ./config.sh --unattended \
  --agent "${AZP_AGENT_NAME:-$(hostname)}" \
  --url "$AZP_URL" \
  --auth PAT \
  --token "$AZP_TOKEN" \
  --pool "${AZP_POOL:-Default}" \
  --work "${AZP_WORK:-_work}" \
  --replace \
  --acceptTeeEula

# Install and start the agent service
./svc.sh install
./svc.sh startx`

chmod u+x build-agent.sh

sudo -E ./build-agent.sh