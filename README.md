# openconnect-sso

Wrapper script for OpenConnect supporting Azure AD (SAMLv2) authentication to Cisco SSL-VPNs

---

## Background

the VPN scenario i'm personally shooting for is cisco anyconnect, with Azure AD MFA running on Windows 11 (currently v 21H2, build: 22000.856).

i created this fork to get the build process working on Windows along with incorporating @impynutz' "CSD hostscan" response logic.

actually i don't think the build process really needs to run on Windows... theoretically we should be able to execute the final python script output from a linux build on Windows...

yet  the last step of creating a self contained .exe DEFINITELY does require running pyinstaller on Windows... if you run pyinstaller on linux it is designed to generate a linux format binary.

as quick background, normally the native cisco anyconnect client facilitates executing a scan of your local computer requested by your vpn server.
  this step is intended to confirm things like a well patched OS and antivirus installed.
  the CSD logic we're doing here primarily boils down to submitting your customized "hostscan-data" file to satisfy the cisco server request during auth handshake.


## Prerequisite installs:
- (optional) vscode: winget install vscode
- (optional) pwsh: winget install pwsh
- git for windows (-i = interactive so we can choose personally preferred git options like which tty console to use): winget install git.git -i
- chocolatey (must be elevated! i'm using choco to obtain make.exe since github windows runner navtively supports `choco install`): Set-ExecutionPolicy Bypass -Scope Process -Force; [System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072; iex ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1'))
- choco install make
- python v3.10: https://python.org... i installed for "all users" path, checked "compile standard library" (hoping this means lxml works) and enabled the "py" convenience launcher
- poetry (used in Makefile): (Invoke-WebRequest -Uri https://install.python-poetry.org -UseBasicParsing).Content | py -
  - add poetry.exe to path: %AppData%\Roaming\Python\Scripts
- pre-commit (used in Makefile... weird this installs to %APPDATA%\Roaming\Python\Python310\Scripts vs the Script folder poetry installed to): pip install pre-commit

now we can successfully run make targets!
  ```
  > make dev
  ```

the poetry build process within the Makefile will create a python "virtual environment" in ".venv" folder...
so the final `make dev` build output winds up in .venv/Scripts...
there will be an openconnect-sso.cmd created which launches the openconnect-sso python file... a sample run command line would be as simple as:
  ```
  > openconnect-sso.cmd -s {hostname}
  ```

we now have a working pipeline to run and tweak openconnect-sso on Windows, sweet!

as a reminder, openconnect-sso should popup a webview GUI for you to MFA with your provider (Azure AD in my case)...
and after gathering the auth output it will eventually launch a command line like this:
  ```
  > sudo openconnect --cookie blah --servercert blah
  ```

### Windows sudo
fyi, here's a very nice sudo for windows: https://github.com/mattn/sudo ... it's written in go and takes some effort to capture the stdin/stdout of the hidden elevated process and pipe that back to your initial console so it seems a lot more equivalent to linux sudo

frankly, i'm thinking it's less overhead to simply start the entire openconnect-sso process elevated in the first place and eliminate this sudo dependency on windows so i'll probably work that in as a subsequent change

---
## Creating Self-Contained openconnect-sso.exe for Windows:

"pyinstaller" bundles all the necessary files into a single exe file...
when you see its runtime output you realize it's doing A LOT of work!, including all the Qt5 dlls for webview UI, etc
  which makes some sense why the final .exe winds up over 100megs!
  and why it takes a few moments upon initial launch to unpack everything to %temp% before it starts executing

to get this piece running we just have a couple more pre-reqs:
- make sure you're working in the activated venv: .venv/scripts/activate
  - there is an activate.bat as well as activate.ps1 depending on which shell you're running
- cd .venv/scripts
- pip install pyinstaller
  - make sure that you've loaded pyinstaller into the venv (not global) by confirming pyinstall.exe is now present in the .venv/scripts folder
- for the logic to be happy we need a "user.js" file present in the right location at runtime...
  - this might be intended for us to inject our own javascript into the MFA webview! this could be interesting for automatically inputting login info... the one @impynutz provided so far is blank except for "// ==UserScript==" which suggests greasemonkey if you're familiar
  - copy it from the $repo/openconnect_sso/browser folder to the .venv/scripts folder
- and one last tweak for Windows due to this: https://github.com/pyinstaller/pyinstaller/wiki/Recipe-Multiprocessing
  - edit openconnect-sso and make it look like this:
    ```python
    import sys
    import multiprocessing
    from openconnect_sso.cli import main

    if __name__ == '__main__':
        multiprocessing.freeze_support()
        sys.exit(main())
    ```
  - it looks like this correponds to the $repo/openconnect_sso/cli.py file so presumably that's the original location we need to modify so that it comes back through the make process, i'll have to confirm that.

lastly here's a sample command line (execute from .venv/scripts as your active directory):
  ```
  > pyinstaller.exe --onefile --clean --add-data "user.js;openconnect_sso/browser" openconnect-sso
  ```
(fyi, the "--add-data" syntax is "source;dest")

this will output our final openconnect-sso.exe to .venv/scripts/dist folder!

---
sample hostscan-data file:
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
