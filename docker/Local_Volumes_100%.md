# Local Volumes 100%

## 起因

使用 `GitLab` `CI/CD` 进行一个 `C/C++` 项目的构建，项目大小为 `20G` 左右，期间有多次的手动取消，或构建失败。

之后发现，再想进行构建失败了。

```
ERROR: Job failed (system failure): Error response from daemon: 
write /var/lib/docker/containers/2898d15ecadf356dd939ae1fef18078f16e713e29e968945f28a6813803dbd13/.tmp-config.v2.json104839763: 
no space left on device (docker.go:654:0s)
```

为了提高编译速度，`runner` 目前是注册在一台配置相对较高的机器上（未来计划上 `K8s`），所以上机器看看。

```shell
$ df -BM
Filesystem     1M-blocks  Used Available Use% Mounted on
udev               7715M    0M     7715M   0% /dev
tmpfs              1549M    1M     1548M   1% /run
/dev/vda1         80506M 805065M      0M 100% /
tmpfs              7744M    0M     7744M   0% /dev/shm
tmpfs                 5M    0M        5M   0% /run/lock
tmpfs              7744M    0M     7744M   0% /sys/fs/cgroup
tmpfs              1549M    0M     1549M   0% /run/user/1001
tmpfs              1549M    0M     1549M   0% /run/user/1003
```

系统盘直接拉满了。

`runner` 的运行方式是 `docker`。所以先清理一下 `docker` 这边的悬空镜像：

```shell
$ docker image prune
WARNING! This will remove all dangling images.
Are you sure you want to continue? [y/N] y
Deleted Images:
untagged: python@sha256:084df6be71c601a47d97c07143c9c16a22d99d3113ec4252a15736aa2d1f5465
deleted: sha256:8642a97eba622f06522fb02467758834e2bc2de7f1b665a4d4764c022f943ced
deleted: sha256:87e592d5948a784609013a7ea89b16afe6de7839d7a1297a42d450296ea21e22
deleted: sha256:ef88f94dbba2152bc76ce17e3248b6585b4cc8e9a97d76d51dd40ee34329ff2f
deleted: sha256:b7af92a520b35b79195ff7859b4ea0d060d55e768bcdca039d90e0beaeec8f8d
deleted: sha256:88e32f372c5dcf23a47f94af2e3f37497df2fd2d7ee617237e56d1e05af5e614

Total reclaimed space: 42.25MB
```

清理的出来的空间不多，看来不是这边的问题。

看看机器自己的情况：

```shell
$ sudo du -sh /* | sort -rh | head -15
du: cannot access '/proc/1765789/task/1765789/fd/4': No such file or directory
du: cannot access '/proc/1765789/task/1765789/fdinfo/4': No such file or directory
du: cannot access '/proc/1765789/fd/3': No such file or directory
du: cannot access '/proc/1765789/fdinfo/3': No such file or directory
76G     /var
4.0G    /usr
948M    /swapfile
209M    /boot
74M     /home
4.6M    /etc
976K    /run
124K    /root
52K     /tmp
16K     /opt
16K     /lost+found
12K     /media
4.0K    /srv
4.0K    /mnt
0       /sys
```

深挖进去：

```shell
$ sudo du -sh /var/* | sort -rh | head -15
75G     /var/lib
315M    /var/log
165M    /var/cache
1.4M    /var/backups
36K     /var/tmp
28K     /var/spool
4.0K    /var/opt
4.0K    /var/mail
4.0K    /var/local
0       /var/run
0       /var/lock
```

还有一层：

```shell
$ sudo du -sh /var/lib/* | sort -rh | head -15
75G     /var/lib/docker
237M    /var/lib/apt
40M     /var/lib/dpkg
3.2M    /var/lib/command-not-found
736K    /var/lib/cloud
608K    /var/lib/usbutils
516K    /var/lib/containerd
428K    /var/lib/systemd
120K    /var/lib/ucf
28K     /var/lib/pam
16K     /var/lib/ubuntu-advantage
16K     /var/lib/polkit-1
16K     /var/lib/initramfs-tools
16K     /var/lib/grub
12K     /var/lib/update-manager
```

果然还是 `docker` 的原因，让我康康！：

```shell
$ docker system df
TYPE            TOTAL     ACTIVE    SIZE      RECLAIMABLE
Images          13        1         3.616GB   2.925GB (80%)
Containers      3         3         36B       0B (0%)
Local Volumes   38        3         75.33GB   75.33GB (100%)
Build Cache     0         0         0B        0B
```

原因见：**[Excessive docker volume disk usage](https://gitlab.com/gitlab-org/gitlab-runner/-/issues/1611)**，至今还存在的 `issue`...

```shell
$ docker system prune --volumes --force
Deleted Networks:
gitlab-runner-docker-2_default
gitlab-runner-prj-01_default
gitlab-group-runner-dev-ops-docker_default

Deleted Volumes:
runner-nco1yzyg-project-3236-concurrent-0-cache-c33bcaa1fd2c77edfc3893b41966cea8
runner-on3jjza9-project-4-concurrent-0-cache-3c3f060a0374fc8bc39395164f415a70
9e6dbc309a2222193d45ea337e0bcbe6ef7ada3dc418c54b26f6edce6e06b0e4
405f54ef8f214cee2035202b31b18956fe56534c7da7741f874d44617a3b3b30
0a799d1b51009a242d9e2637cabc12fcab474a60f474dbea072f780182cbe945
1cb2929e831be66752c89af11da2933c5b1d3d701ccaae66acfafdfdf5ab2167
589ba0f68d6aa6dc4ea994d655731867033eb5a7d77955268c2574d34ad2952a
9f4fc1983c6454d07d670617a9ce2db98669235adc6a1f852d81ff2587cbc33e
runner-u9vsygr-project-3181-concurrent-0-cache-c33bcaa1fd2c77edfc3893b41966cea8
b8a59b3d3a62fb135264cb97ff183ed4be47a6f0cfc53a26a7b051e84d5a3267
99b7e204252c97321c9423f074c59014303f87dd0ccaee36683df740e6a21bd4
871436a8fdf4a47871326a811d7b660692ba7506eecccc205efe7628f92291ce
runner-sawgagoa-project-3246-concurrent-0-cache-3c3f060a0374fc8bc39395164f415a70
runner-ywsogair-project-3244-concurrent-0-cache-c33bcaa1fd2c77edfc3893b41966cea8
a1141447b23787ba02cd4902aa23c4947903f63ae134c75b0932c3f841994a67
ccba2521d94b006d45f030d3e6f16775e6c291a4457e3df5f5de558d35fe2256
runner-u9vsygr-project-3181-concurrent-0-cache-3c3f060a0374fc8bc39395164f415a70
runner-on3jjza9-project-6-concurrent-0-cache-c33bcaa1fd2c77edfc3893b41966cea8
6cbddb998429accd0a3f122405af9408e720aca550cd12ecf6214e5b8cdb5e2a
c9a6fb800ad5d2f86557c171248f3d2aaf3de5703bdd238c28f3b1b315d3670c
runner-on3jjza9-project-3-concurrent-0-cache-c33bcaa1fd2c77edfc3893b41966cea8
runner-nco1yzyg-project-3236-concurrent-0-cache-3c3f060a0374fc8bc39395164f415a70
runner-on3jjza9-project-4-concurrent-0-cache-c33bcaa1fd2c77edfc3893b41966cea8
bdbdd4bfbce7b3417db0cfb7ad9e6bee26dfb4cc92d486540c877335d816f1fd
d1f45bb6d50f12193726b6c43cb9fbf9d59e8ad5154162636e6196c30a45e621
90c72e583071d3a47eb0b0a88b9e85e7e2ea7c70254ef6548a5e0b082dc865e3
9f04c86f88ab552a8c290a81e860c0c99ceb7e222fe237589655380ae843a2ef
1094a05c48618ca0d6320c93c7c6d1eb838becb10e6256a9e246d04fd1ddc2dc
65338588e44be521c02886e7f35706f37a34a2d463e90b3bb7e04ce75fee52d4
runner-ywsogair-project-3244-concurrent-0-cache-3c3f060a0374fc8bc39395164f415a70
runner-on3jjza9-project-6-concurrent-0-cache-3c3f060a0374fc8bc39395164f415a70
runner-sawgagoa-project-3246-concurrent-0-cache-c33bcaa1fd2c77edfc3893b41966cea8
61b3767b0a8dfa630efb69310a327c55785d6bda8dab52e058e7540eb3975a58
cabb14d54da136de7b793bc116e930436edd1447287fd63c211f4a044bfd1155
runner-on3jjza9-project-3-concurrent-0-cache-3c3f060a0374fc8bc39395164f415a70

Total reclaimed space: 75.33GB
```

舒服了。

```shell
$ df -BM
Filesystem     1M-blocks  Used Available Use% Mounted on
udev               7715M    0M     7715M   0% /dev
tmpfs              1549M    1M     1548M   1% /run
/dev/vda1         80506M 6745M    70264M   9% /
tmpfs              7744M    0M     7744M   0% /dev/shm
tmpfs                 5M    0M        5M   0% /run/lock
tmpfs              7744M    0M     7744M   0% /sys/fs/cgroup
tmpfs              1549M    0M     1549M   0% /run/user/1001
tmpfs              1549M    0M     1549M   0% /run/user/1003
```

另外可以注意到

```shell
$ docker system prune --volumes --force
# 或者
$ docker volume prune -f
```

并不会影响到正在执行的 `pipline`。因此可以在 `runner` 的机器机器上跑一个定时任务进行定时清理。

link: 
- https://meta.discourse.org/t/discourse-image-and-installation-size-clean-var-lib-docker-overlay2/197957/2
- https://gitlab.com/gitlab-org/gitlab-runner/-/issues/1611