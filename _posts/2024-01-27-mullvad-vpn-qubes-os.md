---
layout: post
title:  "Mullvad VPN on Qubes OS"
date:   2024-01-27 23:08:14 +0100
categories: Network
tags: VPN Wireguard Mullvad Qubes-OS
---

![Tunnel](/assets/tunnel.jpg "Tunnel")

## Introduction

If you care about security and privacy, Qubes OS is a pretty good choice as it implements securely-isolated compartments called "Qubes" and integrates notably Whonix as a Tor gateway. More precisely, Qubes are virtual machines that takes advantage of all virtualization features to create those isolated compartments.

It is still possible to increase your privacy, espacially from your ISP, in using a VPN service like [Mullvad](https://mullvad.net){:target="_blank"} within Qubes OS. There are plenty of VPN providers but [Mullvad and some others](https://www.privacyguides.org/en/vpn){:target="_blank"} have made big efforts on transparency and are providing open-source clients (cf. [Mullvad VPN application](https://mullvad.net/en/download/vpn){:target="_blank"}). Mullvad already provides guides like [this one](https://mullvad.net/en/help/wireguard-on-qubes-os/){:target="_blank"} to help you configure the service on Qubes OS. It however only creates tunnel with a specifically choosen server and thus doesn't use all the benefits of Mullvad VPN application like switching server, location or protocol.

We will explain how to install Mullvad VPN application in a similar configuration : a proxy Qube which provides to any other Qubes an Internet access through a Mullvad VPN.

The procedure bellow has been tested with **Qubes OS 4.1**.

## Mullvad VPN application installation inside a proxy Qube
### Create a new template

Qubes OS project's security model give a great importance to base templates like Fedora or Debian. This is why any software should be installed from the default repositories of these distributions and, when it is not possible, the security of the template should be considered as downgraded.

As Mullvad VPN application is not in the default repositories, we will clone the Fedora template and use this new one to install the software as discribed in the [Qubes OS documentation](https://www.qubes-os.org/doc/how-to-install-software/#installing-software-from-other-sources){:target="_blank"} :

- Open the `Qube Manager` (`Qubes Tools` / `Qube Manager`)
- Click on the default Fedora template and open the `Settings` window
- You should be able to click on `Clone qube` button at the bottom right of the window
- Choose a non-black color for this template (red for example) indicating that it is potentially less secure that the default template
- Call your new template `fedora-XX-mullvad` where `XX` is the Fedora release number

### Give Internet access to the template

This is the risky part of the process. Qubes OS project's security model considers that templates are secured as it has no access to Internet and can only download packages from default repositories through a proxy. We will however have to give Internet access to our new template in order to download the necessary files :

- In the `Qube Manager`, click on the `fedora-XX-mullvad` template and open the `Settings` window
- For `Net qube` option, select `sys-firewall`
- In `Firewall rules` tab, select `Limit outgoing connections to ...` and click on `Allow full access for 15 min` in order to get an Internet access during 15 minutes (which should be enough to install Mullvad VPN application) and block it again when time has elapsed.


**NOTE :**
Adding and removing Internet access to the template is the prescribed way in Qubes OS documentation. You can however bypass these steps without added risks in reaching directly the Qube proxy :
- Set `https_proxy` environment variable
{% highlight bash %}
export https_proxy=127.0.0.1:8082
{% endhighlight %}
- Install Mullvad VPN App as explained below
- Once everything is installed, unset `https_proxy` environment variable
{% highlight bash %}
unset https_proxy
{% endhighlight %}

### Install Mullvad VPN application

We will simply follow the Mullvad VPN application installation instructions on Fedora with a signature check :

- Start the template `fedora-XX-mullvad`
- Open a terminal for this template
- Download the signing key provided by Mullvad which should has been used to sign the RPM package : 
{% highlight bash %}
curl https://mullvad.net/media/mullvad-code-signing.asc -o mullvad-code-signing.asc
{% endhighlight %}
- Import the key and check that its fingerprint is **`A119 8702 FC3E 0A09 A9AE 5B75 D5A1 D4F2 66DE 8DDF`**
{% highlight bash %}
gpg --import mullvad-code-signing.asc
{% endhighlight %}
(If you want to verify the fingerprint before importing the key, you can do so with `gpg --show-keys --with-fingerprint mullvad-code-signing.asc`)
- Edit the key with the command `gpg --edit-key A1198702FC3E0A09A9AE5B75D5A1D4F266DE8DDF`. The output should be something like this :
{% highlight bash %}
gpg (GnuPG) 2.1.13; Copyright (C) 2016 Free Software Foundation, Inc.
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.

pub  rsa4096/D5A1D4F266DE8DDF
     created: 2016-10-27  expires: never       usage: SC 
     trust: unknown       validity: unknown
sub  rsa4096/C187D22C089EF64E
     created: 2016-10-27  expires: never       usage: E  
sub  rsa4096/A26581F219C8314C
     created: 2016-10-27  expires: never       usage: S  
[ unknown] (1). Mullvad (code signing) <admin@mullvad.net>
{% endhighlight %}

- If everything seems correct, you can trust the key via the gpg prompt : `gpg> trust`. You will be asked to set the trust level. Set it to `5` in order to trust the key ultimately
- Leave the gpg prompt : `gpg> q`
- Download the app and its signature :
{% highlight bash %}
curl https://mullvad.net/download/app/rpm/latest -o mullvad.rpm
curl https://mullvad.net/download/app/rpm/latest/signature -o mullvad.asc
{% endhighlight %}
- In the same folder where the files have been downloaded, run `gpg --verify mullvad.asc mullvad.rpm` to proceed to the signature check. You should see in the output of this command something like this :
{% highlight bash %}
gpg: Good signature from "Mullvad (code signing) <admin@mullvad.net>" [ultimate]
{% endhighlight %}
- If every verifications above have seceeded, you can then install the app with the folowing command :
{% highlight bash %}
sudo dnf install -y ./mullvad.rpm
{% endhighlight %}
- When the installation is finished, enable `mullvad-daemon` service so that the app launches automatically when the Qube starts :
{% highlight bash %}
sudo systemctl enable mullvad-daemon
{% endhighlight %}
- Finally shutdown the template

### Remove Internet access to the template

As we have given access to Internet to our new template, we will remove this one for consistency with Qubes OS project recommendations.

- In the `Qube Manager`, click on the `fedora-XX-mullvad` template and open the `Settings` window
- For `Net qube` option, select back `default (none)`

### Create the proxy Qube

We will use our new template to create the proxy Qube which will privide an Internet access to any other Qube.

- Open the `Qube Manager` (`Qubes Tools` / `Qube Manager`)
- Click on `New qube` to create a new Qube
- In `Basic` tab, you can call this Qube `sys-vpn` and use the color `red`. Select `AppVM` as Type, `fedora-XX-mullvad` as Template and `default (sys-firewall)` as Networking.
- In `Advanced` tab, enable `Provides network access to other qubes`

### Configure Mullvad VPN

Everything should be ready to use Mullvad VPN. Start `sys-vpn` and use the following commands to configure your Mullvad account from CLI :

- Configure your acount :
{% highlight bash %}
mullvad account login
{% endhighlight %}
You will be asked to enter your Mullvad account number.
- Verify that your device is registered :
{% highlight bash %}
mullvad account get
{% endhighlight %}
You should see the device name that has been assigned by Mullvad to your Qube.
- Configure the server you want to use (*e.g.* `se-mma-wg-001` which is a Wireguard server located in Sweden, in the city of Malmö) :
{% highlight bash %}
mullvad relay set location se-mma-wg-001
{% endhighlight %}
- Connect to the server :
{% highlight bash %}
mullvad connect
{% endhighlight %}
- Check that you are connected :
{% highlight bash %}
mullvad status
{% endhighlight %}
You should see something like `Connected to se-mma-wg-001 in Malmö, Sweden`.
- Finally, auto-connect Mullvad VPN each time the Qube starts :
{% highlight bash %}
mullvad auto-connect set on
{% endhighlight %}

### Fix DNS problem

As Mullvad VPN application will change the DNS server in `/etc/resolv.conf`, it is necessary to run again the `/usr/lib/qubes/qubes-setup-dnat-to-ns` script which is normally executed when the Qube starts. It will update the DNAT rules in `iptables` so that to reach the right DNS server. You can verify the consistency of `/etc/resolv.conf` file and the DNAT rules with the command `sudo iptables -t nat -L`. This is a **minimal workaround** and this [DNS update should be directly managed by Mullvad VPN application](https://github.com/mullvad/mullvadvpn-app/issues/3803){:target="_blank"}.

To do so, simply run the following command in `sys-vpn` Qube :
{% highlight bash %}
sudo /usr/lib/qubes/qubes-setup-dnat-to-ns
{% endhighlight %}

### Configure persistency for Mullvad VPN application configuration

Most of data in `AppVM` are volatile, especially the ones in `/etc` folder and subfolders. Each `AppVM` Qube has a persistent folder in `/rw`. This is where you can keep configurations files across Qube reboots (*e.g.* in `/rw/config`). We will use this folder in order to be sure that configurations made in Mullvad VPN application are kept after each reboot. To do so, we will use the `/rw/config/rc.local` script which is executed everytime the Qube starts.

- Create a persistent folder for Mullvad VPN application configuration files :
{% highlight bash %}
sudo mkdir /rw/config/mullvad-vpn
{% endhighlight %}
- Copy the already existing configuration files :
{% highlight bash %}
sudo cp /etc/mullvad-vpn/* /rw/config/mullvad-vpn/
{% endhighlight %}
- Customise the `rc.local` file to reconfigure Mullvad VPN application at every startup. The file should looks like that :
{% highlight bash %}
#!/bin/sh

# Configure Mullvad VPN application
rm -rf /etc/mullvad-vpn/
ln -s /rw/config/mullvad-vpn /etc/mullvad-vpn
systemctl restart mullvad-daemon

# fix DNS problem (https://github.com/mullvad/mullvadvpn-app/issues/3803)
sleep 10 # Waiting a bit that Mullvad succeeds to establish connection
/usr/lib/qubes/qubes-setup-dnat-to-ns
{% endhighlight %}

### Provide Internet access to other Qubes through Mullvad VPN

We now have our proxy Qube called `sys-vpn` configured to build a tunnel to a Mullvad server using Mullvad VPN application. The last thing to do is to configure your Qubes to reach Internet through Mullvad VPN. For those Qubes, the only thing to do is to open the `Settings` and select `sys-vpn` for `Net qube` option instead of `default (sys-firewall)`.

You can verify that you are accessing Internet through a Mullvad VPN :
- Open a terminal from the Qube connected to `sys-vpn`
- Run the following command : `curl https://am.i.mullvad.net/connected`. The output should starts with `You are connected to Mullvad ...`

### (Optional) Manage configuration via Mullvad VPN application GUI

With all the steps above, you should have a working configuration which let you access Internet through a Mullvad VPN. However we interacted with Mullvad VPN application [through the CLI](https://mullvad.net/fr/help/how-use-mullvad-cli/){:target="_blank"}.

But you can also access to the GUI to interact with Mullvad VPN application. In `sys-vpn` qube settings, open `Applications` tab and add `Mullvad VPN` to the selected application.

You can then access the App GUI via `Service : sys-vpn` / `sys-vpn : Mullvad VPN`.

## Conclusion

We have set up a proxy VM which is able to provide to other Qubes an Internet access through a Mullvad VPN. This kind of configuration is based on what is done by default in Qubes OS with `sys-whonix` service which provides an Internet access through Tor to other Qubes.

As it has already been mentionned, the DNS update in `rc.local` file is a minimal workaround and has many ceveats. For example, if Mullvad VPN application connects to another server and changes the DNS, the DNAT rules won't be updated automatically and the `/usr/lib/qubes/qubes-setup-dnat-to-ns` script should be run again manually. This is why the DNS issue should clearly be managed directly by Mullvad VPN application and the is the goal of this [Github issue](https://github.com/mullvad/mullvadvpn-app/issues/3803){:target="_blank"}.

## Sources
- [Qubes OS Documentation : How to install software](https://www.qubes-os.org/doc/how-to-install-software/){:target="_blank"}
- [Qubes OS Documentation : Networking](https://www.qubes-os.org/doc/networking/){:target="_blank"}
- [Mullvad Documentation : Wireguard on Qubes OS](https://mullvad.net/en/help/wireguard-on-qubes-os/){:target="_blank"}
- [VPN providers respecting privacy](https://www.privacyguides.org/en/vpn){:target="_blank"}
