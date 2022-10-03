# (forked) openconnect-sso

Wrapper script for OpenConnect supporting Azure AD (SAMLv2) authentication to Cisco SSL-VPNs

---

## Background

the VPN scenario i'm personally shooting for is:
- Windows 11 (currently v 22H2, build: 22621.521)
- Cisco AnyConnect
- interactive Azure AD MFA via embedded browser
- CSD hostscan response support

i created this fork to get the build process working on Windows along with incorporating @impynutz' "CSD hostscan" response logic ([see thread](https://github.com/vlaci/openconnect-sso/issues/35)).

fyi, i assume the python build output is multiplatform, i.e. build on linux run on Windows or vice versa... yet the final desire of creating a self contained Windows .exe does require running pyinstaller on Windows within the folder of the build output... if you run pyinstaller on linux it is designed to generate a linux format binary... so working back from there, it seems most streamlined to have the build itself running on Windows as well.

as quick background on the need here: VPN administrators can configure the anyconnect auth flow to direct the native cisco vpn client to execute a scan of your local computer and only allow the connection if the scan results are valid. this step is intended to confirm things like a well patched OS and presence of approved antivirus software. the additional script logic we're doing here submits a personally constructed response file to satisfy this scan request. in my environment, i've found the required contents of this scan result to be remarkably trivial ([example below](#sample-hostscan-data-file)).

## Prerequisite installs for building on Windows:
(there's many ways to get python and this toolchain working, this is merely what worked for me and seemed canonically minimal)<br/>
(mild assumption you're using [pwsh](https://learn.microsoft.com/en-us/powershell/scripting/install/installing-powershell-on-windows) for your interactive command line)

- [chocolatey](https://docs.chocolatey.org/en-us/choco/setup) (must be elevated!): `Set-ExecutionPolicy Bypass -Scope Process -Force; [System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072; iex ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1'))`
  - i'm using choco to obtain make.exe since it'd be nice to eventually do this all in a github action and the windows runners appear to natively support `choco install`
- [`choco install make`](https://community.chocolatey.org/packages/make)
- python v3.10 via https://python.org exe installer
  - there's probably a winget entry that would work but there's so many of them i couldn't tell which one would be best
  - for the record, the one significant python distro i did have trouble with was v3.10 from the Windows Store... i ran into lxml package missing .h files during the makefile run.
  - i chose options: "all users" path, "compile standard library" (hoping this means lxml works) and enabled the "py" convenience launcher
- [poetry](https://python-poetry.org/docs/#installation) (used in Makefile): `(Invoke-WebRequest -Uri https://install.python-poetry.org -UseBasicParsing).Content | py -`
  - and then add poetry.exe to your path: `%AppData%\Roaming\Python\Scripts`
- [`pip install pre-commit`](https://pre-commit.com/)
  - used in Makefile... weird this installs to %APPDATA%\Roaming\Python\Python310\Scripts vs the Script folder poetry installed to

* [`winget install git.git -i`](https://git-scm.com/download/win)
  * -i = interactive so we can choose personally preferred git options like which tty console to use
* lastly of course, `clone` this repo to a local folder and `cd` into it

now we can successfully run make targets!
  ```
  > make dev
  ```

the poetry build process within the Makefile will create a python "[virtual environment](https://docs.python.org/3/tutorial/venv.html#creating-virtual-environments)" in ".venv" folder...
so the final `make dev` build output winds up in .venv/Scripts...
there will be an openconnect-sso.cmd created which launches the openconnect-sso python file... a sample run command line would be as simple as:
  ```
  > openconnect-sso.cmd -s {hostname}
  ```

we now have a working pipeline to run and tweak openconnect-sso on Windows, sweet!

as a reminder, openconnect-sso should popup a webview GUI for you to MFA with your provider (Azure AD in my case)...
and after slurping up the pertinent auth handshake output it will eventually launch a command line like below:
  ```
  > openconnect.exe --cookie-on-stdin --servercert blah
  ```

Note the whole idea of this python script is to accomplish the authentication and then hand that off to the openconnect executable to actually create the IP tunnel.  While this does mean we're dealing with two different tool stacks, I welcome the greater flexibility to enhance and build the python script on Windows versus how one must perform the openconnect C source build exclusively on linux (Fedora specifically favored). At this lower level, openconnect is (hopefully) feature complete and we can keep up with any future changes to our vpn flow entirely within this script based front end.  To me this is a good separation of concerns, especially when you see all the necessary debate that must go on within the openconnect community to maintain compatibility with all the vpn client scenarios it aspires to support ([example thread](https://gitlab.com/openconnect/openconnect/-/issues/437)).

### Windows sudo 
i've removed sudo from the latest release... it's less overhead to simply start the entire openconnect-sso process elevated in the first place and eliminate this dependency... also even more importantly, it allows CTRL-C (SIGINT) to be caught by openconnect.exe and thereby clean up gracefully... for example running the "[vpnc-script-win.js](https://github.com/Beej126/openconnect-sso-windows-csd/blob/main/misc/vpnc-script-win.js)" script which i've customized to remove the default gateway entry that was created in your route table for the vpn

---
## Creating Self-Contained Windows .exe:

"pyinstaller" bundles all the necessary files into a single exe file...
when you see its runtime output you realize it's doing A LOT of work!, including all the Qt5 dlls for webview UI, etc
  which makes some sense why the final .exe winds up over 100megs!
  and why it takes a few moments upon initial launch to unpack everything to %temp% before it starts executing

to get this piece running we just have a couple more pre-reqs:
- `cd .venv\Scripts`
- `.\activate.ps1`
- `pip install pyinstaller`
  - confirm pyinstall.exe is now present in the .venv/scripts folder, otherwise the activate didn't work, try again
- `cp ..\..\openconnect_sso\browser\user.js .`
    - for the logic to be happy we need a "user.js" file present in the right location at runtime... this might be intended for us to inject our own javascript into the MFA webview to automatically populate credentials, need to explore... the file @impynutz provided is blank so far except for "// ==UserScript==" which suggests [Greasemonkey](https://en.wikipedia.org/wiki/Greasemonkey) if you're familiar
- and one last tweak for Windows due to this: https://github.com/pyinstaller/pyinstaller/wiki/Recipe-Multiprocessing
  - edit `.venv/scripts/openconnect-sso` and make it look like this:
    ```python
    import sys
    import multiprocessing
    from openconnect_sso.cli import main

    if __name__ == '__main__':
        multiprocessing.freeze_support()
        sys.exit(main())
    ```
  - it looks like this code correponds to the $repo/openconnect_sso/cli.py file but applying these edits there did NOT propogate as expected to the .venv build output...any help here is appreciated.

to run pyinstaller:
  ```
  > cd .venv/Scripts
  > .\pyinstaller.exe --onefile --clean --add-data "user.js;openconnect_sso/browser" openconnect-sso
  ```
(fyi, the "--add-data" syntax is "source;dest")

this will output our final openconnect-sso.exe to .venv/scripts/dist folder!

---
### sample hostscan-data file:
```
endpoint.os.version="Windows 10";
endpoint.os.architecture="x64";
endpoint.os.processor_level="unknown";
endpoint.device.protection="none";
endpoint.device.protection_version="4.8.01064";
endpoint.device.protection_extension="4.3.890.0";
endpoint.device.hostname="winpc";
endpoint.device.id="To Be Filled By O.E.M.";
endpoint.os.hotfix["KB4497165"]="true";
endpoint.os.hotfix["KB4498523"]="true";
endpoint.os.hotfix["KB4503308"]="true";
endpoint.os.hotfix["KB4508433"]="true";
endpoint.os.hotfix["KB4509096"]="true";
endpoint.os.hotfix["KB4515383"]="true";
endpoint.os.hotfix["KB4515871"]="true";
endpoint.os.hotfix["KB4516115"]="true";
endpoint.os.hotfix["KB4520390"]="true";
endpoint.os.hotfix["KB4521863"]="true";
endpoint.os.hotfix["KB4523786"]="true";
endpoint.os.hotfix["KB4524569"]="true";
endpoint.pfw["283"]={};
endpoint.pfw["283"].exists="true";
endpoint.pfw["283"].description="Windows Firewall";
endpoint.pfw["283"].version="10.0.18362.1";
endpoint.pfw["283"].enabled="ok";
endpoint.am["362"]={};
endpoint.am["362"].exists="true";
endpoint.am["362"].description="Windows Defender";
endpoint.am["362"].version="4.18.1902.5";
endpoint.am["362"].activescan="ok";
endpoint.am["362"].lastupdate="12179";
endpoint.am["362"].timestamp="1574131324";
```

---

## Original Docs

## Installation

### Using pip/pipx

A generic way that works on most 'standard' Linux distributions out of the box.
The following example shows how to install `openconect-sso` along with its
dependencies including Qt:

```shell
$ pip install --user pipx
Successfully installed pipx
$ pipx install "openconnect-sso[full]"
⣾ installing openconnect-sso
  installed package openconnect-sso 0.4.0, Python 3.7.5
  These apps are now globally available
    - openconnect-sso
⚠️  Note: '/home/vlaci/.local/bin' is not on your PATH environment variable.
These apps will not be globally accessible until your PATH is updated. Run
`pipx ensurepath` to automatically add it, or manually modify your PATH in your
shell's config file (i.e. ~/.bashrc).
done! ✨ 🌟 ✨
Successfully installed openconnect-sso
$ pipx ensurepath
Success! Added /home/vlaci/.local/bin to the PATH environment variable.
Consider adding shell completions for pipx. Run 'pipx completions' for
instructions.

You likely need to open a new terminal or re-login for the changes to take
effect. ✨ 🌟 ✨
```

If you have Qt 5.x installed, you can skip the installation of bundled Qt version:

``` bash
pipx install openconnect-sso
```

Of course you can also install via `pip` instead of `pipx` if you'd like to
install system-wide or a virtualenv of your choice.

### On Arch Linux

There is an unofficial package available for Arch Linux on
[AUR](https://aur.archlinux.org/packages/openconnect-sso/). You can use your
favorite AUR helper to install it:

``` shell
yay -S openconnect-sso
```

### Using nix

The easiest method to try is by installing directly:

```shell
$ nix-env -i -f https://github.com/vlaci/openconnect-sso/archive/master.tar.gz
unpacking 'https://github.com/vlaci/openconnect-sso/archive/master.tar.gz'...
[...]
installing 'openconnect-sso-0.4.0'
these derivations will be built:
  /nix/store/2z47740z1rr2cfqfin5lnq04sq3c5xjg-openconnect-sso-0.4.0.drv
[...]
building '/nix/store/50q496iqf840wi8b95cfmgn07k6y5b59-user-environment.drv'...
created 606 symlinks in user environment
$ openconnect-sso
```

An overlay is also available to use in nix expressions:

``` nix
let
  openconnectOverlay = import "${builtins.fetchTarball https://github.com/vlaci/openconnect-sso/archive/master.tar.gz}/overlay.nix";
  pkgs = import <nixpkgs> { overlays = [ openconnectOverlay ]; };
in
  #  pkgs.openconnect-sso is available in this context
```

... or to use in `configuration.nix`:

``` nix
{ config, ... }:

{
  nixpkgs.overlays = [
    (import "${builtins.fetchTarball https://github.com/vlaci/openconnect-sso/archive/master.tar.gz}/overlay.nix")
  ];
}
```

### Windows *(EXPERIMENTAL)*

Install with [pip/pipx](#using-pippipx) and be sure that you have `sudo` and `openconnect`
executable commands in your PATH.

## Usage

If you want to save credentials and get them automatically
injected in the web browser:

```shell
$ openconnect-sso --server vpn.server.com/group --user user@domain.com
Password (user@domain.com):
[info     ] Authenticating to VPN endpoint ...
```

User credentials are automatically saved to the users login keyring (if
available).

If you already have Cisco AnyConnect set-up, then `--server` argument is
optional. Also, the last used `--server` address is saved between sessions so
there is no need to always type in the same arguments:

```shell
$ openconnect-sso
[info     ] Authenticating to VPN endpoint ...
```

Configuration is saved in `$XDG_CONFIG_HOME/openconnect-sso/config.toml`. On
typical Linux installations it is located under
`$HOME/.config/openconnect-sso/config.toml`

## Development

`openconnect-sso` is developed using [Nix](https://nixos.org/nix/). Refer to the
[Quick Start section of the Nix
manual](https://nixos.org/nix/manual/#chap-quick-start) to see how to get it
installed on your machine.

To get dropped into a development environment, just type `nix-shell`:

```shell
$ nix-shell
Sourcing python-catch-conflicts-hook.sh
Sourcing python-remove-bin-bytecode-hook.sh
Sourcing pip-build-hook
Using pipBuildPhase
Sourcing pip-install-hook
Using pipInstallPhase
Sourcing python-imports-check-hook.sh
Using pythonImportsCheckPhase
Run 'make help' for available commands

[nix-shell]$
```

To try an installed version of the package, issue `nix-build`:

```shell
$ nix build
[1 built, 0.0 MiB DL]

$ result/bin/openconnect-sso --help
```

Alternatively you may just [get Poetry](https://python-poetry.org/docs/) and
start developing by using the included `Makefile`. Type `make help` to see the
possible make targets.
