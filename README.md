# `autosocks`

Use `systemd` & `ssh` magic, to dynamically punch through firewalls; with "scale to zero" because coolness.

## Use-Case

I have a bunch of systems I need to talk to which are hidden behind multiple jumpboxes.
I want to use local clients to talk to APIs and systems behind those jumpboxes,
I don't want to interactively log in to any jumpbox and run stuff or leave state there[1].
I want my `ssh` connections to be relatively short lived and don't want to
manually manage them; they should be established and closed on demand.

In my case this is mostly to talk to things like
- [BOSH director][bosh]s
- [opsmanager]s
- [credhub]s
- [TKGi][tkgi] APIs
- kube apiservers
- ...

via socks5, where those live inside private networks accessible only via one ore
more bastion / jump hosts.

[1]: That should be forbidden anyway. No state and no interactive sessions on
jumpboxes. Don't do it.

[bosh]: https://bosh.io
[opsmanager]: https://docs.pivotal.io/ops-manager/
[credhub]: https://docs.cloudfoundry.org/credhub/
[tkgi]: https://docs.pivotal.io/tkgi/

## TL;DR

```console
mkdir -p ~/.config/systemd/user
cp units/*-@.{service,socket} ~/.config/systemd/user
cd ~/.config/systemd/user

proxyhost=opsman.nonprod.acme
proxyport=65500

ln -s autosocks-@.socket              autosocks-${proxyhost}@${proxyport}.socket
ln -s autosocks-@.service             autosocks-${proxyhost}@${proxyport}.service
ln -s autosocks-connection-@.service  autosocks-connection-${proxyhost}@${proxyport}.service

systemctl --user reload-daemon
systemctl --user start autosocks-${proxyhost}@${proxyport}.socket
```

## Prerequisites

### `ssh`

I don't care about which or how many jumpboxes I need to use. I set it up once
in my `~/.ssh/config` and forget about it.

```
Host *
  ControlMaster auto
  ControlPath ~/.ssh/cm-%C
  ControlPersist no

Host *.prod.acme
  ForwardAgent yes
  ProxyJump bastion.prod.acme

Host *.sandbox.acme
  ForwardAgent yes
  ProxyJump bastion.sandbox.acme
```

This, and the fact that I use the ssh-agent, pull keys from vault and add them
to the ssh-agent with a limited lifetime (`ssh-add -t ...`) allows me to e.g.
do `ssh opsman.prod.acme` and it logs me.

Going forward, for each system you want to use as a socks proxy this needs to
be true: You need to be able to ssh in with just providing the hostname. There
must not be any other interaction (password prompt) or any other config needed;
everything needs to be handled by the ssh config and the ssh-agent.

### `systemd` foo

Now the fun part: Setting up `systemd` to dynamically manage your connection

1. in [units](./units/) you'll find 3 systemd unit templates. You need to copy
   those to your user's systemd units directory (e.g.
   `~/.config/systemd/user/`). Those are just templates and won't do anything
   by themselves, but implement all the needed parts once they are
   instantiated. 

    - `autosocks-connection-@.service`

      This implements the actual ssh connection, that will be started on-demand.
      Essentially, it will just call `ssh $hostname` and will use a specific
      port to bind ssh as a socks server on.

    - `autosocks-@.service`

      This will run `systemd-socket-proxyd` which will take get the socket from
      systemd once connections have been observed and will proxy to the the
      socks proxy created by ssh.

      It will also make sure to:
      - Start a `autosocks-connection-@.service` instance on-demand and shut it down
      - Shut itself down, once ther is no traffic going through

    - `autosocks-@.socket`

      this will have `systemd` listen on a port and wait for connections. Once
      a connection is observed, `systemd` will start a `autosocks-@.service`
      instance and pass the socket to that.

2. Instantiate & start those units

   `systemd` units can be templated with an "instance" (the thing after the
   `@`) and a "prefix" (the thing between the last `-` and the `@`).
   We'll use that to configure the jumphost we want to go through and the local
   port for the socks service we want to use.

   With the jumpboxes from the above ssh config we can do the following:

   ```console
   cd ~/.config/systemd/user/

   # set up a proxy through opsman.sandbox.acme
   ln -s autosocks-@.socket             autosocks-opsman.sandbox.acme@65500.socket
   ln -s autosocks-@.service            autosocks-opsman.sandbox.acme@65500.service
   ln -s autosocks-connection-@.service autosocks-connection-opsman.sandbox.acme@65500.service
   # enable the sandbox proxy
   systemd --user start autosocks-opsman.sandbox.acme@65500.socket

   # set up a proxy through opsman.prod.acme
   ln -s autosocks-@.socket             autosocks-opsman.prod.acme@65502.socket
   ln -s autosocks-@.service            autosocks-opsman.prod.acme@65502.service
   ln -s autosocks-connection-@.service autosocks-connection-opsman.prod.acme@65502.service
   # enable the prod proxy
   systemd --user start autosocks-opsman.prod.acme@65502.socket
   ```

   This creates a listening socket on `localhost:65500` for sandbox and `localhost:65502` for prod.

   If you did a `netstat -lnp` or similar, you'd see that on the ports 65500
   and 65502 `systemd` is listening on. But not much more.


   **Note**: You might ask "Why don't you use 65501 for the prod proxy?". As it
   stands right now, each "autoproxy" needs two ports, one for "autoproxy"
   permanently listening on, and one, the next higher one, for `ssh` to run
   it's socks proxy on, which "autoproxy" eventually and on-demand will proxy
   too. For now: deal with it, and always leave the next port free!

   **Note**: If you want to connect to a host with a `-` in its FQDN/name, you
   need to escape that `-` by `\x2d`. So the commands to link the units for a
   hostname like `opsman.my-bizunit.acme` would become something like:
   ```console
   ln -s autosocks-@.socket             autosocks-opsman.my\\x2dbizunit.acme@65502.socket
   ln -s autosocks-@.service            autosocks-opsman.my\\x2dbizunit.acme@65502.service
   ln -s autosocks-connection-@.service autosocks-connection-opsman.my\\x2dbizunit.acme@65502.service
   ```

## client setup and usage

Now let's use that thing!

```console
# It takes a bit to start all the systems, but eventually this should return
# (given, the host you proxy through has internet access).
https_proxy=socks5://localhost:65500 curl https://google.com/

# This is faster, because the connection is already established
https_proxy=socks5://localhost:65500 curl https://google.com/
```

And what you see now:
- There is a ssh process, connecting to `opsman.sandbox.acme` which is listening as a socks server on 65501
- There is a process `systemd-socket-proxyd` running now, which proxies everything on port 65500 to port 65501

If you wait a bit and there is no further traffic going through 65500
- both `ssh` and `systemd-socket-proxyd` will shut down automatically.
- the socket listening on 65500 will be moved back to the systemd process


## How it works, how the things interact

- Once the `autosocks-@.socket` instance is running, systemd will listen on that intance's port (e.g. 65500)
- If there is a connection to that port, systemd start
  - `ssh` to the instance's host, running a socks server on the port 1 higher than the instance's port (e.g. 65501)
  - `systemd-socket-proxyd` will be started; it will receive the socket from `systemd` and use it to receive traffic on 65500 and proxy it through to 65501
- `systemd-socket-proxyd` will shutdown, if it's idle for a bit
  - once `systemd-socket-proxyd` shuts down, so will the `ssh` connection providing the socks server
- if any of either the ssh connection or the activation socket will shutdown or
  die, so will the `systemd-socket-proxyd`. The system will however try to
  reestablish all the things on the next connection attempt to the instnace's
  port

## Useful commands

- See all statuses: `systemctl --user status --all 'autosocks*'`
- Tail all logs: `journalctl --user --all -f -u 'autosocks*'`

## Potential improvements

- Use a unix socket or a different net NS or something, so that we don't waste
  2 ports and have this "magic" second port for `ssh` to listen on.

## Shout Out

This is pretty much stolen from [ankostis] over at [StackExchange][se]. The
only thing I did was to make it a bit easier to reuse by making use of the
units' "instance" and "prefix".

[se]: https://unix.stackexchange.com/a/635178
[ankostis]: https://unix.stackexchange.com/users/156357/ankostis
