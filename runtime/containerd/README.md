# containerd

- getting started: [containerd | Getting started with containerd](https://containerd.io/docs/getting-started/)

## Getting started

```
$ vagrant up
$ vagrant ssh
```

In vagrant VM.

Setup golang env.
See: [Download and install - The Go Programming Language](https://golang.org/doc/install)

```
$ wget https://golang.org/dl/go1.15.3.linux-amd64.tar.gz
$ sudo tar -C /usr/local -xzf go1.15.3.linux-amd64.tar.gz
$ export PATH=$PATH:/usr/local/go/bin
$ go version
go version go1.15.3 linux/amd64
$
```

Setup runc.
See: [Getting Started with Containerd](https://medium.com/@Mark.io/getting-started-with-containerd-188b7256c20a)

```
$ sudo apt-get update
$ sudo apt-get install -y gcc pkg-config libseccomp-dev
$ go get github.com/opencontainers/runc
$ cd go/src/github.com/opencontainers/runc
$ make
$ sudo make install
$ cd
```

公式 Doc だと `wget` でとってきているが、インストール方法が何もわからん && `make` が通らないので `go get` を使った方法に変更

See: https://medium.com/@Mark.io/getting-started-with-containerd-188b7256c20a

```
$ go get github.com/containerd/containerd
$ cd go/src/github.com/containerd/containerd
$ make
$ sudo make install
$ cd
```

```
$ sudo mkdir /etc/containerd
$ sudo containerd config default | sudo tee /etc/containerd/config.toml
```

systemd を使って動かす

```
$ sudo cp containerd.service /etc/systemd/system/
$ sudo chmod 664 /etc/systemd/system/containerd.service
```

```
$ sudo systemctl enable containerd
Created symlink /etc/systemd/system/multi-user.target.wants/containerd.service → /etc/systemd/system/containerd.service.
$ sudo systemctl start containerd
$ sudo systemctl status containerd
● containerd.service - containerd container runtime
   Loaded: loaded (/etc/systemd/system/containerd.service; enabled; vendor preset: enabled)
   Active: active (running) since Mon 2020-10-26 14:08:53 UTC; 4s ago
     Docs: https://containerd.io
  Process: 21380 ExecStartPre=/sbin/modprobe overlay (code=exited, status=0/SUCCESS)
 Main PID: 21391 (containerd)
    Tasks: 9
   CGroup: /system.slice/containerd.service
           └─21391 /usr/local/bin/containerd

Oct 26 14:08:53 vagrant containerd[21391]: time="2020-10-26T14:08:53.745364416Z" level=info msg=serving... address=/run/containerd/containerd.sock.ttrpc
Oct 26 14:08:53 vagrant containerd[21391]: time="2020-10-26T14:08:53.745533474Z" level=info msg=serving... address=/run/containerd/containerd.sock
Oct 26 14:08:53 vagrant systemd[1]: Started containerd container runtime.
Oct 26 14:08:53 vagrant containerd[21391]: time="2020-10-26T14:08:53.748542336Z" level=info msg="containerd successfully booted in 0.047725s"
Oct 26 14:08:53 vagrant containerd[21391]: time="2020-10-26T14:08:53.774213162Z" level=info msg="Start subscribing containerd event"
Oct 26 14:08:53 vagrant containerd[21391]: time="2020-10-26T14:08:53.774438336Z" level=info msg="Start recovering state"
Oct 26 14:08:53 vagrant containerd[21391]: time="2020-10-26T14:08:53.774679924Z" level=info msg="Start event monitor"
Oct 26 14:08:53 vagrant containerd[21391]: time="2020-10-26T14:08:53.774871962Z" level=info msg="Start snapshots syncer"
Oct 26 14:08:53 vagrant containerd[21391]: time="2020-10-26T14:08:53.774991182Z" level=info msg="Start cni network conf syncer"
Oct 26 14:08:53 vagrant containerd[21391]: time="2020-10-26T14:08:53.775187678Z" level=info msg="Start streaming server"
```

動作確認

```
$ ps -f -C containerd
UID        PID  PPID  C STIME TTY          TIME CMD
root     21391     1  0 14:08 ?        00:00:00 /usr/local/bin/containerd
```

`/etc/containerd/config.toml` を以下のように修正すると、metrics が取れる。

```
...
[metrics]
        address = "127.0.0.1:1338"
...
```

```
$ curl 127.0.0.1:1338/v1/metrics
...
```

クライアントを使う

```
$ mkdir sample && cd sample
$ go mod init containerdsample
go: creating new go.mod: module containerdsample
```

`main.go` を作成

```go
package main

import (
	"context"
	"fmt"
	"log"
	"syscall"
	"time"

	"github.com/containerd/containerd"
	"github.com/containerd/containerd/cio"
	"github.com/containerd/containerd/oci"
	"github.com/containerd/containerd/namespaces"
)

func main() {
	if err := redisExample(); err != nil {
		log.Fatal(err)
	}
}

func redisExample() error {
	// create a new client connected to the default socket path for containerd
	client, err := containerd.New("/run/containerd/containerd.sock")
	if err != nil {
		return err
	}
	defer client.Close()

	// create a new context with an "example" namespace
	ctx := namespaces.WithNamespace(context.Background(), "example")

	// pull the redis image from DockerHub
	image, err := client.Pull(ctx, "docker.io/library/redis:alpine", containerd.WithPullUnpack)
	if err != nil {
		return err
	}

	// create a container
	container, err := client.NewContainer(
		ctx,
		"redis-server",
		containerd.WithImage(image),
		containerd.WithNewSnapshot("redis-server-snapshot", image),
		containerd.WithNewSpec(oci.WithImageConfig(image)),
	)
	if err != nil {
		return err
	}
	defer container.Delete(ctx, containerd.WithSnapshotCleanup)

	// create a task from the container
	task, err := container.NewTask(ctx, cio.NewCreator(cio.WithStdio))
	if err != nil {
		return err
	}
	defer task.Delete(ctx)

	// make sure we wait before calling start
	exitStatusC, err := task.Wait(ctx)
	if err != nil {
		fmt.Println(err)
	}

	// call start on the task to execute the redis server
	if err := task.Start(ctx); err != nil {
		return err
	}

	// sleep for a lil bit to see the logs
	time.Sleep(3 * time.Second)

	// kill the process and get the exit status
	if err := task.Kill(ctx, syscall.SIGTERM); err != nil {
		return err
	}

	// wait for the process to fully exit and print out the exit status

	status := <-exitStatusC
	code, _, err := status.Result()
	if err != nil {
		return err
	}
	fmt.Printf("redis-server exited with status: %d\n", code)

	return nil
}
```

```
$ go build main.go
$ sudo ./main
1:C 26 Oct 2020 16:17:13.441 # oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
1:C 26 Oct 2020 16:17:13.442 # Redis version=6.0.8, bits=64, commit=00000000, modified=0, pid=1, just started
1:C 26 Oct 2020 16:17:13.442 # Warning: no config file specified, using the default config. In order to specify a config file use redis-server /path/to/redis.conf
1:M 26 Oct 2020 16:17:13.443 # You requested maxclients of 10000 requiring at least 10032 max file descriptors.
1:M 26 Oct 2020 16:17:13.444 # Server can't set maximum open files to 10032 because of OS error: Operation not permitted.
1:M 26 Oct 2020 16:17:13.444 # Current maximum open files is 1024. maxclients has been reduced to 992 to compensate for low ulimit. If you need higher maxclients increase 'ulimit -n'.
1:M 26 Oct 2020 16:17:13.444 * Running mode=standalone, port=6379.
1:M 26 Oct 2020 16:17:13.445 # WARNING: The TCP backlog setting of 511 cannot be enforced because /proc/sys/net/core/somaxconn is set to the lower value of 128.
1:M 26 Oct 2020 16:17:13.445 # Server initialized
1:M 26 Oct 2020 16:17:13.445 # WARNING overcommit_memory is set to 0! Background save may fail under low memory condition. To fix this issue add 'vm.overcommit_memory = 1' to /etc/sysctl.conf and then reboot or run the command 'sysctl vm.overcommit_memory=1' for this to take effect.
1:M 26 Oct 2020 16:17:13.446 * Ready to accept connections
1:signal-handler (1603729036) Received SIGTERM scheduling shutdown...
1:M 26 Oct 2020 16:17:16.465 # User requested shutdown...
1:M 26 Oct 2020 16:17:16.465 * Saving the final RDB snapshot before exiting.
1:M 26 Oct 2020 16:17:16.466 * DB saved on disk
1:M 26 Oct 2020 16:17:16.466 # Redis is now ready to exit, bye bye...
redis-server exited with status: 0
```
