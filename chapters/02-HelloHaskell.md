# Hello, Haskell!

## GHC

TODO: instructions on how to install GHC

TODO: examples of building code

TODO: runhaskell

## Cabal

TODO: introduction to Cabal and Hackage

TODO: instructions on how to install Cabal

TODO: example of creating building projects

## Stack

TODO: introduction to Stack and Stackage

TODO: instructions on how to install Slack

TODO: example of creating and building projects

## Nix

TODO: introduction to Nix, NixOS, and Nixpkgs

### Installing Nix

Nix can be installed on any Linux distribution and [macOS](https://nixos.org/download.html#nix-install-macos). See [here](https://nixos.org/download.html) for instructions for your platfrom.

Nix cannot be installed on Windows. The recommended way to run Nix on Windows is to use WSL. See [here](https://docs.microsoft.com/en-us/windows/wsl/install) for instructions on how to get started with WSL, and [here](https://nixos.org/download.html#nix-install-windows) on how to install Nix in it.

In fact, due to the many system dependencies, the most straightforward way to develop Haskell on Windows in general is to use WSL.

To verify that Nix has been installed correctly, issue the following command in the terminal:
```sh
nix --version
```
You should see an output like this:
```
nix (Nix) 2.8.0
```

### Enable Flakes

Nix Flakes are [an obviously good thing](https://grahamc.com/blog/flakes-are-an-obviously-good-thing). We will use them in this guide to resolve dependencies for our Haskell toolchain.

In order to enable flakes, follow [the following guide](https://nixos.wiki/wiki/Flakes#Installing_flakes). If you're in a hurry, execute these commands in the terminal:
```sh
mkdir -p ~/.config/nix
echo experimental-features = nix-command flakes >> ~/.config/nix/nix.conf
```

Verify that flakes are enabled by running this command:
```sh
nix run nixpkgs#hello
```
You should see the following output:
```
Hello, world!
```

### Configuring Cabal to use Nix

By default, Cabal fetches all its dependencies from Hackage. That means that it will spend a considerable amonut of time building them on first build.

We can avoid this by configuring Cabal to use only Nix for its dependency resolution.

If in a hurry, run this command:
```sh
mkdir -p ~/.config/cabal
echo nix: True >> ~/.config/cabal/config
```

### Creating a Haskell project with Nix

Create a new directory and move into it:
```sh
mkdir HelloHaskell
cd HelloHaskell
```

Initialise a new Cabal project by following the Cabal initialisation wizard:
```sh
nix shell nixpkgs#cabal-install nixpkgs#ghc -c cabal init --interactive
```

When the wizard is done, verify that the directory contains the file `HelloHaskell.cabal`.

Create a file named `flake.nix` with the following contents:
```nix
{
    inputs = {
        nixpkgs.url = "github:nixos/nixpkgs/nixpkgs-unstable";
        flake-utils.url = "github:numtide/flake-utils";
    };

    outputs = inputs: with inputs; flake-utils.lib.eachDefaultSystem (system:
        let
            pkgs = nixpkgs.legacyPackages.${system};
            haskell = pkgs.haskellPackages;
            haskellDeps = drv: builtins.foldl'
                (acc: type: acc ++ drv.getCabalDeps."${type}HaskellDepends")
                [ ]
                [ "executable" "library" "test" ];
            helloHaskell = haskell.callCabal2nix "HelloHaskell" ./. { };
        in
        {
            defaultPackage = helloHaskell;

            devShell = pkgs.mkShell {
                nativeBuildInputs = [
                    (haskell.ghcWithPackages (_: haskellDeps helloHaskell))
                    haskell.cabal-install
                    haskell.haskell-language-server
                ];
            };
        });
}
```

When the file is created, we can enter a development shell for this project to build and run it:
```sh
nix develop
cabal build
cabal run
```

You should see the following output:
```
Hello, Haskell!
someFunc
```

### Nix environment in Visual Studio Code

Visual Studio Code has good plugins for working with Haskell. However, they make assumptions about the system which are not true in the Nix setup.

Luckily, there is [an extension](https://marketplace.visualstudio.com/items?itemName=arrterian.nix-env-selector) that makes VSCode aware of the Nix environment.

The extension requires us to provide a `shell.nix` in addition to the `flake.nix`. We can use the excellent [flake-compat project](https://github.com/edolstra/flake-compat) to make the shell automatically follow our flake.

Add the following input to the `inputs` section in `flake.nix`:
```nix
flake-compat = {
    url = "github:edolstra/flake-compat";
    flake = false;
};
```

Then create a file called `shell.nix`:
```nix
let
    lock = builtins.fromJSON (builtins.readFile ./flake.lock);
    flake-compat = import fetchTarball {
        url = "https://github.com/edolstra/flake-compat/archive/${lock.nodes.flake-compat.locked.rev}.tar.gz";
        sha256 = lock.nodes.flake-compat.locked.narHash;
    } { src = ./.; };
in
flake-compat.shellNix
````

Run the following command to update the Nix flake lock:
```sh
nix flake update
```

Finally, run the command "Nix-Env: Select environment" in VSCode and select `shell.nix` on the list.
