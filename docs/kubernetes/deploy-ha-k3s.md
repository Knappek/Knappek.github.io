# Deploy a HA k3s Cluster

You can use [this script](https://gist.github.com/Knappek/9f8a8a07c4cef4bacf1db554ae265772) to provision a k3s cluster on your computer (tested on a Mac) using  [k3sup](https://github.com/alexellis/k3sup), [multipass](https://github.com/canonical/multipass).

Kudos go to Tom Watt and his article [Creating a k3s cluster with k3sup and multipass](https://dev.to/tomowatt/creating-a-k3s-cluster-with-k3sup-multipass-h26).

## Usage

Print help message:

```shell
./install-k3s-with-multipass.sh -h
```

### Example: Install 3 node cluster wiht 3 CP nodes

```shell
./install-k3s-with-multipass.sh \
  --control-plane-nodes 3 \
  --worker-nodes 3 \
  --name k3s \
  --worker-cpus 2 \
  --worker-memory 8G
```

### Delete everything

Simply delete all VMs managed by multipass:

```shell
multipass delete --all && multipass purge
```

CAUTION: this will delete ALL VMs, if you have more VMs managed with multipass, use this command instead:

```shell
multipass delete --purge <VM name1>
multipass delete --purge <VM name2>
```

## Known Issues

When you use iTerm2 and see the following issue with `multipass shell <node>`

```shell
ssh: connect to host 192.168.252.248 port 22: No route to host
```

then you might need to allow local network connections for iTerm2, see [this stackexchange](https://apple.stackexchange.com/questions/476030/no-route-to-host-on-iterm2-only).
