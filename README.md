# Local LLaMA Server Setup Documentation

_TL;DR_: A guide to setting up a fully local and private language model server and TTS-equipped web UI, using [`ollama`](https://github.com/ollama/ollama), [Open WebUI](https://github.com/open-webui/open-webui), and [OpenedAI Speech](https://github.com/matatonic/openedai-speech).

## Table of Contents

- [Local LLaMA Server Setup Documentation](#local-llama-server-setup-documentation)
  - [Table of Contents](#table-of-contents)
  - [About](#about)
  - [Priorities](#priorities)
  - [System Requirements](#system-requirements)
  - [Prerequisites](#prerequisites)
  - [Essential Setup](#essential-setup)
  - [Additional Setup](#additional-setup)
    - [SSH](#ssh)
    - [Firewall](#firewall)
    - [Docker](#docker)
    - [Open WebUI](#open-webui)
    - [OpenedAI Speech](#openedai-speech)
      - [Open WebUI Integration](#open-webui-integration)
      - [Downloading Voices](#downloading-voices)
  - [Verifying](#verifying)
    - [Ollama](#ollama)
    - [Open WebUI](#open-webui-1)
    - [OpenedAI Speech](#openedai-speech-1)
  - [Updating](#updating)
    - [General](#general)
    - [Nvidia Drivers \& CUDA](#nvidia-drivers--cuda)
    - [Ollama](#ollama-1)
    - [Open WebUI](#open-webui-2)
    - [OpenedAI Speech](#openedai-speech-2)
  - [Troubleshooting](#troubleshooting)
    - [`ssh`](#ssh-1)
    - [Nvidia Drivers](#nvidia-drivers)
    - [Ollama](#ollama-2)
    - [Open WebUI](#open-webui-3)
    - [OpenedAI Speech](#openedai-speech-3)
  - [Monitoring](#monitoring)
  - [Notes](#notes)
    - [Software](#software)
    - [Hardware](#hardware)
  - [References](#references)
  - [Acknowledgements](#acknowledgements)

## About

This repository outlines the steps to run a server for running local language models. It uses Debian specifically, but most Linux distros should follow a very similar process. It aims to be a guide for Linux beginners like me who are setting up a server for the first time.

The process involves installing the NVIDIA drivers, setting the GPU power limit, and configuring the server to run `ollama` at boot. It also includes setting up auto-login and scheduling the `init.bash` script to run at boot. All these settings are based on my ideal setup for a language model server that runs most of the day but a lot can be customized to suit your needs. For example, you can use any OpenAI-compatible server like [`llama.cpp`](https://github.com/ggerganov/llama.cpp) or [LM Studio](https://lmstudio.ai) instead of `ollama`.

## Priorities

- **Simplicity of setup process**: It should be relatively straightforward to set up the components of the solution.
- **Stability of runtime**: The components should be stable and capable of running for weeks at a time without any intervention necessary.
- **Ease of maintenance**: The components and their interactions should be uncomplicated enough that you know enough to maintain them as they evolve (because they *will* evolve).
- **Aesthetics**: The result should be as close to a cloud provider's chat platform as possible. A homelab solution doesn't necessarily need to feel like it was cobbled together haphazardly.
- **Open source**: The code should be able to be verified by a community of engineers. Chat platforms and LLMs involve large amounts of personal data conveyed in natural language and it's important to know that data isn't going outside your machine.

## System Requirements

Any modern CPU and GPU combination should work for this guide. Previously, compatibility with AMD GPUs was an issue but the latest releases of `ollama` have worked through this and [AMD GPUs are now supported natively](https://ollama.com/blog/amd-preview). 

For reference, this guide was built around the following system:
- CPU: Intel Core i5-12600KF
- Memory: 32GB 6000 MHz DDR5 RAM
- Storage: 1TB M.2 NVMe SSD
- GPU: Nvidia RTX 3090 24GB

> [!NOTE] AMD GPU
> Power limiting is skipped for AMD GPUs as [AMD has recently made it difficult to set power limits on their GPUs](https://www.reddit.com/r/linux_gaming/comments/1b6l1tz/no_more_power_limiting_for_amd_gpus_because_it_is/). Naturally, skip any steps involving `nvidia-smi` or `nvidia-persistenced` and the power limit in the `init.bash` script.

> [!NOTE] CPU-only
> You can skip the GPU driver installation and power limiting steps. The rest of the guide should work as expected.

## Prerequisites

- Fresh install of Debian
- Internet connection
- Basic understanding of the Linux terminal
- Peripherals like a monitor, keyboard, and mouse

To install Debian on your newly built server hardware,

- Download the [Debian ISO](https://www.debian.org/distrib/) from the official website.
- Create a bootable USB using a tool like [Rufus](https://rufus.ie/en/) for Windows or [Balena Etcher](https://etcher.balena.io) for MacOS.
- Boot into the USB and install Debian.

For a more detailed guide on installing Debian, refer to the [official documentation](https://www.debian.org/releases/buster/amd64/). For those who aren't yet experienced with Linux, I recommend using the graphical installer - you will be given an option between the text-based installer and graphical installer. 

I also recommend installing a lightweight desktop environment like XFCE for ease of use. Other options like GNOME or KDE are also available - GNOME may be a better option for those using their server as a primary workstation as it is more feature-rich (and, as such, heavier) than XFCE.

## Essential Setup

1. ### Update the system
    - Run the following commands:
        ```
        sudo apt update
        sudo apt upgrade
        ```

2. ### Install drivers
    - #### Nvidia
      - Follow Nvidia's [guide on downloading CUDA Toolkit](https://developer.nvidia.com/cuda-downloads?target_os=Linux&target_arch=x86_64&Distribution=Debian). The instructions are specific to your machine and the website will lead you to them interactively.
      - Run the following commands:
          ```
          sudo apt install linux-headers-amd64
          sudo apt install nvidia-driver firmware-misc-nonfree
          ```
      - Reboot the server.
      - Run the following command to verify the installation:
          ```
          nvidia-smi
          ```
    
    - #### AMD
      - Run the following commands:
          ```
          deb http://deb.debian.org/debian bookworm main contrib non-free-firmware
          apt-get install firmware-amd-graphics libgl1-mesa-dri libglx-mesa0 mesa-vulkan-drivers xserver-xorg-video-all
          ```
      - Reboot the server.

3. ### Install `ollama`
    - Download `ollama` from the official repository:
        ```
        curl -fsSL https://ollama.com/install.sh | sh
        ```
    - (Recommended) We want our API endpoint to be reachable by the rest of the LAN. For `ollama`, this means setting `OLLAMA_HOST=0.0.0.0` in the `ollama.service`. 
      - Run the following command to edit the service:
        ```
        systemctl edit ollama.service
        ```
      - Find the `[Service]` section and add `Environment="OLLAMA_HOST=0.0.0.0"` under it. It should look like this:
        ```
        [Service]
        Environment="OLLAMA_HOST=0.0.0.0"
        ```
       - Save and exit.
       - Reload the environment.
            ```
            systemctl daemon-reload
            systemctl restart ollama
            ```
    
4. ### Create the `init.bash` script

    This script will be run at boot to set the GPU power limit and start the server using `ollama`. We set the GPU power limit lower because it has been seen in testing and inference that there is only a 5-15% performance decrease for a 30% reduction in power consumption. This is especially important for servers that are running 24/7.

    - Run the following commands:
        ```
        touch init.bash
        nano init.bash
        ```
    - Add the following lines to the script:
        ```
        #!/bin/bash
        sudo nvidia-smi -pm 1
        sudo nvidia-smi -pl (power_limit)
        ollama run (model)
        ollama serve
        ```
        > Replace `(power_limit)` with the desired power limit in watts. For example, `sudo nvidia-smi -pl 250`.

        > Replace `(model)` with the name of the model you want to run. For example, `ollama run mistral:latest`.

        For multiple GPUs, modify the script to set the power limit for each GPU:
        ```
        sudo nvidia-smi -i 0 -pl (power_limit)
        sudo nvidia-smi -i 1 -pl (power_limit)
        ```
    - Save and exit the script.
    - Make the script executable:
        ```
        chmod +x init.bash
        ```

5. ### Add `init.bash` to the crontab

    Adding the `init.bash` script to the crontab will schedule it to run at boot.

    - Run the following command:
        ```
        crontab -e
        ```
    - Add the following line to the file:
        ```
        @reboot /path/to/init.bash
        ```
        > Replace `/path/to/init.bash` with the path to the `init.bash` script.
    
    - (Optional) Add the following line to shutdown the server at 12am:
        ```
        0 0 * * * /sbin/shutdown -h now
        ```
    - Save and exit the file.

6. ### Give `nvidia-persistenced` and `nvidia-smi` passwordless `sudo` permissions

    We want `init.bash` to run the `nvidia-smi` commands without having to enter a password. This is done by editing the `sudoers` file.

    AMD users can skip this step as power limiting is not supported on AMD GPUs.

    - Run the following command:
        ```
        sudo visudo
        ```
    - Add the following lines to the file:
        ```
        (username) ALL=(ALL) NOPASSWD: /usr/bin/nvidia-persistenced
        (username) ALL=(ALL) NOPASSWD: /usr/bin/nvidia-smi
        ```
        > Replace `(username)` with your username.

        > [!IMPORTANT]
        > Ensure that you add these lines AFTER `%sudo ALL=(ALL:ALL) ALL`. The order of the lines in the file matters - the last matching line will be used so if you add these lines before `%sudo ALL=(ALL:ALL) ALL`, they will be ignored.
    - Save and exit the file.

7. ### Configure auto-login

    When the server boots up, we want it to automatically log in to a user account and run the `init.bash` script. This is done by configuring the `lightdm` display manager.

    - Run the following command:
        ```
        sudo nano /etc/lightdm/lightdm.conf
        ```
    - Find the following commented line. It should be in the `[Seat:*]` section.
        ```
        # autologin-user=
        ```
    - Uncomment the line and add your username:
        ```
        autologin-user=(username)
        ```
        > Replace `(username)` with your username.
    - Save and exit the file.

## Additional Setup

### SSH

Enabling SSH allows you to connect to the server remotely. After configuring SSH, you can connect to the server from another device on the same network using an SSH client like PuTTY or the terminal. This lets you run your server headlessly without needing a monitor, keyboard, or mouse after the initial setup.

On the server:
- Run the following command:
    ```
    sudo apt install openssh-server
    ```
- Start the SSH service:
    ```
    sudo systemctl start ssh
    ```
- Enable the SSH service to start at boot:
    ```
    sudo systemctl enable ssh
    ```
- Find the server's IP address:
    ```
    ip a
    ```

On the client:
- Connect to the server using SSH:
    ```
    ssh (username)@(ip_address)
    ```
    > Replace `(username)` with your username and `(ip_address)` with the server's IP address.

> [!NOTE]
> If you expect to tunnel into your server often, I highly recommend following [this guide](https://www.raspberrypi.com/documentation/computers/remote-access.html#configure-ssh-without-a-password) to enable passwordless SSH using `ssh-keygen` and `ssh-copy-id`. It worked perfectly on my Debian system despite having been written for Raspberry Pi OS.

### Firewall

Setting up a firewall is essential for securing your server. The Uncomplicated Firewall (UFW) is a simple and easy-to-use firewall for Linux. You can use UFW to allow or deny incoming and outgoing traffic to and from your server.

- Install UFW:
    ```
    sudo apt install ufw
    ```
- Allow SSH, HTTPS, and any other ports you need:
    ```
    sudo ufw allow ssh https 3000 11434 80 8000 8080
    ```
    Here, we're allowing SSH (port 22), HTTPS (port 443), Open WebUI (port 3000), Ollama API (port 11434), HTTP (port 80), OpenedAI Speech (8000), and Docker (port 8080). You can add or remove ports as needed.
- Enable UFW:
    ```
    sudo ufw enable
    ```
- Check the status of UFW:
    ```
    sudo ufw status
    ```

Refer to [this guide](https://www.digitalocean.com/community/tutorials/how-to-set-up-a-firewall-with-ufw-on-debian-10) for more information on setting up UFW.

### Docker

Docker is a containerization platform that allows you to run applications in isolated environments. This subsection follows [this guide](https://docs.docker.com/engine/install/debian/) to install Docker Engine on Debian.

- Run the following commands:
    ```
    # Add Docker's official GPG key:
    sudo apt-get update
    sudo apt-get install ca-certificates curl
    sudo install -m 0755 -d /etc/apt/keyrings
    sudo curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
    sudo chmod a+r /etc/apt/keyrings/docker.asc

    # Add the repository to Apt sources:
    echo \
    "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian \
    $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
    sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
    sudo apt-get update
    ```
- Install the Docker packages:
    ```
    sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
    ```
- Verify the installation:
    ```
    sudo docker run hello-world
    ```

### Open WebUI

Open WebUI is a web-based interface for managing Ollama models and chats, and provides a beautiful, performant UI for communicating with your models. You will want to do this if you want to access your models from a web interface. If you're fine with using the command line or want to consume models through a plugin/extension, you can skip this step.

To install without Nvidia GPU support, run the following command:
```
sudo docker run -d -p 3000:8080 --add-host=host.docker.internal:host-gateway -v open-webui:/app/backend/data --name open-webui --restart always ghcr.io/open-webui/open-webui:main
```

For Nvidia GPUs, run the following command:
```
sudo docker run -d -p 3000:8080 --gpus all --add-host=host.docker.internal:host-gateway -v open-webui:/app/backend/data --name open-webui --restart always ghcr.io/open-webui/open-webui:cuda
```

You can access it by navigating to `http://localhost:3000` in your browser or `http://(server_IP):3000` from another device on the same network. There's no need to add this to the `init.bash` script as Open WebUI will start automatically at boot via Docker Engine.

Read more about Open WebUI [here](https://github.com/open-webui/open-webui).

### OpenedAI Speech

OpenedAI Speech is a text-to-speech server that wraps [Piper TTS](https://github.com/rhasspy/piper) and [Coqui XTTS v2](https://docs.coqui.ai/en/latest/models/xtts.html) in an OpenAI-compatible API. This is great because it plugs in easily to the Open WebUI interface, giving your models the ability to speak their responses.

> As of v0.17 (compared to v0.10), OpenedAI Speech features a far more straightforward and automated Docker installation, making it easy to get up and running.

Piper TTS is a lightweight model that is great for quick responses - it can also run CPU-only inference, which may be a better fit for systems that need to reserve as much VRAM for language models as possible. XTTS is a more performant model that requires a GPU for inference. Piper is:

1) generally easier to setup with out-of-the-box CUDA acceleration, and,
2) has a plethora of voices that can be found [here](https://rhasspy.github.io/piper-samples/), so it's what I would suggest starting with.

- To install OpenedAI Speech, first clone the repository and navigate to the directory:
    ```
    git clone https://github.com/matatonic/openedai-speech
    cd openedai-speech
    ```
- Copy the `sample.env` file to `speech.env`:
    ```
    cp sample.env speech.env
    ```
- Run the following command to start the server.
  - Nvidia GPUs
    ```
    sudo docker compose up -d
    ```
  - AMD GPUs
    ```
    sudo docker compose -d docker-compose.rocm.yml up
    ```
  - CPU only
    ```
    sudo docker compose -f docker-compose.min.yml up
    ```

OpenedAI Speech runs on `0.0.0.0:8000` by default. You can access it by navigating to `http://localhost:8000` in your browser or `http://(server_IP):8000` from another device on the same network without any additional changes.

#### Open WebUI Integration

To integrate your OpenedAI Speech server with Open WebUI, navigate to the `Audio` tab under `Settings` in Open WebUI and set the following values:
- Text-to-Speech Engine: `OpenAI`
- API Base URL: `http://host.docker.internal:8000/v1`
- API Key: `anything-you-like`
- Set Model: `tts-1` (for Piper) or `tts-1-hd` (for XTTS)

> [!NOTE]
> `host.docker.internal` is a magic hostname that resolves to the internal IP address assigned to the host by Docker. This allows containers to communicate with services running on the host, such as databases or web servers, without needing to know the host's IP address. It simplifies communication between containers and host-based services, making it easier to develop and deploy applications.

> [!NOTE]
> The TTS engine is set to `OpenAI` because OpenedAI Speech is OpenAI-compatible. There is no data transfer between OpenAI and OpenedAI Speech - the API is simply a wrapper around Piper and XTTS.

#### Downloading Voices

We'll use Piper here because I haven't found any good resources for high quality .wav files for XTTS. The process is the same for both models, just replace `tts-1` with `tts-1-hd` in the following commands. We'll download the `en_GB-alba-medium` voice as an example.

- Create a new virtual environment named `speech` and activate it. Then, install `piper-tts`:
    ```
    python3 -m venv speech
    source speech/bin/activate
    pip install piper-tts
    ```
    This is a minimal virtual environment that is only required to run the script that downloads voices.
- Download the voice:
    ```
    bash download_voices_tts-1.sh en_GB-alba-medium
    ```
- Update the `voice_to_speaker.yaml` file to include the voice you downloaded. This file maps the voice to a speaker name that can be used in the Open WebUI interface. For example, to map the `en_GB-alba-medium` voice to the speaker name `alba`, add the following lines to the file:
    ```
    alba:
        model: voices/en_GB-alba-medium.onnx
        speaker: # default speaker
    ```
- Run the following command:
    ```
    sudo docker ps -a
    ```
    Identify the container IDs of
    1) OpenedAI Speech
    2) Open WebUI

    Restart both containers:
    ```
    sudo docker restart (openedai_speech_container_ID)
    sudo docker restart (open_webui_container_ID)
    ```
    > Replace `(openedai_speech_container_ID)` and `(open_webui_container_ID)` with the container IDs you identified.

## Verifying

This section isn't strictly necessary by any means - if you use all the elements in the guide, a good experience in Open WebUI means you've succeeded with the goal of the guide. However, it can be helpful to test the disparate installations at different stages in this process.

### Ollama

To test your Ollama installation and endpoint, simply run:
```
curl http://localhost:11434/api/generate -d '{
  "model": "llama2",
  "prompt":"Why is the sky blue?"
}'
```
> Replace `llama2` with your preferred model.

Since the OLLAMA_HOST environment variable is set to 0.0.0.0, it's easy to access `ollama` from anywhere on the network. To do so, simply update the `localhost` reference in your URL or command to match the IP address of your server.

Refer to [Ollama's REST API docs](https://github.com/ollama/ollama/blob/main/docs/api.md) for more information on the entire API.

### Open WebUI

Visit `http://localhost:3000`. If you're greeted by the authentication page, you've successfully installed Open WebUI.

### OpenedAI Speech

To test your TTS server and endpoint, run the following command:
```
curl -s http://localhost:8000/v1/audio/speech -H "Content-Type: application/json" -d '{
    "input": "The quick brown fox jumped over the lazy dog."}' > speech.mp3
```

If you see the `speech.mp3` file in the OpenedAI Speech directory, you should be good to go. If you're paranoid like I am, test it using a player like `aplay`. Run the following commands (from the OpenedAI Speech directory):
```
sudo apt install aplay
aplay speech.mp3
```

## Updating

Updating your system is a good idea to keep software running optimally and with the latest security patches. Updates to Ollama (a Docker-based wrapper around `llama.cpp`) allow for inference from new model architectures and updates to Open WebUI enable new features like voice calling, function calling, pipelines, and more.

I've compiled steps to update these "primary function" installations in a standalone section because I think it'd be easier to come back to one section instead of hunting for update instructions in multiple subsections.

### General

Upgrade Debian packages by running the following commands:
```
sudo apt update
sudo apt upgrade
```

### Nvidia Drivers & CUDA

Follow Nvidia's guide [here](https://developer.nvidia.com/cuda-downloads?target_os=Linux&target_arch=x86_64&Distribution=Debian) to install the latest CUDA drivers.

> [!WARNING]
> Don't skip this step. Not installing the latest drivers after upgrading Debian packages will throw your installations out of sync, leading to broken functionality. When updating, target everything important at once. Also, rebooting after this step is a good idea to ensure that your system is operating as expected after upgrading these crucial drivers.

### Ollama

Rerun the command that installs Ollama - it acts as an updater too:
```
curl -fsSL https://ollama.com/install.sh | sh
```

### Open WebUI

To update Open WebUI once, run the following command:
```
docker run --rm --volume /var/run/docker.sock:/var/run/docker.sock containrrr/watchtower --run-once open-webui
```

To keep it updated automatically, run the following command:
```
docker run -d --name watchtower --volume /var/run/docker.sock:/var/run/docker.sock containrrr/watchtower open-webui
```

### OpenedAI Speech

Navigate to the directory and pull the latest image from Docker:
```
cd openedai-speech
sudo docker compose pull
sudo docker compose up -d
```

## Troubleshooting

For any service running in a container, you can check the logs by running `sudo docker logs -f (container_ID)`. If you're having trouble with a service, this is a good place to start.

### `ssh`
- If you encounter an issue using `ssh-copy-id` to set up passwordless SSH, try running `ssh-keygen -t rsa` on the client before running `ssh-copy-id`. This generates the RSA key pair that `ssh-copy-id` needs to copy to the server.

### Nvidia Drivers
- Disable Secure Boot in the BIOS if you're having trouble with the Nvidia drivers not working. For me, all packages were at the latest versions and `nvidia-detect` was able to find my GPU correctly, but `nvidia-smi` kept returning the `NVIDIA-SMI has failed because it couldn't communicate with the NVIDIA driver` error. [Disabling Secure Boot](https://askubuntu.com/a/927470) fixed this for me. Better practice than disabling Secure Boot is to sign the Nvidia drivers yourself but I didn't want to go through that process for a non-critical server that can afford to have Secure Boot disabled.

### Ollama
- If you receive the `Failed to open "/etc/systemd/system/ollama.service.d/.#override.confb927ee3c846beff8": Permission denied` error from Ollama after running `systemctl edit ollama.service`, simply creating the file works to eliminate it. Use the following steps to edit the file. 
  - Run:
    ```
    sudo mkdir -p /etc/systemd/system/ollama.service.d
    sudo nano /etc/systemd/system/ollama.service.d/override.conf
    ```
  - Retry the remaining steps.
- If you still can't connect to your API endpoint, check your firewall settings. [This guide to UFW (Uncomplicated Firewall) on Debian](https://www.digitalocean.com/community/tutorials/how-to-set-up-a-firewall-with-ufw-on-debian-10) is a good resource.

### Open WebUI
- If you encounter `Ollama: llama runner process has terminated: signal: killed`, check your `Advanced Parameters`, under `Settings > General > Advanced Parameters`. For me, bumping the context length past what certain models could handle was breaking the `ollama` server. Leave it to the default (or higher, but make sure it's still under the limit for the model you're using) to fix this issue.

### OpenedAI Speech
- If you encounter `docker: Error response from daemon: Unknown runtime specified nvidia.` when running `docker compose up -d`, ensure that you have `nvidia-container-toolkit` installed (this was previously `nvidia-docker2`, which is now deprecated). If not, installation instructions can be found [here](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html). Make sure to reboot the server after installing the toolkit. If you still encounter issues, ensure that your system has a valid CUDA installation by running `nvcc --version`.
- If `nvcc --version` doesn't return a valid response despite following Nvidia's installation guide, the issue is likely that CUDA is not in your PATH variable.
    Run the following command to edit your `.bashrc` file:
    ```
    sudo nano /home/(username)/.bashrc
    ```
    > Replace `(username)` with your username.
    Add the following to your `.bashrc` file:
    ```
    export PATH="/usr/local/(cuda_version)/bin:$PATH"
    export LD_LIBRARY_PATH="/usr/local/(cuda_version)/lib64:$LD_LIBRARY_PATH"
    ```
    > Replace `(cuda_version)` with your installation's version. If you're unsure of which version, run `ls /usr/local` to find the CUDA directory. It is the directory with the `cuda` prefix, followed by the version number. 
    
    Save and exit the file, then run `source /home/(username)/.bashrc` to apply the changes (or close the current terminal and open a new one). Run `nvcc --version` again to verify that CUDA is now in your PATH. You should see something like the following:
    ```
    nvcc: NVIDIA (R) Cuda compiler driver
    Copyright (c) 2005-2024 NVIDIA Corporation
    Built on Thu_Mar_28_02:18:24_PDT_2024
    Cuda compilation tools, release 12.4, V12.4.131
    Build cuda_12.4.r12.4/compiler.34097967_0
    ```
    If you see this, CUDA is now in your PATH and you can run `docker compose up -d` again.
- If you run into a `VoiceNotFoundError`, you may either need to download the voices again or the voices may not be compatible with the model you're using. Make sure to check your `speech.env` file to ensure that the `PRELOAD_MODEL` and `CLI_COMMAND` lines are configured correctly.

## Monitoring

To monitor GPU usage, power draw, and temperature, you can use the `nvidia-smi` command. To monitor GPU usage, run:
```
watch -n 1 nvidia-smi
```
This will update the GPU usage every second without cluttering the terminal environment. Press `Ctrl+C` to exit.

## Notes

This is my first foray into setting up a server and ever working with Linux so there may be better ways to do some of the steps. I will update this repository as I learn more.

### Software

- I chose Debian because it is, apparently, one of the most stable Linux distros. I also went with an XFCE desktop environment because it is lightweight and I wasn't yet comfortable going full command line.
- Use a user for auto-login, don't log in as root unless for a specific reason.
- If something using a Docker container doesn't work, try running `sudo docker ps -a` to see if the container is running. If it isn't, try running `sudo docker compose up -d` again. If it is and isn't working, try running `sudo docker restart (container_ID)` to restart the container.
- If something isn't working no matter what you do, try rebooting the server. It's a common solution to many problems. Try this before spending hours troubleshooting. Sigh.

### Hardware

- The power draw of my EVGA FTW3 Ultra RTX 3090 was 350W at stock settings. I set the power limit to 250W and the performance decrease was negligible for my use case, which is primarily code completion in VS Code and the Q&A via chat. 
- Using a power monitor, I measured the power draw of my server for multiple days - the running average is ~60W. The power can spike to 350W during prompt processing and token generation, but this only lasts for a few seconds. For the remainder of the generation time, it tended to stay at the 250W power limit and dropped back to the average power draw after the model wasn't in use for about 20 seconds. 
- Ensure your power supply has enough headroom for transient spikes (particularly in multi GPU setups) or you may face random shutdowns. Your GPU can blow past its rated power draw and also any software limit you set for it based on the chip's actual draw. I usually aim for +50% of my setup's estimated total power draw.

## References

Downloading Nvidia drivers:
- https://developer.nvidia.com/cuda-downloads?target_os=Linux&target_arch=x86_64&Distribution=Debian
- https://wiki.debian.org/NvidiaGraphicsDrivers

Downloading AMD drivers:
- https://wiki.debian.org/AtiHowTo

Secure Boot:
- https://askubuntu.com/a/927470

Monitoring GPU usage, power draw: 
- https://unix.stackexchange.com/questions/38560/gpu-usage-monitoring-cuda/78203#78203

Passwordless `sudo`:
- https://stackoverflow.com/questions/25215604/use-sudo-without-password-inside-a-script
- https://www.reddit.com/r/Fedora/comments/11lh9nn/set_nvidia_gpu_power_and_temp_limit_on_boot/
- https://askubuntu.com/questions/100051/why-is-sudoers-nopasswd-option-not-working

Auto-login:
- https://forums.debian.net/viewtopic.php?t=149849
- https://wiki.archlinux.org/title/LightDM#Enabling_autologin

Expose Ollama to LAN:
- https://github.com/ollama/ollama/blob/main/docs/faq.md#setting-environment-variables-on-linux
- https://github.com/ollama/ollama/issues/703

Firewall:
- https://www.digitalocean.com/community/tutorials/how-to-set-up-a-firewall-with-ufw-on-debian-10

Passwordless `ssh`:
- https://www.raspberrypi.com/documentation/computers/remote-access.html#configure-ssh-without-a-password

Adding CUDA to PATH:
- https://askubuntu.com/questions/885610/nvcc-version-command-says-nvcc-is-not-installed

Docs:

- [Debian](https://www.debian.org/releases/buster/amd64/)
- [Ollama](https://github.com/ollama/ollama/blob/main/docs/api.md)
- [Docker for Debian](https://docs.docker.com/engine/install/debian/)
- [Open WebUI](https://github.com/open-webui/open-webui)
- [OpenedAI Speech](https://github.com/matatonic/openedai-speech)

## Acknowledgements

Cheers to all the fantastic work done by the open-source community. This guide wouldn't exist without the effort of the many contributors to the projects and guides referenced here. 

To stay up-to-date on the latest developments in the field of machine learning, LLMs, and other vision/speech models, check out [r/LocalLLaMA](https://reddit.com/localllama).

> [!NOTE]
> Please star any projects you find useful and consider contributing to them if you can. Stars on this guide would also be appreciated if you found it helpful, as it helps others find it too. 