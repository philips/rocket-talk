# App Container Demo

### Packaging a Go Application

Edit: main.go

```
package main

import (
    "log"
    "net/http"
)

func main() {
    http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        log.Printf("request from %v\n", r.RemoteAddr)
        w.Write([]byte("hello\n"))
    })
    log.Fatal(http.ListenAndServe(":5000", nil))
}
```

#### Build a statically linked Go binary

```
CGO_ENABLED=0 GOOS=linux go build -o hello -a -tags netgo -ldflags '-w' .
```

```
file hello
```

#### Create the image manifest

Edit manifest.json

##### Validate the image manifest

```
actool -debug validate manifest.json
```

#### Create the layout and the rootfs

```
mkdir -p hello-layout/rootfs/bin
```

Copy the image manifest

```
cp manifest.json hello-layout/manifest
```

Copy the hello binary

```
cp hello hello-layout/rootfs/bin/
```

#### Build the application image

```
actool build hello-layout/ hello-0.0.1-linux-amd64.aci
```

##### Validate the application image

```
actool -debug validate hello-0.0.1-linux-amd64.aci
```

### Signing the ACI

#### Generate a gpg signing key

Edit: gpg-batch

```
%echo Generating a default key
Key-Type: default
Subkey-Type: default
Name-Real: Kelsey Hightower
Name-Comment: ACI signing key
Name-Email: kelsey.hightower@coreos.com
Expire-Date: 0
Passphrase: rocket
%pubring rocket.pub
%secring rocket.sec
%commit
%echo done
```

```
gpg --batch --gen-key gpg-batch
```

List the keys.

```
gpg --no-default-keyring --secret-keyring ./rocket.sec --keyring ./rocket.pub --list-keys
```

Export the public key.

```
gpg --no-default-keyring --armor \
--secret-keyring ./rocket.sec --keyring ./rocket.pub \
--output pubkeys.gpg \
--export "<kelsey.hightower@coreos.com>"
```

#### Create the detached signature

```
gpg --no-default-keyring --armor \
--secret-keyring ./rocket.sec --keyring ./rocket.pub \
--output hello-0.0.1-linux-amd64.sig \
--detach-sig hello-0.0.1-linux-amd64.aci
```

```
hello-0.0.1-linux-amd64.sig
hello-0.0.1-linux-amd64.aci
pubkeys.gpg
```

```
cp hello-0.0.1-linux-amd64.aci hello-0.0.1-linux-amd64.sig /opt/images/example.com/
```

```
cp pubkeys.gpg /opt/images
```

#### Test the discover end-point

```
actool discover --insecure example.com/hello:0.0.1,os=linux,arch=amd64
```

```
http://example.com/images/example.com/hello-0.0.1-linux-amd64.sig
http://example.com/images/example.com/hello-0.0.1-linux-amd64.aci
http://example.com/pubkeys.gpg
```

# Rocket Demo

### Fetch the example.com/hello:0.0.1 ACI

```
rkt fetch example.com/hello:0.0.1
```

### Trust the example.com signing key

```
rkt trust --prefix example.com/hello
```

### Run the example.com/hello:0.0.1 ACI

```
rkt run example.com/hello:0.0.1
```

### Listing Containers

```
rkt list
```

After the container has stopped:

```
rkt list
```

### rocket gc

Cleaning up old containers with rocket gc

```
rkt gc
```

```
rkt gc
```

Noting happens? Why?

```
rkt gc --grace-period=10s
```

#### Create a systemd timer unit

Edit: /etc/systemd/system/rocket-gc.service

```
[Unit]
Description=Garbage-collect rocket containers no longer in use

[Service]
Type=simple
ExecStart=/opt/bin/rkt gc --grace-period=10s

[Install]
WantedBy=multi-user.target
```

Edit: /etc/systemd/system/rocket-gc.timer

```
[Unit]
Description=Execute rkt gc every 15 mins

[Timer]
OnCalendar=*:0/15
Unit=rocket-gc.service

[Install]
WantedBy=multi-user.target
```

#### Start the rocket-gc timer unit

```
systemctl enable rocket-gc.timer
systemctl start rocket-gc.timer
```	

#### Check the status

```
systemctl status rocket-gc.timer
```
