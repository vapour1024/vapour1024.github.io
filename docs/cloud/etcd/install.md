# etcd安装

---

## install go

参考[Download and install](https://golang.google.cn/doc/install)

## download etcd

```shell
wget https://github.com/etcd-io/etcd/releases/download/v3.4.9/etcd-v3.4.9-linux-amd64.tar.gz
```

## install goreman

```shell
go install github.com/mattn/goreman@latest
```

## download procfile

```shell
wget https://github.com/etcd-io/etcd/blob/v3.4.9/Procfile
```

## start

```shell
goreman -f Procfile start
```

## test

```shell
etcdctl put hello world
etcdctl get hello --endpoints http://127.0.0.1:2379
```
