# runc(低レベルランタイム)によるコンテナの実装

## 検証環境

- Macにはruncがないため、virtualboxで起動した仮想マシン内(centos)で作業を行う
    - vcpu数は2つ以上で検証する(cgroupの検証で必要なため)
	- ややこしいが、本メモでの**ホストはcentosを指す**

- vagrant使うのであれば、以下のVagrantfileで仮想マシンを作成できる

```ruby
Vagrant.configure("2") do |config|
  config.vm.define 'sandbox' do |main|
    main.vm.box = "centos/7"
    main.vm.box_version = "1905.1"
    main.vm.hostname = "sandbox"

    # https://www.vagrantup.com/docs/networking/private_network.html
    main.vm.network "private_network", ip: "192.168.33.10"

    main.vm.synced_folder "./data", "/vagrant"

    # https://www.vagrantup.com/docs/virtualbox/configuration.html#vboxmanage-customizations
    main.vm.provider "virtualbox" do |vb|
      vb.memory = "2048"
      vb.cpus = 2
    end
    # main.vm.provision : shell, path: "bootstrap.sh"
  end
end
```

---
## 作業流れ

- コンテナイメージを取得

- OCI仕様で定義された、 File system bundle を作成
    - コンテナ実行環境の定義ファイル
    - コンテナのルートファイルシステム

- File system bundle を基にコンテナを起動、停止

---
## runcを使ったコンテナの作成、起動

- runcがコンテナを起動するために必要となるデータを格納したディレクトリを作成

```shell
# mkdir bundle
```

- コンテナのルートファイルシステムを取得
    - 本来はイメージの各レイヤをoverlayfsなどで重ね合わせる必要がある

```shell
### ルートファイルシステム取得のために実行
# docker run --rm -d --name rootfs ubuntu:18.04 tail -f /dev/null

### ルートファイシステムをexport
# mkdir -p bundle/rootfs && docker export rootfs | tar x -C bundle/rootfs

### 終了
# docker kill rootfs
```

- コンテナ実行環境用の定義ファイルを入手

```shell
# runc spec -b bundle
# cat bundle/config.json
```

```json
{
	"ociVersion": "1.0.1-dev",
	"process": {
		"terminal": true,
		"user": {
			"uid": 0,
			"gid": 0
		},
		"args": [
			"sh"
		],
		"env": [
			"PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
			"TERM=xterm"
		],
		"cwd": "/",
		"capabilities": {
			"bounding": [
				"CAP_AUDIT_WRITE",
				"CAP_KILL",
				"CAP_NET_BIND_SERVICE"
			],
			"effective": [
				"CAP_AUDIT_WRITE",
				"CAP_KILL",
				"CAP_NET_BIND_SERVICE"
			],
			"inheritable": [
				"CAP_AUDIT_WRITE",
				"CAP_KILL",
				"CAP_NET_BIND_SERVICE"
			],
			"permitted": [
				"CAP_AUDIT_WRITE",
				"CAP_KILL",
				"CAP_NET_BIND_SERVICE"
			],
			"ambient": [
				"CAP_AUDIT_WRITE",
				"CAP_KILL",
				"CAP_NET_BIND_SERVICE"
			]
		},
		"rlimits": [
			{
				"type": "RLIMIT_NOFILE",
				"hard": 1024,
				"soft": 1024
			}
		],
		"noNewPrivileges": true
	},
	"root": {
		"path": "rootfs",
		"readonly": true
	},
	"hostname": "runc",
	"mounts": [
		{
			"destination": "/proc",
			"type": "proc",
			"source": "proc"
		},
		{
			"destination": "/dev",
			"type": "tmpfs",
			"source": "tmpfs",
			"options": [
				"nosuid",
				"strictatime",
				"mode=755",
				"size=65536k"
			]
		},
		{
			"destination": "/dev/pts",
			"type": "devpts",
			"source": "devpts",
			"options": [
				"nosuid",
				"noexec",
				"newinstance",
				"ptmxmode=0666",
				"mode=0620",
				"gid=5"
			]
		},
		{
			"destination": "/dev/shm",
			"type": "tmpfs",
			"source": "shm",
			"options": [
				"nosuid",
				"noexec",
				"nodev",
				"mode=1777",
				"size=65536k"
			]
		},
		{
			"destination": "/dev/mqueue",
			"type": "mqueue",
			"source": "mqueue",
			"options": [
				"nosuid",
				"noexec",
				"nodev"
			]
		},
		{
			"destination": "/sys",
			"type": "sysfs",
			"source": "sysfs",
			"options": [
				"nosuid",
				"noexec",
				"nodev",
				"ro"
			]
		},
		{
			"destination": "/sys/fs/cgroup",
			"type": "cgroup",
			"source": "cgroup",
			"options": [
				"nosuid",
				"noexec",
				"nodev",
				"relatime",
				"ro"
			]
		}
	],
	"linux": {
		"resources": {
			"devices": [
				{
					"allow": false,
					"access": "rwm"
				}
			]
		},
		"namespaces": [
			{
				"type": "pid"
			},
			{
				"type": "network"
			},
			{
				"type": "ipc"
			},
			{
				"type": "uts"
			},
			{
				"type": "mount"
			}
		],
		"maskedPaths": [
			"/proc/acpi",
			"/proc/asound",
			"/proc/kcore",
			"/proc/keys",
			"/proc/latency_stats",
			"/proc/timer_list",
			"/proc/timer_stats",
			"/proc/sched_debug",
			"/sys/firmware",
			"/proc/scsi"
		],
		"readonlyPaths": [
			"/proc/bus",
			"/proc/fs",
			"/proc/irq",
			"/proc/sys",
			"/proc/sysrq-trigger"
		]
	}
}
```

- Filesystem bundleからコンテナを起動

```shell
### bundleディレクトリの中身をもとにmyubuntuという名前でコンテナ実行(ホスト側)
# runc run -b bundle myubuntu

### コンテナ内に入れた
# cat /etc/os-release
NAME="Ubuntu"
VERSION="18.04.4 LTS (Bionic Beaver)"

### コンテナのファイルシステムが広がっている
# ls
bin  boot  dev	etc  home  lib	lib64  media  mnt  opt	proc  root  run  sbin  srv  sys  tmp  usr  var

### プロセスは隔離されているので、ホストのものは見えない
# ps aux
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1  0.0  0.0   4616   664 pts/0    Ss   08:26   0:00 sh
root         8  0.0  0.0  34392  1456 pts/0    R+   08:29   0:00 ps aux

### ホスト側からコンテナを停止
# runc kill myubuntu KILL
```

- **runcが作成するコンテナは単なるLinuxのプロセスにすぎない**
    - 他のプロセスから隔離された特殊なプロセス
    - 仮想マシンのようにカーネルの起動は不要
    - プロセスをforkすれば隔離環境が手に入る

---
## 隔離を支える技術の確認

### namespace

- あるプロセスから見えるプロセスを他のプロセスから隔離する
    - システム全体を占有しているように見せることができる

- 主要なnamespace
    - User namespace
    - PID namespace
    - Mount namespace
    - UTS namespace(ホスト名)
    - IPC namespace(プロセス間通信が可能なプロセス集合)

```shell
### bundle/rootfsをルートファイルシステムとして隔離環境を作成(ホスト側)
### p -> PID
### f -> fork
### u -> uts
### m -> mount
### ルートファイルシステムにchrootを使う
# unshare -pfum chroot bundle/rootfs /bin/sh

### 隔離空間ができている
# cat /etc/os-release
NAME="Ubuntu"
VERSION="18.04.4 LTS (Bionic Beaver)"
ID=ubuntu
ID_LIKE=debian
PRETTY_NAME="Ubuntu 18.04.4 LTS"
VERSION_ID="18.04"
HOME_URL="https://www.ubuntu.com/"
SUPPORT_URL="https://help.ubuntu.com/"
BUG_REPORT_URL="https://bugs.launchpad.net/ubuntu/"
PRIVACY_POLICY_URL="https://www.ubuntu.com/legal/terms-and-policies/privacy-policy"
VERSION_CODENAME=bionic
UBUNTU_CODENAME=bionic

# ls
bin  boot  dev	etc  home  lib	lib64  media  mnt  opt	proc  root  run  sbin  srv  sys  tmp  usr  var
```

### PID namespace(PIDの隔離)の確認

```shell
### unshareコマンドの引数で渡した /bin/sh がPIDD1として動作
# ps aux
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1  0.0  0.0   4616   660 ?        S    08:58   0:00 /bin/sh
root         5  0.0  0.0  34392  1476 ?        R+   09:03   0:00 ps aux

### 隔離環境とホストではPIDが違う(新たなプロセスツリーが組まれる)
# sleep 12345 &
# ps -ao pid,cmd | grep "sleep 12345" | grep -v grep
    6 sleep 12345

### ホスト側のPID
# ps -ao pid,cmd | grep "sleep 12345" | grep -v grep
 4895 sleep 12345
```

### Mount namespaceの確認

- 隔離環境で作成されたマウントポイントはホスト側からは見えない

```shell
### 隔離環境内でprocfsをマウント(-t)
### 疑似ファイルシステムのひとつ
### カーネルだけが知っている情報が取得できる
### 例) システム全体のロードアベレージ/CPU負荷/メモリ利用状況や、プロセスごとの情報
### mount -t [マウントタイプ] [デバイス] [マウント先ディレクトリ]
# mount -t proc proc /proc

# mount
proc on /proc type proc (rw,relatime)

# ls /proc
1	   cgroups    dma	   iomem      kmsg	  misc		sched_debug  swaps	    uptime
13	   cmdline    driver	   ioports    kpagecount  modules	schedstat    sys	    version
6	   consoles   execdomains  irq	      kpageflags  mounts	scsi	     sysrq-trigger  vmallocinfo
acpi	   cpuinfo    fb	   kallsyms   loadavg	  mtrr		self	     sysvipc	    vmstat
asound	   crypto     filesystems  kcore      locks	  net		slabinfo     timer_list     zoneinfo
buddyinfo  devices    fs	   key-users  mdstat	  pagetypeinfo	softirqs     timer_stats
bus	   diskstats  interrupts   keys       meminfo	  partitions	stat	     tty

### ホストの側からは見えない
# mount | grep "bundle/rootfs/proc"
# ls bundle/rootfs/proc/
```

### User namespaceの確認

- ホスト常に存在するユーザ群を隔離し、隔離環境ないで新たなユーザ群を定義する

```shell
### 新規作成したユーザ(usernamespace)がsleepコマンドを実行
# useradd usernamespace
# su - usernamespace
No directory, logging in with HOME=/
$ id
uid=1000(usernamespace) gid=1000(usernamespace) groups=1000(usernamespace)
$ sleep 12345

### ホスト側では、新規作成したユーザ名でプロセスが実行されていない
### 今回のケースはホスト側の一般ユーザ(vagrant)が実行していることになっている
root      6689  0.0  0.0   4616   664 pts/1    S    09:28   0:00 /bin/sh
root      6690  0.0  0.0  49260  1600 pts/1    S    09:29   0:00 su - usernamespace
vagrant   6691  0.0  0.0   4616   660 pts/1    S    09:29   0:00 -su
vagrant   6694  0.0  0.0   4520   384 pts/1    S+   09:29   0:00 sleep 12345
root      6695  0.0  0.0  55172  1880 pts/0    R+   09:29   0:00 ps auxwww
```

---
### cgroup

- プロセスの持つリソースに関して、様々な設定をする
    - 利用可能なCPUやNUMUノード
    - 利用可能なメモリサイズ
    - 利用可能なデバイス

- cgroupの操作ではファイルがインターフェースとされ、ファイルを通じて設定できる
    - cgroup filesystem によりこのファイル群が見れる

```shell
### ホストの、cgroup filesystem がマウントされているディレクトリ
# ls  -l /sys/fs/cgroup/
total 0
drwxr-xr-x. 5 root root  0 Mar  8 08:02 blkio
lrwxrwxrwx. 1 root root 11 Mar  8 08:02 cpu -> cpu,cpuacct
lrwxrwxrwx. 1 root root 11 Mar  8 08:02 cpuacct -> cpu,cpuacct
drwxr-xr-x. 5 root root  0 Mar  8 08:02 cpu,cpuacct
drwxr-xr-x. 3 root root  0 Mar  8 08:36 cpuset
drwxr-xr-x. 5 root root  0 Mar  8 08:02 devices
drwxr-xr-x. 3 root root  0 Mar  8 08:36 freezer
drwxr-xr-x. 3 root root  0 Mar  8 08:36 hugetlb
drwxr-xr-x. 5 root root  0 Mar  8 08:02 memory
lrwxrwxrwx. 1 root root 16 Mar  8 08:02 net_cls -> net_cls,net_prio
drwxr-xr-x. 3 root root  0 Mar  8 08:36 net_cls,net_prio
lrwxrwxrwx. 1 root root 16 Mar  8 08:02 net_prio -> net_cls,net_prio
drwxr-xr-x. 3 root root  0 Mar  8 08:36 perf_event
drwxr-xr-x. 5 root root  0 Mar  8 08:02 pids
drwxr-xr-x. 5 root root  0 Mar  8 08:02 systemd
```

- 隔離環境で利用できるCPUを制限

```shell
### ホスト側のCPUが2コアであることを確認
# cat /sys/fs/cgroup/cpuset/cpuset.cpus
0-1

### 隔離環境に入り、それぞれのCPUが使えていることを確認
# unshare -pfum chroot bundle/rootfs /bin/sh
# taskset -c 0 echo "On core 0"
On core 0
# taskset -c 1 echo "On core1"
On core1

### 3つ目のコアは存在しないためエラー
# taskset -c 2 echo "On core2"
taskset: failed to set pid 4's affinity: Invalid argument
```

- 隔離環境のリソースの使用に制限をかける

```shell
### ホスト側から隔離環境内のシェルのPIDを確認
# ps -ao pid,cmd | grep /bin/sh | grep -v grep | grep -v unshare
 4396 /bin/sh

### ホスト側で設定用のディレクトリを作成
# mkdir /sys/fs/cgroup/cpuset/unshare_demo

### カーネルにより設定ファイルが自動的に作成される
# ls /sys/fs/cgroup/cpuset/unshare_demo/
cgroup.clone_children  cpuset.cpus            cpuset.mem_hardwall        cpuset.memory_spread_slab        notify_on_release
cgroup.event_control   cpuset.effective_cpus  cpuset.memory_migrate      cpuset.mems                      tasks
cgroup.procs           cpuset.effective_mems  cpuset.memory_pressure     cpuset.sched_load_balance
cpuset.cpu_exclusive   cpuset.mem_exclusive   cpuset.memory_spread_page  cpuset.sched_relax_domain_level

### 利用可能なCPUの設定
# echo 0 > /sys/fs/cgroup/cpuset/unshare_demo/cpuset.cpus

### NUMAノードは規定の設定を引き継ぐ
# cat /sys/fs/cgroup/cpuset/cpuset.mems > /sys/fs/cgroup/cpuset/unshare_demo/cpuset.mems

### 対象プロセス(隔離環境内でも実行されているもの)をcgroupに紐づける
### これにより、0番のCPUのみが使えるように
# echo 4396 > /sys/fs/cgroup/cpuset/unshare_demo/tasks
```

- 隔離環境にて、リソースが制限されていることを確認

```shell
# taskset -c 0 echo "On core 0"
On core 0

### 先ほどは使えていた、1番のCPUが使えなくなっている
# taskset -c 1 echo "On core 1"
taskset: failed to set pid 8's affinity: Invalid argument
```

- **コンテナが必要以上のリソースを消費して、他のコンテナに影響を与えることがない！**
