# Secure GCP VM with OWASP ZAP Proxy

The goal of the guide is to get you up and running with a GUI on a GCP VM with the OWASP ZAP proxy.  It's an open source penetration testing tool for finding web app vulnerabilities.

But we don't want just any old VM on the default network.  This guide will enable the above to work on a VM **without an external IP address and tunnel your VNC Client over SSH** into the VM using GCP's Identity Aware Proxy.

## Requirements

1. If using Windows on your workstation (not in GCP), you must install WSL2 (IAP does not work with Windows `gcloud compute ssh`).
1. Have a fully working Google Cloud SDK (`gcloud` CLI)
1. Have created a project, and know its Project ID.
1. Have choosen a desired GCP Region and Zone (pick one phsically closest to you).

## Create a Network

Set some env vars.  Alternatively, when you do `gcloud init` you can set defaults and no not need these args.

    export PROJECT=<your project id>
    export REGION=<your desired region, us-west1 for example>
    export ZONE=<your desired zone, us-west1-a for example>

Enable Compute API

    gcloud services enable compute.googleapis.com --project=$PROJECT

Create a custom network, don't use the default one!

    gcloud compute networks create simple-net \
    --subnet-mode=custom \
    --bgp-routing-mode=regional \
    --project $PROJECT

Create a subnet

    gcloud compute networks subnets create simple-subnet \
    --network=simple-net \
    --range=10.122.0.0/20 \
    --region=$REGION \
    --project=$PROJECT

## Enable SSH and Cloud NAT

Add a firewall rule.  The IP in this command the IP of GCP's IAP.

    gcloud compute firewall-rules create allow-ssh-from-iap \
    --network=simple-net \
    --direction=INGRESS \
    --action=allow \
    --rules=tcp:22 \
    --source-ranges=35.235.240.0/20 \
    --project=$PROJECT

Grant a user access to use IAP, replace `<your email>` below with one you use to login into the GCP console. So regardless of what IP the user is coming from, IAP will only authorize the user or group you set in this command. Keep in mind if your gcloud tokens are comprised then they can still be used from any source IP unless you restrict further.

    gcloud projects add-iam-policy-binding $PROJECT \
    --member=user:<your email> \
    --role=roles/iap.tunnelResourceAccessor

This is only required if your VM needs connectivity to the Internet. This is safe and permits egress from your GCP network out and not ingress from external IP's.  **This is required since we need to get updates from the internet.**

    gcloud compute routers create nat-router \
    --network simple-net \
    --region $REGION \
    --project $PROJECT

Configure the router

    gcloud compute routers nats create nat-config \
    --router-region $REGION \
    --router nat-router \
    --nat-all-subnet-ip-ranges \
    --auto-allocate-nat-external-ips \
    --project $PROJECT

## Create a VM

You will need to right size this for your use.  Since I'm only using OWASP ZAP Proxy it wont need much.  The default disk (10GB) will fill up a little due to the deps we will be installing.  If you are planning to install other recon software refer to the GCP docs on how to make a bigger disk for this VM.

    gcloud compute instances create zap-vm \
    --image-family=ubuntu-2004-lts \
    --image-project=ubuntu-os-cloud \
    --machine-type=e2-standard-2 \
    --tags=allow-ssh-from-iap \
    --network-interface=no-address,network=simple-net,subnet=simple-subnet \
    --zone=$ZONE \
    --project=$PROJECT

Make sure you can SSH into your VM, it should say you are using the IAP tunnel.  Test connectivity.

    gcloud compute ssh zap-vm --project=$PROJECT

    curl -L https://github.com

    exit
  
## Install a VNC Server and GUI

Here is where you have a lot of options.  These steps work for me, there are probably better ones.  The gist of these steps for tightvncserver came from [this guide here](https://www.digitalocean.com/community/tutorials/how-to-install-and-configure-vnc-on-ubuntu-20-04).

SSH to your VM

    gcloud compute ssh zap-vm --project=$PROJECT

Install dependancies.  When prompted for a display manager your choice doesnt matter. I did not use GNOME or other GUI's due to their extra memory and CPU requirements, I would like to keep my VM small.

    sudo apt update

    sudo apt install xfce4 xfce4-goodies firefox tightvncserver

Install `owasp-zap`. There are a lot of ways to [install this](https://software.opensuse.org/download.html?project=home%3Acabelo&package=owasp-zap). Doing it this way ensures you get the latest official release.  Using `snap` will not give you the latest release (in my case, snap gave me 2.9, but this option gave me 2.10).

    echo 'deb http://download.opensuse.org/repositories/home:/cabelo/xUbuntu_20.04/ /' | sudo tee /etc/apt/sources.list.d/home:cabelo.list
    
    curl -fsSL https://download.opensuse.org/repositories/home:cabelo/xUbuntu_20.04/Release.key | gpg --dearmor | sudo tee /etc/apt/trusted.gpg.d/home_cabelo.gpg > /dev/null

    sudo apt update

    sudo apt install owasp-zap

Start VNC Server, the password you enter will be what you use on the VNC Client to connect to this VM, so dont forget it.  You do not need a view-only password.

    vncserver

Stop VNC and create a config file.

    vncserver -kill :1

    mv ~/.vnc/xstartup ~/.vnc/xstartup.bak

Add the following contents to a new config file by opening a new file with VIM.

    vim ~/.vnc/xstartup

    #!/bin/bash
    xrdb $HOME/.Xresources
    startxfce4 &

Make it executable.

    chmod +x ~/.vnc/xstartup

Start the VNC Server and then exit.

    tightvncserver :1 -geometry 1700x1100 -depth 24

    exit

Connect to the server, creating a tunnel from your local box to the VM.  Also use the `-- -X` SSH arg so that x11 forwarding will work (required for the ZAP Proxy JAVA GUI).

    gcloud compute ssh zap-vm --zone $ZONE --project $PROJECT --ssh-flag "-L 5901:localhost:5901" -- -X

## Setup Client for VNC

Do these steps on your local workstation, not in the cloud.

1. Install TightVNC (client only)

1. If you are on Windows, and installed TightVNC there, and, you did all of the above on WSL2 (gcloud commands), make it so VNC ports get routed correctly to WSL2.

    1. Using Powershell, **as admin**.  
    
            $wslIp=(wsl -d Ubuntu -e sh -c "ip addr show eth0 | grep 'inet\b' | awk '{print `$2}' | cut -d/ -f1")

            netsh interface portproxy add v4tov4 listenport="5900" connectaddress="$wslIp" connectport="5900"

1. Start TightVNC.

1. In the *Remote Host:* field enter `localhost::5901`.  Make sure in the previous section you still have your ssh tunnel running.

1. You should now be on your VM in GCP, with a GUI (xfce) and a secure tunnel with no public IP's.  Pat yourself on the back.

    1. Once connected...open a terminal
    1. Enable xforwarding `xhost +`
    1. Start ZAP Proxy `owasp-zap`
    1. To maximize the ZAP window, click on the window border and hold ALT + right click + drag.

1. Start your research, hunting, hacking!