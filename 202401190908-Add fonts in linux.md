---
date:  2024-01-19
tags:
  - howto
  - linux
  - font
---

# Add fonts in linux

## Check if the font exists

```bash
fc-list | rg -i <fontname>
```

## Install new fonts

Download to some directory, e.g. ~/Downloads/fonts.

```bash
mkdir ~/.fonts
mv ~/Downloads/fonts/* ~/.fonts
sudo fc-cache -f -v # force the re-creation of the font cache
```

For more details see [here](https://wiki.ubuntu.com/Fonts).
