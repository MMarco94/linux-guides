# Install Syncthing on an immutable OS

This guide assumes your system distribution uses `podman` and `systemd`. The guide was tested on Fedora Silverblue 35.

---

Syncthing is an open-source software that allows you to share files between devices. It's completely free, secure and decentralised.

At the end of this guide, Syncthing will be installed on your system as an user service, meaning it will start when you login and stop when you logout. Moreover, podman will automatically keep the container up to date automatically!  
Following the philosophy of immutable operative systems, no command in this guide needs super user privileges, nor will touch any file outside the user portion of the file system. 

As a secondary objective, this guide will try to explain each step, so you can hopefully adapt this guide to other services.

## Step 1: Create the required folders

As a first step, create the folder(s) that you'd like to share with other devices.  
I'll create two folders called `share1` and `share2` inside `Documents`. 

At the end of this guide, both folders will be shared to another device.

You will also need to create a folder that will contain the Syncthing's config. You can create it by running the following command on your terminal:

```
mkdir ~/.config/syncthing
```


## Step 2: Create the Podman container

Podman is a software to run other programs in a container completely isolated from the rest of the system.  
This is perfect for our use case, since we can use it to run Syncthing on top of our current installation, without the need to touch our system.

One of the biggest catalog of images that Podman can run is available on https://hub.docker.com. The images available there are made for Docker (an alternative to Podman), however they can usually run on Podman without any issue.

For this guide, we'll use the Syncthing image that's available on https://hub.docker.com/r/linuxserver/syncthing. To run it, we'll need to find the command to start the container. You can find it under the "docker cli" section. With a few modifications (explained below), we obtain the following command:

```
podman run -d \
  --name=syncthing \
  --label io.containers.autoupdate=registry \
  -e PUID=0 \
  -e PGID=0 \
  -p 127.0.0.1:8384:8384 \
  -p 22000:22000/tcp \
  -p 22000:22000/udp \
  -p 21027:21027/udp \
  -v ~/.config/syncthing:/config:Z \
  -v ~/Documents/share1:/data1:Z \
  -v ~/Documents/share2:/data2:Z \
  lscr.io/linuxserver/syncthing:latest
```

I've made the following modifications:
- Replace the `docker` command with `podman`
- Remove optional arguments, to make it simpler
- Add `--label io.containers.autoupdate=registry`, that will instruct podman to keep the container up to date
- Remove the restart policy, as later in the guide we'll use `systemd` to manage the execution of this container
- Add `127.0.0.1:` to the port binding, to make the administration webpage accessible only on your computer, to make things a bit more secure
- Use `0` instead of `1000` for PUID and GUID. While not ideal, those settings will make it so the processes *inside* the container will run as root, while *outside* the container will behave as your user. This setting is not suggested for production uses, but will work for our use case.
- Add `:Z` to the volume binding, to allow the container to read and write the folders we want to share.

**Please replace `~/Documents/share1` and `~/Documents/share2` with the folder(s) you've created in step 1**. As a reminder, `~` represents your home folder. Feel free to add or remove as many folders as you want.

Now you can open your terminal, and run the command above.  
After a few seconds (or minutes, depending on your connection), the command should return a string of letter and numbers, like `a6f7b9580c1fe90ea60cd8b56fdbd18bd0dbd4814a2cc9b63b6d440fc8f4528b`. You can ignore it, that's just the identifier for the container that is now running.

You can test that everything is working by connecting to http://localhost:8384. If everything worked, you should see the Syncthing homepage.

If you can't reach http://localhost:8384, those commands can help you troubleshoot:
- `podman ps` will list all the running containers. You should see `syncthing` in there
- `podman logs syncthing` to see the messages produced by Syncthing
- `podman exec -it syncthing /bin/bash` to open a terminal inside the container
- `podman stop syncthing && podman rm syncthing` to stop and remove the container. Useful if you need to start again from scratch



## Step 3: Configure Syncthing

If you open http://localhost:8384, you'll see a warning that asks you to set a password. I suggest you do that by going into "Settings" > "GUI" > "GUI Authentication User" and "GUI Authentication Password".

You can also go ahead and remove the folder shared by default. Expand "Default folder", then "Edit" and "Remove".

### Adding a device

*This step is not specific to our setup*

To share the folders on another device, you need to add that device first. Open Syncthing on the other device (e.g. oyur phone, with the [Syncthing](https://play.google.com/store/apps/details?id=com.nutomic.syncthingandroid) app), then add a device.  
On http://localhost:8384, press "Actions" > "Show ID" to get the necessary information.

After you've added the device, you'll get a prompt on the other one, that you'll need to accept.

I suggest adding the devices as "Introducer", so adding a third device will be easier, as new devices will get shared with the existing network.


### Sharing folders

When sharing new folders, or adding folders shared by other devices, make sure to use **/data1/** or **/data2/** as the folder path, since that's how the folders created on step 1 are called inside the Syncthing container.

Apart from that, sharing folders between devices should work as expected.

## Step 4: Create the `systemd` service

Right now the container has been created, but it needs to be started and stopped manually.

To start it:
```
podman start syncthing
```

To stop it:
```
podman stop syncthing
```


Ideally, we'd like that to be done automatically when we log in to our system. To achieve that, we can use `systemd`, a software that among other things can manage the execution of the container for us.

First, we need to create its configuration folder. On a terminal, run
```
mkdir -p ~/.config/systemd/user/
```

Then, enter that directory:

```
cd ~/.config/systemd/user/
```

Now we can use podman to generate the config for us. Run:
```
podman generate systemd --new --name syncthing -f
```

As output, you should see a file path. That means it worked!

The very last step is to make `systemd` aware of the new file, and enable the new service:

```
systemctl --user daemon-reload
systemctl --user enable container-syncthing.service
```

All is done! Now Syncthing will start automatically when you log in your system!


# TODO

This guide is not complete yet.

## Missing steps
- How to add new folders after the container has already been created
- How to uninstall everything

## To improve
- How to create the container so the internal process doesn't need to run as root
