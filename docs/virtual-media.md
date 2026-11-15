# Virtual media (ISO files)

The BMCs' own Java consoles can attach a disk image to the server as if it were
a USB or optical drive. SquidBMC makes a folder of ISO files on the host
available to those consoles.

## Adding ISO files

1. Put `.iso` files in the `isos` folder in the repository root.
2. Open the graphical console for the server (see [Using SquidBMC](usage.md)).
3. In the console's **Virtual Media** menu choose **Image File** and browse to
   `/isos`.

New files appear without a restart: the folder is a bind mount, not a copy.

## Using a different folder

`docker-compose.yml` mounts the folder into the `console` (engine) container:

```yaml
volumes:
  - ./isos:/isos:ro
```

Change the left-hand side to use another folder on the host, for example
`/mnt/c/Users/you/Downloads:/isos:ro` on WSL2, then run `docker compose up -d`
to recreate the container. Keep the right-hand side as `/isos`.

## Notes

- The folder is mounted read-only (`:ro`), so the consoles cannot change or
  delete your images.
- Only the `console` container sees the folder. Virtual media is part of the
  graphical console; the text console and the dashboard do not use it.
- The image is streamed from the engine container to the BMC over the network,
  so the host must be able to reach the BMC's virtual media port (see
  [Installation](installation.md#requirements)).
- The repository tracks only `isos/.keep`. Do not commit ISO files.
