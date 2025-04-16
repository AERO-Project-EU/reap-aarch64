# reap-aarch64
This repo contains instructions in order to run REAP [1] in an aarch64 processor.
REAP is a technique to reduce the time it takes to spawn a firecracker microVM using snapshots.
We adopt REAP code as given by the authors of the FaaSnap paper [2].
The author's code can be found in their github repo (https://github.com/ucsdsysnet/faasnap) however it is not
ready to run in aarch64 processors, only on x86_64.
In order to run it on an aarch64 processor we needed to change a few things in their code. Below we give the
detailed instructions on how we managed to run their code in an aarch64 processors (namely, Neoverse-V2).

## Step 1: Build linux kernel for guest microVMs.
We were able to run a vanilla linux kernel inside a firecracker microVM with no modifications.
So, this step is simply a download and build of a linux kernel:
```
wget https://cdn.kernel.org/pub/linux/kernel/v5.x/linux-5.15.178.tar.xz
tar xvf linux-5.15.178.tar.xz
cd linux-5.15.178/
cp config-5.15.178-reap-iccs ./.config
make menuconfig
make -j
ls arch/arm64/boot/Image
```

## Step 2: Build firecracker.
With our changes in REAP+faasnap code we are able to run REAP using a vanilla latest version of firecracker.
```
git clone https://github.com/firecracker-microvm/firecracker.git
cd firecracker
git checkout tags/v1.12.0-dev
tools/devtool build
ls build/cargo_target/aarch64-unknown-linux-musl/debug
```

## Step 3: Download GO.
```
wget https://go.dev/dl/go1.24.1.linux-arm64.tar.gz
tar xvfz go1.24.1.linux-arm64.tar.gz
ls ./go/bin/go
```

## Step 4: Clone the original faasnap code and apply our provide patch.
```
git clone https://github.com/ucsdsysnet/faasnap.git
cd faasnap
git checkout 6d47f5a808d34d37213c57e42a302b351e904614
git apply --reject --whitespace=fix -- faasnap-reap-aarch64-iccs.patch
apt update && apt install net-tools
./prep.sh
go install github.com/go-swagger/go-swagger/cmd/swagger@latest
~/go/bin/swagger generate server -f api/swagger.yaml
go get -u ./... && go build cmd/faasnap-server/main.go
ls ./main
cp -r reap-aarch64/models ./
```

## Step 5: Build functions' rootfs:
```
cd rootfs
make debian-rootfs.ext4
```

## Step 6: Install Redis server and populate it with the functions’ inputs:
We provide in the current repo a populate_redis.py file which can be used to 
```
apt install redis-server python3-pip
pip3 install redis
python3 populate_redis.py
Validate with redis-cli:
auth <pass>
keys *
```

## Step 7: Modify test-2inputs.json and run.
Change the appropriate paths for guests kernel (see Step 1), rootfs (see Step 5) and firecracker binary (see Step2) and run as root:
```
./test.py test-2inputs.json
```

[1] "Benchmarking, Analysis, and Optimization of Serverless Function Snapshots": https://arxiv.org/abs/2101.09355  
[2] "FaaSnap: FaaS made fast using snapshot-based VMs": https://dl.acm.org/doi/10.1145/3492321.3524270
