

# How to setup [docker-minecraft-server](https://github.com/itzg/docker-minecraft-server) with plugins and mods in docker-compose.yml file.

## What's the purpose of the tutorial and why did I create it?

I had a HUGE problem with setting up everything that I needed in one file. I'm not going to explain the easy part like: how to turn off online mode or how to set-up game version but the things I had problem with becasue there's already a documentation that explains it very well and where to put it in the file. Unfortunately it doesn't explain very well how to do everything in a `docker-compose.yml`.
## How to start and what do I need?

### You need to install docker.
Since I was using linux I'll be reffering only for the linux. I did eventually launch it on windows machine but I just copied the docker-compose.yml with installed [Docker Desktop](https://docs.docker.com/desktop/install/windows-install/) and used `docker compose up` to launch it.

To install it on linux machine follow the [official](https://docs.docker.com/engine/install/ubuntu/#install-using-the-repository) manual guide on how to install.
\
For lazy people here's a code snippet for Ubuntu distro:
1. Setup Docker ```apt``` repository.
```
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
```
>If you use an Ubuntu derivative distro, such as Linux Mint, you may need to use ```UBUNTU_CODENAME``` instead of ```VERSION_CODENAME```.
2. Install the Docker packages.
```
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```
3. Verify that the Docker Engine installation is successful by running the hello-world image.
```
sudo docker run hello-world
```

This command downloads a test image and runs it in a container. When the container runs, it prints a confirmation message and exits.
\
For any errors and problems please visit [Docker Installation Guide](https://docs.docker.com/engine/install/ubuntu/#install-using-the-repository)

# Create and launch the server.
>Besides the most basic server everything in this tutorial will be based on basic server + lazytainer
### Most basic server.

1. Create a new directory
2. Put the contents of the file below in a file called docker-compose.yml
3. Run `docker compose up -d` in that directory
4. Done! Point your client at your host's name/IP (the main PC you are using) address and port 25565.
```yml
services:
  mc:
    image: itzg/minecraft-server
    tty: true
    stdin_open: true
    ports:
      - "25565:25565"
    environment:
      EULA: "TRUE"
    volumes:
      # attach the relative directory 'data' to the container's /data path
      - ./data:/data
```
To apply changes made to the compose file, just run `docker compose up -d` again.
\
\
Follow the logs of the container using `docker compose logs -f`, check on the status with `docker compose ps`, and stop the container using `docker compose stop`.

 

### Basic server with lazytainer.

To get it up and running edit the `docker-compose.yml` file and add on the beggining:
```yml
services:
  lazytainer:
    image: ghcr.io/vmorganp/lazytainer:master
    environment:
      VERBOSE: false
    ports:
      - 25565:25565
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
    labels:
      - lazytainer.group.minecraft.sleepMethod=pause
      - lazytainer.group.minecraft.ports=25565
      - lazytainer.group.minecraft.minPacketThreshold=2 # Start after two incomming packets
      - lazytainer.group.minecraft.inactiveTimeout=600 # 10 minutes, to allow the server to bootstrap. You can probably make this lower later if you want.
    restart: unless-stopped
    network_mode: bridge
  mc:
    image: itzg/minecraft-server
    tty: true
    stdin_open: true
    restart: unless-stopped
    environment:
      EULA: TRUE
    volumes:
      - ./data:/data
    labels:
      - lazytainer.group=minecraft
    depends_on:
      - lazytainer
    network_mode: service:lazytainer
```
>### Disclaimer
>Why lazytainer and not lazymc? No idea it was my first option that I used because I got scared of creating my own proxy.  I'm propably not using it correcly thought since the documentation says to use `sleepMethod=stop` instead of `pause`.
 
### Basic server with lazytainer and whitelist

```yml
services:
  lazytainer:
    image: ghcr.io/vmorganp/lazytainer:master
    environment:
      VERBOSE: false
    ports:
      - 25565:25565
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
    labels:
      - lazytainer.group.minecraft.sleepMethod=pause
      - lazytainer.group.minecraft.ports=25565
      - lazytainer.group.minecraft.minPacketThreshold=2 # Start after two incomming packets
      - lazytainer.group.minecraft.inactiveTimeout=600 # 10 minutes, to allow the server to bootstrap. You can probably make this lower later if you want.
    restart: unless-stopped
    network_mode: bridge
  mc:
    image: itzg/minecraft-server
    tty: true
    stdin_open: true
    restart: unless-stopped
    environment:
      EULA: TRUE
      HIDE_ONLINE_PLAYERS: "true"
      ENABLE_WHITELIST: "true"
      EXISTING_WHITELIST_FILE: SYNC_FILE_MERGE_LIST # ensures that the file inside /data is always up to date with whitelist.json from docker-compose.yml
    volumes:
      - ./data:/data
      - ./whitelist.json:/data/whitelist.json
    labels:
      - lazytainer.group=minecraft
    depends_on:
      - lazytainer
    network_mode: service:lazytainer
```
This is how typical `whitelist.json` looks like:
```json
[
  {
    "uuid": "playeruuid",
    "name": "playername"
  }
]
```
>Remember to populate the `whitelist.json` with correct user `uuid` and `name`

> To enforce the whitelist changes immediately when whitelist commands are used , set `ENFORCE_WHITELIST` to "true" in `environment` similar to `HIDE_ONLINE_PLAYERS`.

Place the `whitelist.json` in the same folder as `docker-compose.yml` and run.
\
If you want to add someone on the whitelist you can either use a command from server console or while being on the server.
> If you plan on using `online_mode: false` then then you have to manually enter the `uuid` in the `whitelist.json`
> #### How to get `uuid` of the player?
> You can either use online tools to get `uuid` based on the player's name OR let them join the server, copy player's `id` from server's information of player trying to connect and put it in the `whitelist.json` file.


### Spigot with lazytainer,whitelist and plugins
[Auto-download using Spiget](https://docker-minecraft-server.readthedocs.io/en/latest/mods-and-plugins/spiget/) is explained well in the docs. Unfortunately it's not well explained on how to achieve it with already downloaded plugins so I'll show an example that should work. Unfortunately I didn't test that on my server since I wanted the plugins to be always up to date.
\
\
This will be used in the example of Magma
```yml
services:
  lazytainer:
    image: ghcr.io/vmorganp/lazytainer:master
    environment:
      VERBOSE: false
    ports:
      - 25565:25565
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
    labels:
      - lazytainer.group.minecraft.sleepMethod=pause
      - lazytainer.group.minecraft.ports=25565
      - lazytainer.group.minecraft.minPacketThreshold=2 # Start after two incomming packets
      - lazytainer.group.minecraft.inactiveTimeout=600 # 10 minutes, to allow the server to bootstrap. You can probably make this lower later if you want.
    restart: unless-stopped
    network_mode: bridge
  mc:
    image: itzg/minecraft-server
    tty: true
    stdin_open: true
    restart: unless-stopped
    environment:
      TYPE: SPIGOT
      SPIGET_RESOURCES: 28140,19362 # this'll download LoginSecurity and LuckyPerms
      HIDE_ONLINE_PLAYERS: "true"
      EULA: "TRUE"
      ENABLE_WHITELIST: "true"
      EXISTING_WHITELIST_FILE: SYNC_FILE_MERGE_LIST
    volumes:
      - ./data:/data
      - ./whitelist.json:/data/whitelist.json
    labels:
      - lazytainer.group=minecraft
    depends_on:
      - lazytainer
    network_mode: service:lazytainer
```
> This is an automatic download of the mods. To see manual installation with mounted option see below

```yml
services:
  lazytainer:
    image: ghcr.io/vmorganp/lazytainer:master
    environment:
      VERBOSE: false
    ports:
      - 25565:25565
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
    labels:
      - lazytainer.group.minecraft.sleepMethod=pause
      - lazytainer.group.minecraft.ports=25565
      - lazytainer.group.minecraft.minPacketThreshold=2 # Start after two incomming packets
      - lazytainer.group.minecraft.inactiveTimeout=600 # 10 minutes, to allow the server to bootstrap. You can probably make this lower later if you want.
    restart: unless-stopped
    network_mode: bridge
  mc:
    image: itzg/minecraft-server
    tty: true
    stdin_open: true
    restart: unless-stopped
    environment:
      TYPE: SPIGOT
      PLUGINS: ./plugins
      HIDE_ONLINE_PLAYERS: "true"
      EULA: "TRUE"
      ENABLE_WHITELIST: "true"
      EXISTING_WHITELIST_FILE: SYNC_FILE_MERGE_LIST
    volumes:
      - ./data:/data
      - ./whitelist.json:/data/whitelist.json
      - ./plugins:/data/plugins
    labels:
      - lazytainer.group=minecraft
    depends_on:
      - lazytainer
    network_mode: service:lazytainer
```
> This function is not tested as I said above. I didn't need the manual installation.
>
> Folder `/plugins` with plugins is required in the same path as `docker-compose.yml`

### Magma with lazytainer,whitelist,plugins and mods
>#### Only already downloaded mods will be used here since auto-download mods is well documented in the [docs](https://docker-minecraft-server.readthedocs.io/en/latest/mods-and-plugins/#modsplugins-list).
```yml
services:
  lazytainer:
    image: ghcr.io/vmorganp/lazytainer:master
    environment:
      VERBOSE: false
    ports:
      - 25565:25565
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
    labels:
      - lazytainer.group.minecraft.sleepMethod=pause
      - lazytainer.group.minecraft.ports=25565
      - lazytainer.group.minecraft.minPacketThreshold=2 # Start after two incomming packets
      - lazytainer.group.minecraft.inactiveTimeout=600 # 10 minutes, to allow the server to bootstrap. You can probably make this lower later if you want.
    restart: unless-stopped
    network_mode: bridge
  mc:
    image: itzg/minecraft-server:java8
    tty: true
    stdin_open: true
    restart: unless-stopped
    environment:
      TYPE: MAGMA_MAINTAINED
      VERSION: "1.12.2"
      FORGE_VERSION: "14.23.5.2860"
      MAGMA_MAINTAINED_TAG: "4c00eba"
      MODS: ./mods
      SPIGET_RESOURCES: 28140,19362 # this'll download LoginSecurity and LuckyPerms
      HIDE_ONLINE_PLAYERS: "true"
      EULA: "TRUE"
      ENABLE_WHITELIST: "true"
      EXISTING_WHITELIST_FILE: SYNC_FILE_MERGE_LIST
    volumes:
      - ./data:/data
      - ./whitelist.json:/data/whitelist.json
      - ./mods:/data/mods
    labels:
      - lazytainer.group=minecraft
    depends_on:
      - lazytainer
    network_mode: service:lazytainer
```
How do I check which `forge version` I need and what the hell is `magma maintained tag`?
Simply put: 
* `forge version` is the version you want the player's and server to work on
* `magma maintained tag` is the tag taken from github release id that tells magma which version to use

How do I take check the `magma maintained tag`? 
1. Goto [official magma github repository](https://github.com/magmamaintained), pick a version that you are planning to play.
2. After going into the version repository [FOR EXAMPLE 1.12.2](https://github.com/magmamaintained/Magma-1.12.2) click on the right on Releases.
3. Copy the id next to the Release name. For newest 1.12.2 version it'll be `4c00eba`