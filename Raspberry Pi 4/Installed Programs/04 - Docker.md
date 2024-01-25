# Docker 
Containerize certain programs for easy removal. Finding guides for docker container programs are harder than finding guides to normally install programs. The benefit for docker is being able to quickly and easily remove the entire program, which is much harder than normally installed programs. Installing docker containers can also be easier as there is a less chance of it conflicting with other programs.
## Installation
1. Update and Upgrade:
    ```
    sudo apt-get update && sudo apt-get upgrade
    ```
2. Install Docker:
    ```
    curl -sSL https://get.docker.com | sh
    ```
3. Add a Non-Root User to the Docker Group:
    ```
    sudo usermod -aG docker pi
    ```
4. Then add permissions to the current user:
    ```
    sudo usermod -aG docker ${USER}
    ```
5. Check it running with:
    ```
    groups ${USER}
    ```
6. Reboot:
    ```
    sudo reboot
    ```
7. Install Docker-Compose:
    ```
    sudo apt-get install docker-compose
    ```
8. Enable the Docker system service to start your containers on boot:
    ```
    sudo systemctl enable docker
    ```
## Testing
Test by running the Hello World container:
```
docker run hello-world
```
## Sources
* https://dev.to/elalemanyo/how-to-install-docker-and-docker-compose-on-raspberry-pi-1mo
