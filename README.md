# debian-pihole-node
Instructions for a minimalist Debian Pi-hole server (e.g., running as a VirtualBox virtual machine)

## Why?

- I want to use Pi-hole.
- I don't want to use the Docker image because I want a bit more control over the settings.
- Debian is lightweight and stable, which is good for being able to set the RAM quite low and keep things still functional. I like Fedora but I wouldn't use it here, it's a bit heavy and SELinux will cause trouble for Pi-hole.

## Download Installer

These instructions are based on `debian-11.4.0-amd64-DVD-1.iso` from https://mirror.aarnet.edu.au/pub/debian-cd/current/amd64/iso-dvd/

## Virtual Machine Configuration

_(Based on recommendations provided by Fedora Cockpit Virtual Machines)_

- SSD: 10GiB
- RAM: 1GiB

When using Fedora Cockpit:

- Do not run the VM right away after creating! Make sure you Edit it first.
- Remove the default Network entry
- Create a new Network entry set to **Direct Attachment to en1234** as per https://unix.stackexchange.com/questions/564353/passthrough-of-vms-to-local-network-bridging-fedora-31-server

## Installation of Debian

- Use normal install, not graphical install, since we have only 512MB RAM
- Continent = Oceania _(obviously, feel free to customise according to your needs)_
	- Newer versions of Debian might not ask this
- Country = Australia _(obviously, feel free to customise according to your needs)_
  	- Or Country = Ireland if operating in Ireland
- Keymap = American English (default)
- DHCP will automatically run, just use `<Go Back>` to force Manual config
- Set Name server address to default, I'm sure we can come back for it later
- Set Hostname (e.g., `sushi`)
- Domain name set to blank (default)
- Set root password
- Username (e.g., `sushichef`)
- Set password
- If prompted: Timezone = New South Wales _(obviously, feel free to customise according to your needs)_
- Guided - use entire disk (default: doesn't use LVM)
- All files in one partition
- Scan extra installation media? = `<No>`
- Network mirror = `<Yes>`
  	- And then select the server appropriate for your location
- Package usage survey = `<No>`
- Choose software to install:
	- `[*] SSH server`
	- `[*] Standard system utilities`
	- nothing else!
- Install the GRUB boot loader to your primary drive? `<Yes>`
	- VirtualBox: Choose `/dev/sda`
	- KVM/Cockpit: Choose `/dev/vda`
   	- Newer versions of Debian might not ask this


## Configuration Part 1 - set up networking

Follow instructions at https://www.snel.com/support/how-to-configure-static-ip-on-debian-10/

## Configuration Part 2 - set up package manager

NOTE: This step is NOT necessary if the Network Mirror was set up in the initial installation of Debian above.

Login to root, `su -`, and then `nano /etc/apt/sources.list`

Remove the line about the cdrom.

This is also where you can change the mirror (e.g., to `http://mirror.aarnet.edu.au/debian/` in Australia or `http://ftp.no.debian.org/debian/` in Northern Europe e.g., Ireland, Norway).


## Configuration Part 3 - install packages as root

```bash
apt-get update && apt-get upgrade
apt install git sudo
```

Now edit `/etc/sudoers` to add:

```
sushichef ALL=(ALL:ALL) ALL
```

To do so, do not use nano, do the following:

```zsh
export EDITOR=nano
visudo
```

Now go back to the main user! (Get out of root)

## Configuration Part 4 - setup Pi-hole

As per https://docs.pi-hole.net/main/basic-install/#one-step-automated-install

```zsh
git clone --depth 1 https://github.com/pi-hole/pi-hole.git Pi-hole
cd "Pi-hole/automated install/"
sudo bash basic-install.sh
```

During setup:

- Set upstream DNS provider to Quad9, unfiltered, no DNSSEC.
- Allow the default blocklist (StevenBlack)
- Install the admin interface, web server, and logging
- Privacy mode for FTL = 0 (show everything)
- Don't worry about the Admin Webpage login password, you can reset it later

## Configuration Part 5 - HTTPS

Generate self-signed certificate (adapted from https://redmine.lighttpd.net/projects/1/wiki/HowToSimpleSSL, https://discourse.pi-hole.net/t/enabling-https-for-your-pi-hole-web-interface/5771):

```
mkdir -p ~/selfsigned
cd ~/selfsigned
openssl req -new -x509 -keyout lighttpd.pem -out lighttpd.pem -days 99999 -nodes # just ENTER everything
sudo chown www-data -R ~/selfsigned
sudo apt-get install lighttpd-mod-openssl
```

Now, place the following into `/etc/lighttpd/conf-enabled/external.conf` ...

(Source: https://discourse.pi-hole.net/t/external-conf-not-being-loaded-in-2023-1/60543/2)

```
$HTTP["host"] == "sushi" {
  # Ensure the Pi-hole Block Page knows that this is not a blocked domain
  setenv.add-environment = ("fqdn" => "true")

  # Enable the SSL engine with a LE cert, only for this specific host
  $SERVER["socket"] == ":443" {
	ssl.engine = "enable"
	ssl.pemfile = "/home/sushichef/selfsigned/lighttpd.pem"
  }

  # Redirect HTTP to HTTPS
  $HTTP["scheme"] == "http" {
	$HTTP["host"] =~ ".*" {
	  url.redirect = (".*" => "https://%0$0")
	}
  }
}
```

Now add `mod_openssl` to `server.modules` in `/etc/lighttpd/lighttpd.conf`

Finally, `sudo service lighttpd restart`

On any clients using Edge (cf. https://www.tecklyfe.com/how-to-access-site-with-neterr_cert_invalid-error-in-chrome-and-edge/):

- Click anywhere on the page
- Type `thisisunsafe` manually (NOTE: copy-paste doesn't work)
- The page is now allowed


From Debian, you can reset the Pi-hole password at any time by running:

```bash
sudo pihole -a -p
```

Happy Debian+Pi-hole adventures! 🐧
