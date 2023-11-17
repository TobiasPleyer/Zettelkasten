---
date:  2023-10-20
tags:
  - nix
  - flake
  - buildEnv
---

# Reproducible work environment with Nix flake

flake.nix

```nix
{
  inputs = {
    nixpkgs.url = "github:NixOS/nixpkgs";
  };
  
  outputs = { self, nixpkgs }: {
    defaultPackage.x86_64-linux =
      with import nixpkgs { system = "x86_64-linux"; };
      buildEnv {
        name = "tui-email";
        paths = [
          aerc
          pass
          isync
          goimapnotify
          msmtp
          notmuch
        ];
      };
    };
}
```
