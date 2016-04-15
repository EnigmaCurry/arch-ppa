arch-ppa
========

arch-ppa is a tool that easily creates and maintains your own Arch
Linux package repositories. Kind of like the Personal Package Archives
(PPA) that Ubuntu has, but way easier.

The [Arch User Repository](https://aur.archlinux.org/) (AUR) is
convenient, has tons of software, is generally awesome, but is
inherently insecure. Anyone can upload anything they want to the
AUR. This is why I don't like to use AUR helpers like `yaourt` or
`pacaur`. Using the AUR with a helper requires you to be diligent in
reviewing the PKGBUILDs it downloads, in order to make sure it doesn't
include things like viruses or trojans, or downloading from a weird
URL.

I wanted a way to maintain my own repository of PKGBUILDs, downloaded
from the AUR, that I have manually verified. Building and installing
packages built from those pre-verified PKGBUILDs resolves the
insecurity of the AUR in my mind. This gives me the full power of the
AUR, but allows me to automate my package installs in a way that I
never felt comfortable with before. Seriously, why does pacaur have a
`--noconfirm` option? That's scary. 

The packages this tool builds can be hosted as a regular arch
repository, which you put into your `/etc/pacman.conf`. The added
convenience here is that although the packages came from the AUR, your
clients install it through regular-old pacman.


Usage
-----

Clone this repo somewhere. Everything will be self contained in this
directory wherever you put it.

arch-ppa should not be run as root, but the user needs to have sudo
privileges as the underlying devtools need it. (If you know how to
make arch-nspawn create files with the current uid, please let me
know.)

Run setup:

    ./arch-ppa setup

The setup installs a few dependencies like `devtools` and `git`. It
also creates a chroot directory which is a container that will be used
to build packages in a completely clean environment using
`systemd-nspawn`.

Add packages from the AUR:

    ./arch-ppa add cower curlbomb pasystray
	
This downloads PKGBUILDs from the AUR for the listed packages: cower,
curlbomb, pasystray, as well as all of their AUR dependencies, and
placed into the `src` directory. You can manually put any PKGBUILDs
you have into the `src` directory; they don't have to be from the
AUR. Note that any PKGBUILD that lists a dependency of another
package, that is not found in one of the arch repositories, needs to
have it's own PKGBUILD in the `src` directory too. (The `add` command
does this for you automatically, thanks to `cower -d -d`)

Build everything:

    ./arch-ppa clean ryan
    ./arch-ppa build ryan

The build process operates on a single repository, in this example
called `ryan`. You can maintain several repositories, each containing
different sets of packages. Just make sure to give each repository a
unique name.

The clean process removes the repository directory containing all the
built packages. It also deletes the chroot for the repository from the
`chroot` directory.

The build process creates a new package repository called `ryan` (or
whatever you called yours.) It finds PKGBUILD files in the `src`
directory and figures out the dependency chain and builds all the
packages in the correct order. Additionally, you can specify
individual package names after the repository name if you only wish to
build certain packages. If you do specify package names, make sure to
include all dependencies, as they will not be included otherwise.

The repository directory can be listed in your /etc/pacman.conf like this:

    [ryan]
	Server = file:///home/ryan/git/arch-ppa/ryan
	SigLevel = Required TrustedOnly
	
This is the full path to the ryan repository just created. Replace the
name in brackets with your chosen repository name and use the path
appropriate for your machine. Run `pacman -Sy` and you should see
pacman synchronize with the new repository name. Alternatively, upload
the directory to a webserver to share it with all your friends.

The SigLevel option specifies how pacman should trust our
repository. `Required TruestedOnly` is a strict rule that the key must
be in the local pacman keyring and be assigned a trust level. Pacman
will usually download the key without a problem, but you will still
need to locally sign the key to trust it. 

See the next section if you're having problems with package signatures
not working.

Mini gpg tutorial
-----------------
View your key information:

    gpg -K 

This should output something like this:

    /home/ryan/.gnupg/pubring.kbx
    -----------------------------
    sec   rsa2048/4BAACCF8 2016-04-15 [SC]
    uid         [ultimate] test guy <ryantest@enigmacurry.com>
    ssb   rsa2048/C22BDAA5 2016-04-15 [E]

My public key ID is 4BAACCF8. Always omit the part before the
slash. If it didn't output any key information at all, this means you
don't have a key yet. If that's the case, create one and follow the
prompts:

    gpg --gen-key

Send your public key to the keyserver (replace with your ID):

    gpg --send-keys 4BAACCF8

On each machine you plan to use your package repository, run the
following to import the key and to locally sign it (meaning to trust
it from pacman's perspective. Like before, replace with your key ID):

    sudo pacman-key -r 4BAACCF8
	sudo pacman-key --lsign-key 4BAACCF8

If you don't sign the key, pacman will complain that your packages are
not trusted.
