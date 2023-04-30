# Portainer 
Provides GUI for docker containers to easily manage.
## Installation
1. Update and Upgrade:
    ```
    sudo apt update
    sudo apt upgrade
    ```
2. Install Portainer
    ```
    sudo docker pull portainer/portainer-ce:latest
    ```
3. Run Portainer
    ```
    sudo docker run -d -p 9000:9000 --name=portainer --restart=always -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data portainer/portainer-ce:latest
    ```
A few of the big things we do here is first define the ports we want Portainer to have access to. In our case, this will be port 9000. 
We assign this docker container the name “portainer” so we can quickly identify it if we ever needed.
Additionally, we also tell the Docker manager that we want it to restart this Docker if it is ever unintentionally offline.
## Testing
You can now access the WebUI by typing `[PIIPADDRESS]:9000` into your search bar. Follow the link under Sources to learn how to use Portainer.
## Updating
1. Stop Portainer
    ```
    docker stop portainer
    ```
2. Remove Portainer
    ```
    docker rm portainer
    ```
3. Repeat steps 2-3 under `Installation`.
## Sources
* https://pimylifeup.com/raspberry-pi-portainer/