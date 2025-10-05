---
author: yoonhyunwoo
title: DIY(Do It Yourself) 컨테이너 런타임
---

## 컨테이너 런타임(Container runtime)
흔히 도커(Docker)를 **컨테이너 런타임**이라고 부르고는 합니다. 이런 컨테이너 생태계의 용어들은 아직 명확한 용어정의가 없는 상황이라 휴리스틱에 의존합니다. 보통 컨테이너 런타임이라고 하면 두 가지의 스펙이 나오는데, **OCI(Open container initiative) Runtime**과 **CRI(Container Runtime Interface)** 입니다. 앞서 설명한 도커의 경우 OCI Runtime과 CRI스펙을 모두 구현하고 있으며, 이러한 구성요소들이 잘 결합되어있는 형태입니다.

CRI의 경우 쿠버네티스에서 정의한 kubelet이 이용하기 위한 인터페이스이기 때문에 직접적인 컨테이너 생성들을 맡지는 않고 실질적인 컨테이너 생성을 담당하는 runc등의 유틸리티를 활용하고 생성된 컨테이너등을 매니징하는 역할을 맡습니다.
이러한 인터페이스를 구현한 런타임을 **고수준 컨테이너 런타임** 혹은 **컨테이너 런타임 매니저** 라고 부르고, 실질적으로 컨테이너의 스펙을 해석하고 그에 맞게 프로세스 실행 및 격리등을 담당하는 런타임을 **저수준 컨테이너 런타임**이라고 부릅니다.

본 글에서는 컨테이너를 실행하는 저수준 컨테이너 런타임 스펙인 OCI Runtime을 직접 구현해보며 이에 대해 더 이해해보려고 합니다. 물론 지루하고 현학적인 글이 되지 않기 위해 containerd에 대체로 사용가능한 최소한의 구현만 수행할 예정입니다.

구현순서는 `cli > 파일시스템 격리 > 기타 네임스페이스 격리 > Cgroups 격리`와 같이 진행합니다.
이 과정에서 가능한 한 코어 로직에 대해서만 서술하고, 기타 부분에 대해서는 소홀해질 수 있습니다.(제가 인프라에 가까운 SRE 엔지니어라 코드를 못 짜는 탓도 큽니다..) 

OCI Runtime spec은 [opencontainers/runtime-spec](https://github.com/opencontainers/runtime-spec)에 정의되어있습니다. 

고수준 컨테이너 런타임들은 위 스펙을 구현한 프로그램을 CLI(Command Line Interface) 형태로 호출해서 사용하고는 하는데, 문제는 스펙이 정확한 CLI 구성 지침을 제공하는 것이 아니라는 겁니다. 

이를테면 create command는 container-id와 bundle_path을 입력받아 실행하는데, 이것을 환경변수, 플래그, 아규먼트등 어떤 수단으로 입력받을지까지 정의해주지는 않습니다. 본 구현의 최종 목표는 containerd에 런타임을 통합시키는 작업이기 때문에 이런 스펙은 runc, crun등의 런타임과 동일하게 맞춥니다.

## 1. CLI 구현
CLI 명령어를 살펴보며 컨테이너 런타임의 동작 순서에 대해 간단하게 알아보겠습니다.

[runtime-spec/runtime.md](https://github.com/opencontainers/runtime-spec/blob/main/runtime.md)에서 구현해야 할 정보에 대해 알 수 있습니다. 이에 대해 각각의 명령어를 동작방식과 함께 알아보겠습니다.

**create \<container-id> \<path-to-bundle>**

create 명령은 컨테이너 식별자와 번들 디렉터리를 입력받아 컨테이너를 생성합니다.
번들 디렉터리(컨테이너 번들)은 그 컨테이너가 시작되기 위해 필요한 정보를 지닙니다.
여기서 필요한 건 두 가지입니다.
```
bundle 디렉터리
ㄴ config.json
ㄴ 컨테이너 루트 파일시스템
```

config.json은 컨테이너가 어떻게 격리되고 실행되어야 하는지에 대해 정의합니다.
간단한 config.json의 예시는 아래와 같습니다.
```json
{
  "ociVersion": "1.2.0",
  "process": {
    "terminal": true,
    "user": {
      "uid": 0,
      "gid": 0
    },
    "args": [
      "/bin/sh"
    ],
    "env": [
      "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
      "TERM=xterm"
    ],
    "cwd": "/"
  },
  "root": {
    "path": "rootfs",
    "readonly": true
  },
  "hostname": "simple-runc",
  "mounts": [
    {
      "destination": "/proc",
      "type": "proc",
      "source": "proc"
    },
    {
      "destination": "/dev",
      "type": "tmpfs",
      "source": "tmpfs"
    }
  ],
  "linux": {
    "namespaces": [
      {
        "type": "pid"
      },
      {
        "type": "mount"
      }
    ]
  }
}
```

config.json에서는 컨테이너 런타임이 마운트해야 할 마운트포인트, 격리해야 할 네임스페이스, 호스트네임, 환경변수 등 컨테이너가 실행되기 위해 필요한 정보들이 있습니다.

rootfs(루트 파일시스템)를 살펴보기 위헤 ubuntu:latest 이미지의 rootfs를 뜯어보겠습니다.
이미지 pull -> 컨테이너 생성 -> 내보내기와 같이 확인 가능합니다.
```bash
[hwyoon@rocky9 mycontainer]$ sudo docker create --name ubuntu-extract ubuntu:latest /bin/true
Emulate Docker CLI using podman. Create /etc/containers/nodocker to quiet msg.
92e9c6d85748cfdc7c6c899e85be7e953b3647a8360551d519200cf68a7e0c96
[hwyoon@rocky9 mycontainer]$ sudo docker export ubuntu-extract -o ubuntu.tar
Emulate Docker CLI using podman. Create /etc/containers/nodocker to quiet msg.
[hwyoon@rocky9 mycontainer]$ sudo tar -xf ubuntu.tar -C rootfs
[hwyoon@rocky9 mycontainer]$ cd rootfs/
[hwyoon@rocky9 rootfs]$ ls
bin   dev  home  lib64  mnt  proc  run   srv  tmp  var
boot  etc  lib   media  opt  root  sbin  sys  usr
```
bin, dev, home, lib64 등등 ubuntu컨테이너가 사용할 rootfs가 있습니다. ubuntu 이미지를 베이스 이미지로 직접 만든 어플리케이션을 COPY로 집어넣었다면, 여기에 어플리케이션 실행을 위한 바이너리가 들어가고, config.json에는 그를 실행하기 위한 커맨드를 CMD, ENTRYPOINT등에서 정의한 프로세스가 정의될 것 입니다.

본론으로 돌아와서, create 명령은 이러한 번들 디렉터리와 컨테이너 아이디를 받아 컨테이너 프로세스를 준비합니다.

이 시점에서는 config.json에 명시되어있는 **process 항목을 제외**하고 그 외의 부분들을 준비합니다. process의 경우 사용자가 지정한 프로   그램으로, 이후 `start`단계에서 실행되어야 합니다

runc 스펙에 맞게 이를 구현하는 방법은 아래와 같습니다.
```bash
gosudacon create <container-id> --bundle <bundle-directory>
```

urfave/cli를 이용해 더미만 구현해봅시다.

`cmd/main.go`
```go
package main

import (
	"context"
	"fmt"
	"log"
	"os"
	"time"

	"github.com/urfave/cli/v3"
)

var createCommand = &cli.Command{
	Name:      "create",
	Usage:     "create container",
	ArgsUsage: "<container-id>", 
	Flags: []cli.Flag{
		&cli.StringFlag{
			Name:     "bundle",
			Usage:    "Path to the bundle directory containing container configuration",
			Required: true,
			Aliases:  []string{"b"},
		},
	},
	Action: func(ctx context.Context, command *cli.Command) error {
		containerID := command.Args().First()
		bundlePath := command.String("bundle")
		return nil
	},
}

func main() {
	rootCmd := &cli.Command{
		Name:  "gosudacon",
		Usage: "A simple container runtime in Go",
		Commands: []*cli.Command{
			createCommand,
		},
	}

	if err := rootCmd.Run(context.Background(), os.Args); err != nil {
		log.Fatal(err)
	}
}
```

**start**

start 명령은 사용자가 지정해준 프로그램을 실행합니다. config.json의 process 영역에 해당하는 부분입니다.
```go
gosudacon start <container-id>
```

**state**

**kill**

**delete**

