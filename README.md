# Recompile kernel modules on boot with systemd

Anyone who has used Vmware on linux knows very well that it tends to break after updating the linux kernel. To fix that, you could get back to the previously working linux kernel, wait for the Vmware team to update vmware kernel modules, or ... manually recompile the modules

This command, used to manually recompile VMware kernel modules, should fix any break after updating the linux kernel:

**`sudo vmware-modconfig --console --install-all`**

Now, for some reason, the module `vmmon` doesn't load when my computer boots up. I have to type the command above every time i restart my OS to use VMware.
It is worth mentioning that I am running `RHEL 9` , Linux kernel `5.14.0-70.26.1` with secure boot disabled 

```console
$ dmesg | grep -i secure
[    0.000000] secureboot: Secure boot disabled
```

I decided to set up a systemd service to automate that command at every restart.

# Creating Systemd Services

Three services are needed and will  be installed in **/etc/systemd/system/**

The first service is `sudo vim /etc/systemd/system/vmware-modules-rebuild.service` and paste this code:

```console
[Unit]
Description=Recompiles vmware modules
Requires=vmware.service
Before=vmware.service
ConditionPathExists=!/dev/vmmon
 
[Service]
Type=oneshot
ExecStart=/usr/bin/vmware-modconfig --console --install-all
 
[Install]
WantedBy=multi-user.target
```
The next service is
`sudo vim /etc/systemd/system/vmware.service` and paste
```console
[Unit]
Description=VMware daemon
Requires=vmware-usbarbitrator.service
Before=vmware-usbarbitrator.service
After=network.target

[Service]
ExecStart=/etc/init.d/vmware start
ExecStop=/etc/init.d/vmware stop
PIDFile=/var/lock/subsys/vmware
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

and the last service 
`sudo vim /etc/systemd/system/vmware-usbarbitrator.service` and paste

```console
[Unit]
Description=VMware USB Arbitrator
Requires=vmware.service
After=vmware.service

[Service]
ExecStart=/usr/bin/vmware-usbarbitrator
ExecStop=/usr/bin/vmware-usbarbitrator --kill
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

You should now have 3 new services in **/etc/systemd/system** , `vmware-modules-rebuild.service` , `vmware.service` and `vmware-usbarbitrator.service`
Now reload systemd manager configuration, then enable and start the services just created with
```console
sudo systemctl daemon-reload
sudo systemctl enable vmware-modules-rebuild
sudo systemctl enable vmware.service
sudo systemctl enable vmware-usbarbitrator
sudo systemctl start vmware-modules-rebuild
sudo systemctl start vmware.service
sudo systemctl start vmware-usbarbitrator
```
I did not give it a try but there may be other ways to get this issue fixed. 

One would be to add the line `ibt=off` to your kernel command line.

The other option would be to load the modules at boot with `modules-load.d`

```console
sudo mkdir /etc/modules-load.d
sudo vim /etc/modules-load.d/vmware.conf

vmw_vmci 
vmmon 
vmnet

sudo modprobe -v vmmon
reboot
```

# References
https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/system_administrators_guide/chap-managing_services_with_systemd

https://wiki.archlinux.org/title/VMware

https://confluence.jaytaala.com/display/TKB/Recompile+VMWare+Workstation+%28or+player%29+kernel+modules+on+boot+with+systemd+servic

https://www.freedesktop.org/software/systemd/man/systemd.service.html

https://community.clearlinux.org/t/vmware-workstation-pro-post-installation-issues/5612
