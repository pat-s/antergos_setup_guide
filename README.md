This guide does not claim to be complete and only reflects my personal view on how to setup a working Arch Linux system tailored to data science, R and LaTeX usage. 
I you have suggestions for modifications, please open an issue.


# Arch Linux setup guide

I recommend using [Antergos | Your Linux. Always Fresh. Never Frozen.](https://antergos.com). 
Officially its a distribution but most people refer to it as a graphical installer for Arch Linux.
It comes with the choice of 6 different desktop systems.
Choose a desktop that suites you. Choose wisely. Your desktop is responsible for the look, feel and standard applications of your installation. 
See [10 Best Linux Desktop Environments And Their Comparison | 2018 Edition](https://fossbytes.com/best-linux-desktop-environments/) for some inspiration.
What makes Antergos a distribution is that it also comes with its own libraries maintained by the Antergos developers that you can install.

Create a installer by following [Create a working Live USB | Antergos Wiki](https://antergos.com/wiki/uncategorized/create-a-working-live-usb/).

Make sure to check out [Frequently asked questions - ArchWiki](https://wiki.archlinux.org/index.php/Frequently_asked_questions) and [Arch compared to other distributions - ArchWiki](https://wiki.archlinux.org/index.php/Arch_compared_to_other_distributions) to get a better understanding of Arch. 

# 1. Setting up the partitions

There are many concepts on how to partition a Linux system correctly. The following reflects my current view:

1. Select "Manual" partitioning when being prompted
2. Create a SWAP partition that is >= your amount of RAM (e.g. for 16 GB RAM use 16.5 GB partition size). Format: Linux Swap
3. Create a 1 GB partition. Mount point: `/boot`. Format: `ext4`
2. Create a 50 GB partition for "root". Mount point: `/`. Format: `ext4`
3. With the remaining space create "home". Mount point: `/home`. Format: `ext4`


# 2. Install package manager

I prefer `trizen`. 
Here is a list ([AUR helpers - ArchWiki](https://wiki.archlinux.org/index.php/AUR_helpers)) comparing more alternatives. 
(Scroll to the bottom.)

Install `trizen`: 

```
git clone https://aur.archlinux.org/trizen-git.git
cd trizen-git
makepkg -si
```

(We need the git version as it has a fix for the wrapper functions below that is not yet included in the latest release when writing this guide.)

In `~/.config/trizen/trizen.conf` set "noedit" to "1" to not being prompted to edit source code on every install.

(Optional) [GitHub - gavinlyonsrepo/cylon: Updates, maintenance, backups and system checks in a TUI menu driven bash shell script for an Arch based Linux distro](https://github.com/gavinlyonsrepo/cylon) -> Wrapper around `trizen`.

## 2.1 (Optional) Set up and configure zsh 

The `zsh` (Z shell) is an alternative to the default installed `bash`(Bourne-again Shell). It has several advantages (file globbing, visual appearance, etc.).

To set it up, do the following (see [GitHub - sorin-ionescu/prezto: The confi)guration framework for Zsh](https://github.com/sorin-ionescu/prezto):

```
trizen zsh

zsh

git clone --recursive https://github.com/sorin-ionescu/prezto.git "${ZDOTDIR:-$HOME}/.zprezto"

setopt EXTENDED_GLOB
for rcfile in "${ZDOTDIR:-$HOME}"/.zprezto/runcoms/^README.md(.N); do
  ln -s "$rcfile" "${ZDOTDIR:-$HOME}/.${rcfile:t}"
done

chsh -s /bin/zsh
```

Afterwards set up some custom wrapper functions (`aliases`) around `trizen` to simplify usage:

In your `~/.zshrc`, append the following line: 

```
source "${ZDOTDIR:-$HOME}/.zprezto/pac.zsh"
```

Next, create the following file `.zprezto/aur.zsh`:

```
pac () {
  case $* in
    install* ) shift 1; cd ~ && trizen -S "$@" --movepkg-dir=pkgs;;
    get* ) shift 1; cd ~ && trizen -G --aur "$@" ;;
    remove* ) shift 1; cd ~ && trizen -R --aur "$@" ;;
    search* ) shift 1; cd ~ && trizen -s "$@" --movepkg-dir=pkgs;;
    update-git* ) shift 1; cd ~ && trizen -Syu --devel --show-ood --movepkg-dir=pkgs;;
    update* ) shift 1; cd ~ && trizen -Syu --needed --show-ood --movepkg-dir=pkgs;;
    * ) echo "Invalid choice, see ~/.zpreto/pac.zsh for available commands." ;;
  esac
}
```

Open a new terminal window and the function `pac` should be available now.
You can now do call `pac` with all arguments listed above (`install`, `search`, etc.). 
Check [GitHub - trizen/trizen: Lightweight AUR Package Manager](https://github.com/trizen/trizen#usage) for an explanation of the created functions.

Some explanations:

* `pac install <pkg>`: Will install the specified package (if it exists) and moves it into `~/pkgs`.
* `pac search <pkg>`: Executes a search with the specified `<pkg>` returning all matches. You can then type a number of the package you want to install. It will be moved to `~/pkgs`.
* `pac update`: Updates all installed packages (from both Arch repos and AUR). Shows packages which are marked as `out-of-date` by the community.
* `pac update-git`: Updates all packages installed from `git`. Note: These are usually build from source and some may take some time to install. Don't do that daily.

**Note:** Git packages will never update automatically as they are just a snapshot build of the (at the time of installation) most recent state of the respective repository. 
So think twice if you need a git package as it is in your responsibility to update it. 
I usually have `rstudio-desktop-git` installed to have the latest features of RStudio as the release cycles for the stable version are quite long.

One argument of the wrapper functions to explain explicitly is `--movepkg`. 
It will move all built packages `<package.tar.xz>` into `~/pkgs`.
This has the advantage that you do not need to rebuild a package that took a long time to install if you want to re-install it - just do a `pacman -U ~/pkgs/<package.tar.xz>`.


## 2.2 Enable parallel compiling

Compiling packages from source can take some time. 
To speed up the process by enabling parallel compiling, set the `MAKEVARS` variable in `/etc/makepkg.conf`:

`MAKEFLAGS="-j$(nproc)"`

This will use all available cores on your machine for compiling.

## 3. System related

### 3.1 Install system libraries

For the following install calls, you can either use `trizen` or (if you added the `zsh` wrapper functions above) `pac`.
While calling `trizen <package>` will first do a search in AUR and then install the package, the complementary function for this would be `pac search <package>`. Calling `pac install` will directly install the given package.

**Never install python libraries via `pip` as otherwise there will be conflicts with AUR packages depending on certain libraries!**

Always install them via your package manager, e.g. for `numpy`: `trizen python-numpy`

Python Modules for QGIS:

```
pac install python-gdal python-gdal python-yaml python-yaml python-jinja
python-psycopg2 python-owslib python-numpy python-pygments
```

Other important system libraries: 

* `pac install gdal2`
* `pac install udunits`
* `pac install postgis`
* `pac install jdk8-openjdk openjdk8-src`(jdk9 still has problems with some R packages)
* `pac search texlive` (install 1-19)

### 3.2 Apps

Apps are (of course) completely opinionated.
Feel free to try out my favorite ones or stick with your favorites :smile: 

Messaging: `pac install franz`
Mail: `pac install mailspring`
Notes: `pac install boostnote`
Reference Manager: `pac install Jabref`
Google Drive: `pac install insync`
Dropbox: `pac install dropbox-nautilus`
GIS: `pac install qgis` (careful, takes > 30min - 1h to compile)

Tip: You can install both `QGIS2` and `QGIS3` and switch between them. 
For this you need to build both once with the wrapper functions from section 2.
Afterwards, you can switch between them using `pacman -U <package_source>`. 
E.g. to install `QGIS2` after you installed `QGIS3` :

```
pacman -U ~/pkgs/qgis-ltr-2.18.17-1-x86_64.pkg.tar
```

SAGA: `pac install saga-gis`
Skype: `pac install skypeforlinux-preview-bin`
Screenshot tool: `pac install shutter`
Image viewer: `pac install xnviewmp`
Virtualbox: [VirtualBox – wiki.archlinux.de](https://wiki.archlinux.de/title/VirtualBox)
Terminal: `pac install tilix`
Browser: `pac install vivaldi-snapshot`
Dock: `pac install latte-dock` (optional if you prefer a dock over the default taskbar)
Twitter client: `pac install corebird` 

## 4. R 

### 4.1 General

For fast package installation using `ccache`, put the following into `~/.R/Makevars`:

```r
CXXFLAGS=-O3 -mtune=native -march=native -Wno-unused-variable -Wno-unused-function

CXXFLAGS=-O3 -mtune=native -march=native -Wno-unused-variable -Wno-unused-function -DBOOST_PHOENIX_NO_VARIADIC_EXPRESSION

# Eddelbuettel blog
VER=
CCACHE=ccache
CC=$(CCACHE) gcc$(VER)
CXX=$(CCACHE) g++$(VER)
C11=$(CCACHE) g++$(VER)
C14=$(CCACHE) g++$(VER)
FC=$(CCACHE) gfortran$(VER)
F77=$(CCACHE) gfortran$(VER)
```

Additionally, install `ccache` on your system: `trizen ccache`.
See [this blog post](http://dirk.eddelbuettel.com/blog/2017/11/27/#011_faster_package_installation_one) by Dirk Eddelbuettel as a reference.

### 4.2 RStudio

Use `pac search rstudio` and pick your favorite release channel.

## 4.3 Packages

Install `usethis` and then call `usethis::browse_github_pat()`.
Follow the instructions to set up a valid `GITHUB_PAT` environment variable that will be used for installing packages from Github.

### 4.3.1 Task view "Spatial"

Required system libraries: 

* jq (`pac install jq`)
* udunits (`pac install udunits`)
* fortran (`pac install gcc-fortran`)
* v8-3.14 (`pac install v8-3.14`)

Some packages (V8, geojsonlite, etc.) require the `V8` [package](https://github.com/jeroen/V8) which depends on the oudated `v8-314` library. 

### 4.3.2 Task view "Machine Learning"

Required system libraries: 

* `pac install nlopt`

## 5. Mounting servers

There are multiple ways to do so ([Auto-mount network shares (cifs, sshfs, nfs) on-demand using autofs | Patrick Schratz](https://pat-s.github.io/post/autofs/), [fstab - ArchWiki](https://wiki.archlinux.org/index.php/fstab)).

Here  is an example of a `fstab` setup for a `sshfs` (to Linux server) and `cifs` (to Windows server) mount.
Append those lines to `/etc/fstab`; don't overwrite the existing content as this will result in boot errors otherwise!

```
# sshfs
sshfs#<username>@<ip>:<remote mount point> <local mount point> fuse        reconnect,idmap=user,transform_symlinks,identityFile=~/.ssh/id_rsa,allow_other,cache=yes,kernel_cache,compression=no,default_permissions,uid=1000,gid=100,umask=0,_netdev,x-systemd.after=network-online.target   0 0

# cifs
//<ip>/<remote mount point> <local mount point> cifs        credentials=/etc/.smbcredentials.txt,uid=1000,file_mode=0775,dir_mode=0775,gid=100,sec=ntlm,vers=1.0,dom=ads.uni-jena.de,forcegid,_netdev,x-systemd.after=network-online.target 0 0
```

**Notes:**

* (cifs) Depending how new the Windows server is, you do not need `vers=1.0`.
* (cifs) Store your login credentials for the windows server in a file, e.g. `/etc/.smbcredentials.txt` with contents being `username = <username>` and `password = <password>`.
* (sshfs) Copy `.ssh/id_rsa` to `root/.ssh/` as the mount will be executed by the root user.
* (cifs) Install the Arch Linux kernel headers for the `cifs` package to work (and later on for Virtualbox): `trizen linux-headers`

Reboot.

## 6. Desktop related

### KDE

If you want to use an automatic login to a VPN and the networkmanager-daemon (e.g. Openconnect) does not store your password, try the `network-manager-applet` package. 
It is the GNOME network-manager and has for some reason no problems with storing the password.

## 7. Battery life optimization

Although the Linux kernel has a lot of power saving options, they are not all enabled by default.

There are two main power optimization tools:

* Powertop
* TLP

I prefer `tlp` as `powertop`often causes trouble with USB devices going into sleep mode. 
Also, applying the changes on boot is easier with `tlp`.

`pac install tlp`

Then follow the instructions on [TLP - ArchWiki](https://wiki.archlinux.org/index.php/TLP) to configure it correctly.

`powertop`though is useful to check the applied settings. Do `sudo powertop` and go to the "tunables" section and check if most settings are "GOOD" (most are "BAD" before applying `tlp`).


## 8. Other stuff

### arara

[GitHub - cereda/arara: arara is a TeX automation tool based on rules and directives. It gives you subsidies to enhance your TeX experience.](https://github.com/cereda/arara)
An automatization tool for TeX: `pac install arara-git`.late

### Perl modules needed for latexindent.pl

`latexindent` is a library which automatically indents your LaTeX document during compilation: [GitHub - cmhughes/latexindent.pl: Perl script to add indentation (leading horizontal space) to LaTeX files; as of V3.0, the script can also modify line breaks. The script is customisable through its YAML interface.](https://github.com/cmhughes/latexindent.pl)

`aur install perl-log-dispatch`
`aur install perl-dbix-log4perl`
`pac install perl-file-homedir`
`pac install perl-unicode-linebreak`
