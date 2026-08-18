### Problem Description

Running `mitmptoxy --mode local:process` as a sidecar _privileged_ container in Kubernetes with `kubectl debug --profile=sysadmin --image=mitmproxy/mitmproxy` requires `sudo`. Since `mitmproxy` is already running as root, it should not require executing `sudo` (which also is not in the image and has to be installed).

Naively running `mitmproxy --mode local` in the official image gives error:

```
Error logged during startup:
Failed to run sudo.

Caused by:
    No such file or directory (os error 2)
```

This is due to the following: https://github.com/mitmproxy/mitmproxy_rs/blob/edeb8a23c9b292a9029c26af939de654207a76f3/src/packet_sources/linux.rs#L33-L53

#### I managed to make it work with the following workaround:
1. create ephemeral debug container in a pod you want to debug: `kubectl debug pod/mypod -it --image=mitmproxy/mitmproxy --profile=sysadmin --target=mycontainer -- bash`
2. use `nano` (conveniently in the image) or `kubectl cp` to place a fake `sudo` wrapper in `/usr/bin/sudo` that just drops args while they start with `-` and execs the rest
3. run `mitmproxy --mode local` successfully to debug container egress 🎉 

Fake `sudo` replacement:

```bash
#!/bin/bash
if [[ $1 =~ -h|--help ]]; then
    echo "$(basename "$0"): drop args while start with -, exec the rest"
    exit
fi
args=("$@")
i=0
while [[ ${args[$i]} =~ ^- ]]; do ((i++)); done
if [[ $i -lt ${#args[@]} ]]; then
    exec "${args[@]:$i}"
fi
echo "$(basename "$0"): nothing to run" >&2
exit 1
```

#### Steps to reproduce the behavior:
1. `kubectl debug pod/mypod -it --image=mitmproxy/mitmproxy --profile=sysadmin --target=mycontainer -- mitmproxy --mode local`
2. cannot find `sudo` even if already privileged root


### System Information

```raw
Mitmproxy: 12.2.1
Python:    3.14.0
OpenSSL:   OpenSSL 3.5.4 30 Sep 2025
Platform:  Linux-6.18.5-200.fc43.x86_64-x86_64-with-glibc2.41

Docker image digest: `sha256:743b6cdc817211d64bc269f5defacca8d14e76e647fc474e5c7244dbcb645141`
```

### Checklist

- [x] This bug affects the [latest mitmproxy release](https://github.com/mitmproxy/mitmproxy/releases).