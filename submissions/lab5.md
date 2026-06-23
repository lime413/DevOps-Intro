# Lab 5 Submission

## Task 1

### Vagrantfile

See [Vagrantfile](/Users/tatyana/Documents/DevOps-Intro/Vagrantfile).

### `vagrant up` Output

```text
Bringing machine 'default' up with 'virtualbox' provider...
==> default: Box 'bento/ubuntu-24.04' could not be found. Attempting to find and install...
    default: Box Provider: virtualbox
    default: Box Version: 202510.26.0
==> default: Loading metadata for box 'bento/ubuntu-24.04'
    default: URL: https://vagrantcloud.com/api/v2/vagrant/bento/ubuntu-24.04
==> default: Adding box 'bento/ubuntu-24.04' (v202510.26.0) for provider: virtualbox (arm64)
    default: Downloading: https://vagrantcloud.com/bento/boxes/ubuntu-24.04/versions/202510.26.0/providers/virtualbox/arm64/vagrant.box
==> default: Successfully added box 'bento/ubuntu-24.04' (v202510.26.0) for 'virtualbox (arm64)'!
==> default: Importing base box 'bento/ubuntu-24.04'...
```

### Verification

Inside the VM:

```text
$ vagrant ssh -c 'go version'
go version go1.24.5 linux/arm64

$ vagrant ssh -c 'systemctl is-active quicknotes && curl -s http://127.0.0.1:8080/health'
active
{"notes":4,"status":"ok"}
```

From the host:

```text
$ curl -sS -D - http://127.0.0.1:18080/health
HTTP/1.1 200 OK
Content-Type: application/json
Date: Tue, 23 Jun 2026 12:57:24 GMT
Content-Length: 26

{"notes":4,"status":"ok"}
```

### Design Answers

#### a) Synced folders

I used the `virtualbox` synced folder type and mounted `./app` into `/srv/quicknotes/app`. I picked it because it gives a real mounted path inside the guest and does not need a separate sync command after boot. The trade-off is that it is provider-specific and file I/O is usually slower than native disk access.

#### b) NAT vs Bridged vs Host-only

The VM uses the default NAT network mode plus a forwarded port from `127.0.0.1:18080` on the host to guest port `8080`. This is safer than Bridged networking for a course lab because the service is reachable only from my laptop and is not exposed to the whole local network.

#### c) Provisioning option

I used the `shell` provisioner. For this lab the setup is small and direct: install `curl`, install the pinned Go release, build QuickNotes, and register one systemd service. Using Ansible or another heavier provisioner here would add complexity before Lab 7, where configuration management becomes the real topic.

#### d) Why pin Go `1.24.5`

Pinning `1.24.5` makes the environment reproducible. `1.24` is only a major.minor line and can drift to a newer patch release with different fixes or behavior, while `1.24.5` gives every student the same toolchain and the same patch-level security fixes.

## Task 2

### Snapshot Commands

```text
$ vagrant snapshot save clean-go-installed
==> default: Snapshotting the machine as 'clean-go-installed'...
==> default: Snapshot saved!

$ vagrant ssh -c 'sudo rm -rf /usr/local/go /usr/local/bin/go'

$ vagrant ssh -c 'go version'
bash: line 1: go: command not found

$ /usr/bin/time -p vagrant snapshot restore --no-provision clean-go-installed
==> default: Forcing shutdown of VM...
==> default: Restoring the snapshot 'clean-go-installed'...
==> default: Resuming suspended VM...
==> default: Booting VM...
==> default: Waiting for machine to boot. This may take a few minutes...
==> default: Machine booted and ready!
==> default: Machine not provisioned because `--no-provision` is specified.
real 14.11
user 1.74
sys 1.06

$ vagrant ssh -c 'go version && curl -s http://127.0.0.1:8080/health'
go version go1.24.5 linux/arm64
{"notes":4,"status":"ok"}

$ curl -sS http://127.0.0.1:18080/health
{"notes":4,"status":"ok"}
```

### Restore Timing

```text
real 14.11
user 1.74
sys 1.06
```

### Design Answers

#### e) Why snapshots are not backups

Snapshots are stored next to the same VM disk chain, so they do not protect against host disk failure, accidental deletion of the VM directory, or corruption of the whole VirtualBox machine. They are useful for quick rollback, but they are not an independent copy of the system.

#### f) Copy-on-write

Copy-on-write means a snapshot mostly records blocks that change after the snapshot is taken instead of copying the whole disk immediately. Because of that, one snapshot is small at first, but ten snapshots still grow over time as more blocks diverge across the chain.

#### g) When snapshotting is an antipattern

Snapshotting becomes an antipattern when long snapshot chains replace normal rebuild and configuration practices. Long chains consume storage, make rollback history harder to reason about, and can slow down VM operations.

## Bonus Task

### Comparison Table

| Dimension | Vagrant VM | Docker container |
|-----------|-----------:|-----------------:|
| Cold start | 20.59 s | 0.09 s |
| Idle RAM | 242 Mi used | 6.17 MiB used |
| On-disk size | 3.0 G | 1.33 G |
| Process count (guest) | 106 | 2 |

### Analysis

The start-time gap was the clearest result: the VM needed about 20.6 seconds to boot, while the container restarted in about 0.09 seconds. The container also used much less idle memory, but the image size was still not tiny because the `golang:1.24` base image includes a full toolchain. The VM is the better tool when I need a full guest OS boundary, systemd, and behavior that is close to a real server. The container is the better tool for fast, repeatable, stateless application processes. These numbers explain why containers became the default for microservices: they start quickly, pack densely on the host, and make replacement cheaper than repair.
