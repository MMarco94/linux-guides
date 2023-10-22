# Install Syncthing on an immutable OS

This guide assumes your system distribution uses `podman` and `systemd`. The guide was tested on Fedora Silverblue 35.

---

Syncthing is an open-source software that allows you to share files between devices. It's completely free, secure and decentralised.

At the end of this guide, Syncthing will be installed on your system as an user service, meaning it will start when you login and stop when you logout. Moreover, podman will automatically keep the container up to date automatically!  
Following the philosophy of immutable operative systems, no command in this guide needs super user privileges, nor will touch any file outside the user portion of the file system. 

As a secondary objective, this guide will try to explain each step, so you can hopefully adapt this guide to other services.

## Step 1: Create the required folders

As a first step, create the folder(s) that you'd like to share with other devices.  
I'll create two folders called `share1` and `share2` inside `Documents`, however feel free to create (or reuse) any folder(s) you like: at the end of this guide, those folders will be shared to another device.

You will also need an additional folder to store Syncthing's configuration. You can create it by running the following command on your terminal:

```
mkdir ~/.config/syncthing
```


## Step 2: Create the systemd Podman definition

Podman is a software to run other programs in a container completely isolated from the rest of the system.  
This is perfect for our use case, since we can use it to run Syncthing on top of our current installation, without the need to touch our system.

Systemd is a software suit that provides many functionalities, but for the context of this guide, we can think about it as the manager of all other software that runs on your machine.

In this step, we're going to create a systemd definition for Podman.  
Advanced users might recognise that what we're doing is a bit different: we're actually creating a Quadlet definition, that will get automatically converted to a systemd definition on our behalf. You can find more informations in [the official documentation](https://docs.podman.io/en/latest/markdown/podman-systemd.unit.5.html).

----

One of the biggest catalog of images that Podman can run is available on https://hub.docker.com. The images available there are made for Docker (an alternative to Podman), however they can usually run on Podman without any issue.

For this guide, we'll use the Syncthing image that's available on [https://hub.docker.com/r/linuxserver/syncthing](hub.docker.com/r/syncthing/syncthing).

1. Create a folder to store the systemd configuration:
    ```
    mkdir -p ~/.config/containers/systemd/
    ```
1. With your favorite text editor, create the file `~/.config/containers/systemd/syncthing.container` (`~` is your home directory).
    Using nano:
    ```
    nano ~/.config/containers/systemd/syncthing.container
    ```
1. Paste the following content
    ```
    [Unit]
    Description=Podman syncthing.service
    Wants=network-online.target
    After=network-online.target
    
    [Service]
    Restart=on-failure
    TimeoutStartSec=900
    
    [Container]
    Image=docker.io/syncthing/syncthing:latest
    AutoUpdate=registry
    PublishPort=127.0.0.1:8384:8384
    PublishPort=22000:22000/tcp
    PublishPort=22000:22000/udp
    PublishPort=21027:21027/udp
    UserNS=keep-id:uid=1000,gid=1000
    Volume=%h/.config/syncthing:/var/syncthing/config:Z

    # Folders to share
    Volume=%h/Documents/share1:/var/syncthing/share1:Z
    Volume=%h/Documents/share2:/var/syncthing/share2:Z
    
    [Install]
    WantedBy=default.target
    ```
    **Please replace `%h/Documents/share1` and `%h/Documents/share2` with the folder(s) you've created in step 1**. `%h` represents your home folder. Feel free to add or remove as many folders as you want.
1. Save the file (with nano, press `CTRL+O`, `Enter`, and then exit with `CTRL+X`).

Now, the systemd definition is in place. We just need to load it and start Syncthing:
```
systemctl --user daemon-reload
systemctl --user start syncthing.service
```

You can test that everything is working by connecting to http://localhost:8384. If everything worked, you should see the Syncthing homepage.

If you can't reach http://localhost:8384, those commands can help you troubleshoot:
- `systemctl --user status syncthing.service` to see the status of Syncthing, as seen by systemd. It contains a log of information, but in particular, you might be interested in the `Active` status and on the logs.
- `podman ps` will list all the running containers. You should see `syncthing` in there
- `podman logs syncthing` to see the messages produced by Syncthing


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

When sharing new folders, or adding folders shared by other devices, make sure to use **/var/syncthing/share1/** or **/var/syncthing/share2/** as the folder path, since that's how the folders created on step 1 are called inside the Syncthing container.

Apart from that, sharing folders between devices should work as expected.


## Step 4: Auto update

If you followed this guide, the Podman container is already setup to be auto-updated (we added `AutoUpdate=registry` in the systemd definition).

You can trigger an update manually, by calling
```
podman auto-update
```

Podman has also the optional ability to update containers automatically. To do so, you need to enable the relative systemd service:
```
systemctl enable --user --now podman-auto-update.timer
```

That's it! Now your container will stay up to date automatically!

# Extra

## How to add a folder
Adding folders after the container has been created it's pretty straight forward:
1. Locate the `systemd` service definition inside `~/.config/containers/systemd/`. In should be called `syncthing.container`
2. Using your favorite text editor, open the file and add another `Volume=` parameter. For example, add the argument `Volume=%h/Documents/share3:/var/syncthing/share3:Z` to share the folder `share3` inside `Documents`.  
3. Reload the `systemd` configuration using the command `systemctl --user daemon-reload`
4. Restart Syncthing with `systemctl --user restart container-syncthing.service`
5. On the Syncthing's web interface (`http://localhost:8384/`), add the folders. Using the same example as above, the folder to share is called `/var/syncthing/share3` inside the container

## Local discovery

If you have a keen eye, you might have noticed that files over the local network are not transferring as fast as they could. That's because local discovery doesn't work in the network isolation offered by Podman. To work around that, there are two options:
1. From _other_ devices, you change the device address from `dynamic` to your local IP. For example to `tcp://192.168.1.14`. This works only if the IP of your device is fixed, and the two devices never leave the local network.
1. Change the Podman definition to allow full network access to Syncthing. While it's not ideal from a security perspective, it does work. Edit `~/.config/containers/systemd/syncthing.container` to add `Network=host` inside the `[Container]` section. For example:
    ```
    [Unit]
    Description=Podman syncthing.service
    ...
    
    [Container]
    Image=docker.io/syncthing/syncthing:latest
    ...
    Network=host
    
    [Install]
    WantedBy=default.target
    ```
    In this setup, you can remove all `PublishPort` settings, as the container has unbounded access to the network.

# TODO

- How to uninstall everything
