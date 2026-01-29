This repository sets up a clawbot docker container setup with tailnet so you can easily add it to any network and make it as secure as possible.

## Submodule (moltbot)

[moltbot](https://github.com/moltbot/moltbot) is included as a git submodule. After cloning this repo, initialize and fetch it with:

```bash
git submodule update --init
```

To update the submodule to the latest commit later:

```bash
git submodule update --remote moltbot
```
