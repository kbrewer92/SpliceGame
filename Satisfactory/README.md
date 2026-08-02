# Satisfactory Dedicated Server

A Satisfactory dedicated server running on the `splicecloud` k3s cluster, built on
[wolveix/satisfactory-server](https://github.com/wolveix/satisfactory-server).

| | |
|---|---|
| Image | `wolveix/satisfactory-server:latest` |
| Namespace | `default` |
| Node | `spliceprd.local` (`10.0.0.5`) |
| Storage | 50Gi, `local-path` (cluster default StorageClass) |

## Connecting

In game: **Server Manager → Add Server**, then enter:

```
10.0.0.5:30000
```

That is the *game* port. The messaging port (30001) is used for server-to-client
messaging and is not the address you type into the client.

## Deploying

```sh
kubectl apply -f service.yaml -f statefullset.yaml
```

Validate without touching the cluster first:

```sh
kubectl apply -f service.yaml -f statefullset.yaml --dry-run=server
```

**First boot takes a while.** SteamCMD downloads roughly 8GB of game files
before anything listens on the game port. The `startupProbe` allows up to 60
minutes for this; the pod will not report Ready until the server is actually
accepting connections. Watch progress with:

```sh
kubectl logs -f statefulset/satisfactory-server
```

## Ports

| Purpose | Container | NodePort | Protocol |
|---|---|---|---|
| Game | 30000 | 30000 | TCP + UDP |
| Messaging | 30001 | 30001 | TCP |

Two things are deliberate here:

**Both game entries share nodePort 30000.** Satisfactory uses the same port
number for its reliable (TCP) and unreliable (UDP) channels. Kubernetes permits
the shared number because nodePort uniqueness is per port *and* protocol.

**The container listens on the NodePort numbers directly** — `SERVERGAMEPORT`
and `SERVERMESSAGINGPORT` are set to 30000/30001 rather than the upstream
defaults of 7777/8888. The server advertises its own game port to clients, so if
it listened on 7777 while the outside world had to reach 30000, clients would be
directed to a port that isn't published.

To serve on the real 7777 instead, the k3s `--service-node-port-range` would have
to be widened (a server flag change and restart), and the two env vars and all
port numbers reset to 7777/8888.

## Configuration

Environment variables are set on the container in `statefullset.yaml`. The ones
currently set:

| Variable | Value | Purpose |
|---|---|---|
| `PUID` / `PGID` | `1000` | User/group the server drops to |
| `SERVERGAMEPORT` | `30000` | Game port (see above) |
| `SERVERMESSAGINGPORT` | `30001` | Messaging port |
| `MAXPLAYERS` | `6` | Player limit |
| `MAXTICKRATE` | `30` | Maximum simulation tick rate |
| `AUTOSAVENUM` | `5` | Rotating autosave count |
| `TIMEOUT` | `30` | Client timeout, seconds |
| `SERVERSTREAMING` | `true` | Asset streaming |
| `DISABLESEASONALEVENTS` | `false` | FICSMAS toggle |
| `SKIPUPDATE` | `false` | Skip game updates on start |
| `DEBUG` | `false` | Server debug mode |
| `LOG` | `false` | Disable log pruning |
| `VMOVERRIDE` | `false` | Bypass the CPU model check |

Others supported upstream but left at their defaults include `STEAMBETA`,
`STEAMBETAID`, `STEAMBETAKEY`, `MAXOBJECTS`, and `MULTIHOME`.

Most in-game settings (server name, password, admin password) are configured
through the client's Server Manager after the first connection, not through
these variables.

## Storage

One 50Gi PVC mounted at `/config`, containing:

| Path | Contents |
|---|---|
| `/config/gamefiles` | Game installation, ~8GB |
| `/config/saved` | Blueprints, saves, server configuration |
| `/config/backups` | Automatic save backups |
| `/config/logs` | Steam and Satisfactory logs |

`persistentVolumeClaimRetentionPolicy` is `Retain` for both `whenDeleted` and
`whenScaled`, so the world survives deleting the StatefulSet or scaling it to
zero. Deleting the PVC itself is what destroys the save — `local-path` uses
`Delete` as its reclaim policy, so the underlying directory goes with it.

Copy a save out to your workstation:

```sh
kubectl cp default/satisfactory-server-0:/config/saved ./saved-backup
```

## Operations

```sh
# Logs
kubectl logs -f statefulset/satisfactory-server

# Shell (lands as root; the server itself runs as uid 1000)
kubectl exec -it satisfactory-server-0 -- bash

# Restart
kubectl rollout restart statefulset/satisfactory-server

# Stop without losing the world
kubectl scale statefulset/satisfactory-server --replicas=0

# Resource usage
kubectl top pod satisfactory-server-0
```

The game checks for and applies updates on every container start unless
`SKIPUPDATE=true`. A restart is therefore not instant, and a large game patch
will extend it.

## Notes and gotchas

**The container must start as root.** `init.sh` runs
`chown -R $PUID:$PGID /config` — necessary because `local-path` provisions
root-owned directories — and then `exec gosu steam run.sh`. It aborts outright if
started as a non-root UID that doesn't match the steam user baked into the image.
Do not add a pod `securityContext` with `runAsUser` or `fsGroup`; use `PUID`/`PGID`
instead. Changing to a UID other than 1000 requires rebuilding the image with the
`UID`/`GID` build args.

**Resource sizing.** Requests are 2 CPU / 8Gi, limits 8 CPU / 24Gi. The official
wiki recommends 8–16GB of RAM; the headroom above that is deliberate, since
late-game factories at six players are what actually drive memory use.
`spliceprd.local` has 16 CPU / 64Gi total, so the limits leave room for the
existing 7DTD and Hytale servers.

**Health probes are TCP-only.** They confirm something is listening on the game
port, not that the world has finished loading or that the simulation is healthy.
The `livenessProbe` is deliberately slack (60s period, 5 failures) so a long
autosave stall doesn't restart a working server.

**Scaling past one replica does not work.** Each replica would get its own PVC
and run an independent world rather than clustering.

**`VMOVERRIDE`** is `false` because `spliceprd.local` is bare metal. If the
entrypoint ever refuses to start over its CPU model check, set it to `"true"`.
