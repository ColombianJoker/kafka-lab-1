# Sample commands

## Synopsis

As planed, there will be three **kafka** nodes, `server1` (**kafka1** or `192.168.25.221`), `server2` (**kafka2** or `192.168.25.103`), and `server3` (**kafka3** or `192.168.25.104`). I used different names to show/demo the ansible nomenclature. I have two x86-64 physical servers (`server1` and `server2`) and one virtual aarch64 VM (`server3`) running on VMware Fusion on MacOS 15. That demos the possibility of use of mixed architectures on Kafka clusters. There will be two producers, these will be containers running on another VM on Fusion on MacOS. There will be two consumers, these will be containers running on QNAP Container Station.

### Machines

```sh
[0 e@Gunther:~/Dropbox/Documents/Code/Kafka/kafka-lab-1]
▶︎ ansible-inventory --graph
@all:
  |--@ungrouped:
  |--@kafka:
  |  |--kafka1
  |  |--kafka2
  |  |--kafka3
  |--@producers:
  |  |--producer1
  |  |--producer2
[0 e@Gunther:~/Dropbox/Documents/Code/Kafka/kafka-lab-1]
▶︎
```

## Installation

The full kafka nodes will be done using ansible, with the `init-kraft-cluster.yml` playbook. Note the playbook has a _fixed_ ID that must be changed to be reused.

```sh
[0 e@Gunther:~/Dropbox/Documents/Code/Kafka/kafka-lab-1]
▶︎ ansible-playbook init-kraft-cluster.yml

PLAY [Install and Initialize Kafka KRaft Cluster on RHEL 10] *************************************

TASK [Gathering Facts] ***************************************************************************
ok: [kafka2]
ok: [kafka1]
ok: [kafka3]

TASK [Install OpenJDK 21 headless and firewalld] *************************************************
ok: [kafka3]
ok: [kafka1]
ok: [kafka2]

TASK [Ensure firewalld is running] ***************************************************************
ok: [kafka3]
ok: [kafka2]
ok: [kafka1]

TASK [Open Kafka ports in firewalld] *************************************************************
ok: [kafka3] => (item=9092/tcp)
ok: [kafka2] => (item=9092/tcp)
ok: [kafka1] => (item=9092/tcp)
ok: [kafka3] => (item=9093/tcp)
ok: [kafka2] => (item=9093/tcp)
ok: [kafka1] => (item=9093/tcp)

TASK [Ensure kafka system group exists] **********************************************************
ok: [kafka3]
ok: [kafka2]
ok: [kafka1]

TASK [Ensure kafka system user exists] ***********************************************************
ok: [kafka3]
ok: [kafka2]
ok: [kafka1]

TASK [Ensure kafka data storage directory exists and has correct ownership] **********************
changed: [kafka3]
changed: [kafka2]
changed: [kafka1]

TASK [Set cluster ID environment variable] *******************************************************
ok: [kafka3]
ok: [kafka2]
ok: [kafka1]

TASK [Check if Kafka archive is already downloaded] **********************************************
ok: [kafka3]
ok: [kafka2]
ok: [kafka1]

TASK [Download Kafka distribution] ***************************************************************
skipping: [kafka1]
skipping: [kafka2]
skipping: [kafka3]

TASK [Extract Kafka archive to /opt] *************************************************************
skipping: [kafka3]
skipping: [kafka1]
skipping: [kafka2]

TASK [Create /opt/kafka symlink] *****************************************************************
ok: [kafka3]
ok: [kafka2]
ok: [kafka1]

TASK [Ensure log directory exists] ***************************************************************
ok: [kafka3]
ok: [kafka2]
ok: [kafka1]

TASK [Deploy server.properties] ******************************************************************
ok: [kafka3]
ok: [kafka2]
ok: [kafka1]

TASK [Check if storage is already formatted] *****************************************************
ok: [kafka3]
ok: [kafka2]
ok: [kafka1]

TASK [Format Kafka Storage Directory with Shared Cluster ID] *************************************
skipping: [kafka1]
skipping: [kafka2]
skipping: [kafka3]

TASK [Deploy systemd service unit] ***************************************************************
ok: [kafka3]
ok: [kafka2]
ok: [kafka1]

TASK [Ensure Kafka service is started and enabled] ***********************************************
changed: [kafka3]
ok: [kafka2]
ok: [kafka1]

PLAY RECAP ***************************************************************************************
kafka1                     : ok=15   changed=1    unreachable=0    failed=0    skipped=3    rescued=0    ignored=0
kafka2                     : ok=15   changed=1    unreachable=0    failed=0    skipped=3    rescued=0    ignored=0
kafka3                     : ok=15   changed=2    unreachable=0    failed=0    skipped=3    rescued=0    ignored=0

[0 e@Gunther:~/Dropbox/Documents/Code/Kafka/kafka-lab-1]
▶︎
```

# Validation

## Are these active?

Checking if the nodes and services in the cluster are active (using **Ansible**):

```sh
[0 e@Gunther:~/Dropbox/Documents/Code/Kafka/kafka-lab-1]
▶︎ ansible kafka -m command -a "systemctl is-active kafka"
kafka3 | CHANGED | rc=0 >>
active
kafka2 | CHANGED | rc=0 >>
active
kafka1 | CHANGED | rc=0 >>
active
[0 e@Gunther:~/Dropbox/Documents/Code/Kafka/kafka-lab-1]
▶︎
[0 e@Gunther:~/Dropbox/Documents/Code/Kafka/kafka-lab-1]
▶︎ ansible kafka -b -m shell -a "ss -tulpn | grep -E ':(9092|9093)'"
kafka3 | CHANGED | rc=0 >>
tcp   LISTEN 0      50                                    *:9093             *:*    users:(("java",pid=266999,fd=145))
tcp   LISTEN 0      50                                    *:9092             *:*    users:(("java",pid=266999,fd=173))
kafka2 | CHANGED | rc=0 >>
tcp   LISTEN 0      50                                      *:9093             *:*    users:(("java",pid=24981,fd=145))
tcp   LISTEN 0      50                                      *:9092             *:*    users:(("java",pid=24981,fd=173))
kafka1 | CHANGED | rc=0 >>
tcp   LISTEN 0      50                                      *:9093             *:*    users:(("java",pid=24996,fd=145))
tcp   LISTEN 0      50                                      *:9092             *:*    users:(("java",pid=24996,fd=177))
[0 e@Gunther:~/Dropbox/Documents/Code/Kafka/kafka-lab-1]
▶︎
[0 e@Gunther:~/Dropbox/Documents/Code/Kafka/kafka-lab-1]
▶︎ ansible kafka1 -m command -a "/opt/kafka/bin/kafka-metadata-quorum.sh --bootstrap-server 192.168.25.221:9092 describe --status"
kafka1 | CHANGED | rc=0 >>
ClusterId:              4L6g3nShT-eMCtK--X86sw
LeaderId:               1
LeaderEpoch:            1
HighWatermark:          751
MaxFollowerLag:         0
MaxFollowerLagTimeMs:   0
CurrentVoters:          [1,2,3]
CurrentObservers:       []
[0 e@Gunther:~/Dropbox/Documents/Code/Kafka/kafka-lab-1]
▶︎
```

## Smoke test:

Smoke test to verify cluster minimal functionality:

```sh
[0 e@Gunther:~/Dropbox/Documents/Code/Kafka/kafka-lab-1]
▶︎ ansible kafka1 -m command -a "/opt/kafka/bin/kafka-topics.sh --bootstrap-server 192.168.25.221:9092 --create --topic lab-healthcheck --partitions 3 --replication-factor 3"
kafka1 | CHANGED | rc=0 >>
Created topic lab-healthcheck.
[0 e@Gunther:~/Dropbox/Documents/Code/Kafka/kafka-lab-1]
▶︎
▶︎ ansible kafka1 -m command -a "/opt/kafka/bin/kafka-topics.sh --bootstrap-server 192.168.25.221:9092 --describe --topic lab-healthcheck"
kafka1 | CHANGED | rc=0 >>
Topic: lab-healthcheck	TopicId: uG3KrhFqQ7mwPzmFU3wlIw	PartitionCount: 3	ReplicationFactor: 3	Configs: min.insync.replicas=2
	Topic: lab-healthcheck	Partition: 0	Leader: 2	Replicas: 2,3,1	Isr: 2,3,1
	Topic: lab-healthcheck	Partition: 1	Leader: 3	Replicas: 3,1,2	Isr: 3,1,2
	Topic: lab-healthcheck	Partition: 2	Leader: 1	Replicas: 1,2,3	Isr: 1,2,3
[0 e@Gunther:~/Dropbox/Documents/Code/Kafka/kafka-lab-1]
▶︎
[0 e@Gunther:~/Dropbox/Documents/Code/Kafka/kafka-lab-1]
▶︎ ansible kafka1 -m shell -a "echo 'cluster-operational-test' | /opt/kafka/bin/kafka-console-producer.sh --bootstrap-server 192.168.25.221:9092 --topic lab-healthcheck"
kafka1 | CHANGED | rc=0 >>

[0 e@Gunther:~/Dropbox/Documents/Code/Kafka/kafka-lab-1]
▶︎
[0 e@Gunther:~/Dropbox/Documents/Code/Kafka/kafka-lab-1]
▶︎ ansible kafka2 -m command -a "/opt/kafka/bin/kafka-console-consumer.sh --bootstrap-server 192.168.25.103:9092 --topic lab-healthcheck --from-beginning --max-messages 1 --timeout-ms 5000"
kafka2 | CHANGED | rc=0 >>
cluster-operational-testProcessed a total of 1 messages
[0 e@Gunther:~/Dropbox/Documents/Code/Kafka/kafka-lab-1]
▶︎
```

## Precreate the Target Topic

Create this topic on your cluster first so partitions and replication factors match the 3-node topology:

```sh
[0 e@Gunther:~/Dropbox/Documents/Code/Kafka/kafka-lab-1]
▶︎ ansible kafka1 -m command -a "/opt/kafka/bin/kafka-topics.sh   --bootstrap-server 192.168.25.221:9092   --create   --topic lab-telemetry   --partitions 3   --replication-factor 3"
kafka1 | CHANGED | rc=0 >>
Created topic lab-telemetry.
[0 e@Gunther:~/Dropbox/Documents/Code/Kafka/kafka-lab-1]
▶︎
```

# Configure producers

## On **producer1** (`192.168.209.133`)

### Install **podman**

```sh
[e@producer1 ~]$ sudo dnf install -y podman
[sudo] password for e:
Updating Subscription Management repositories.
Red Hat Enterprise Linux 10 for ARM 64 - AppStream (RPMs)         3.7 MB/s | 7.5 MB     00:02
Red Hat Enterprise Linux 10 for ARM 64 - BaseOS (RPMs)             28 MB/s | 156 MB     00:05
Dependencies resolved.
==================================================================================================
 Package              Arch    Version                    Repository                          Size
==================================================================================================
Installing:
 podman               aarch64 7:6.1.0-2.el10             rhel-10-for-aarch64-appstream-rpms  14 M
Installing dependencies:
 aardvark-dns         aarch64 4:2.1.0-1.el10             rhel-10-for-aarch64-appstream-rpms 771 k
 catatonit            aarch64 5:0.2.1-4.el10             rhel-10-for-aarch64-appstream-rpms 298 k
 conmon               aarch64 3:2.2.1-4.el10             rhel-10-for-aarch64-appstream-rpms  57 k
 container-selinux    noarch  4:2.235.0-2.el10_0         rhel-10-for-aarch64-appstream-rpms  62 k
 containers-common    noarch  6:6.0-3.el10               rhel-10-for-aarch64-appstream-rpms 180 k
 containers-common-extra
                      noarch  6:6.0-3.el10               rhel-10-for-aarch64-appstream-rpms  14 k
 criu                 aarch64 4.2-9.el10                 rhel-10-for-aarch64-appstream-rpms 569 k
 crun                 aarch64 1.30.1-2.el10_2            rhel-10-for-aarch64-appstream-rpms 273 k
 libnet               aarch64 1.3-7.el10                 rhel-10-for-aarch64-appstream-rpms  67 k
 netavark             aarch64 4:2.1.0-1.el10             rhel-10-for-aarch64-appstream-rpms 3.2 M
 passt                aarch64 0^20260728.gf8df3f1-1.el10 rhel-10-for-aarch64-appstream-rpms 206 k
 passt-selinux        noarch  0^20260728.gf8df3f1-1.el10 rhel-10-for-aarch64-appstream-rpms  33 k
 podman-sequoia       aarch64 0.4.0~pqc.3-1.el10_2       rhel-10-for-aarch64-appstream-rpms 2.3 M
 protobuf-c           aarch64 1.5.0-6.el10               rhel-10-for-aarch64-baseos-rpms     35 k
 shadow-utils-subid   aarch64 2:4.15.0-13.el10           rhel-10-for-aarch64-baseos-rpms     32 k
Installing weak dependencies:
 criu-libs            aarch64 4.2-9.el10                 rhel-10-for-aarch64-appstream-rpms  36 k

Transaction Summary
==================================================================================================
Install  17 Packages

Total download size: 22 M
Installed size: 72 M
Downloading Packages:
(1/17): container-selinux-2.235.0-2.el10_0.noarch.rpm              62 kB/s |  62 kB     00:01
(2/17): libnet-1.3-7.el10.aarch64.rpm                              47 kB/s |  67 kB     00:01
(3/17): podman-sequoia-0.4.0~pqc.3-1.el10_2.aarch64.rpm           1.5 MB/s | 2.3 MB     00:01
(4/17): crun-1.30.1-2.el10_2.aarch64.rpm                          745 kB/s | 273 kB     00:00
(5/17): aardvark-dns-2.1.0-1.el10.aarch64.rpm                     692 kB/s | 771 kB     00:01
(6/17): catatonit-0.2.1-4.el10.aarch64.rpm                        285 kB/s | 298 kB     00:01
(7/17): conmon-2.2.1-4.el10.aarch64.rpm                            67 kB/s |  57 kB     00:00
(8/17): containers-common-6.0-3.el10.noarch.rpm                   292 kB/s | 180 kB     00:00
(9/17): containers-common-extra-6.0-3.el10.noarch.rpm              18 kB/s |  14 kB     00:00
(10/17): criu-libs-4.2-9.el10.aarch64.rpm                          50 kB/s |  36 kB     00:00
(11/17): criu-4.2-9.el10.aarch64.rpm                              402 kB/s | 569 kB     00:01
(12/17): passt-0^20260728.gf8df3f1-1.el10.aarch64.rpm             300 kB/s | 206 kB     00:00
(13/17): netavark-2.1.0-1.el10.aarch64.rpm                        3.2 MB/s | 3.2 MB     00:00
(14/17): passt-selinux-0^20260728.gf8df3f1-1.el10.noarch.rpm       50 kB/s |  33 kB     00:00
(15/17): protobuf-c-1.5.0-6.el10.aarch64.rpm                       47 kB/s |  35 kB     00:00
(16/17): shadow-utils-subid-4.15.0-13.el10.aarch64.rpm             44 kB/s |  32 kB     00:00
(17/17): podman-6.1.0-2.el10.aarch64.rpm                           10 MB/s |  14 MB     00:01
--------------------------------------------------------------------------------------------------
Total                                                             4.0 MB/s |  22 MB     00:05
Red Hat Enterprise Linux 10 for ARM 64 - AppStream (RPMs)         3.6 MB/s | 3.7 kB     00:00
Importing GPG key 0xFD431D51:
 Userid     : "Red Hat, Inc. (release key 2) <security@redhat.com>"
 Fingerprint: 567E 347A D004 4ADE 55BA 8A5F 199E 2F91 FD43 1D51
 From       : /etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release
Key imported successfully
Importing GPG key 0x5A6340B3:
 Userid     : "Red Hat, Inc. (auxiliary key 3) <security@redhat.com>"
 Fingerprint: 7E46 2425 8C40 6535 D56D 6F13 5054 E4A4 5A63 40B3
 From       : /etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release
Key imported successfully
Running transaction check
Transaction check succeeded.
Running transaction test
Transaction test succeeded.
Running transaction
  Preparing        :                                                                          1/1
  Installing       : protobuf-c-1.5.0-6.el10.aarch64                                         1/17
  Running scriptlet: container-selinux-4:2.235.0-2.el10_0.noarch                             2/17
  Installing       : container-selinux-4:2.235.0-2.el10_0.noarch                             2/17
  Running scriptlet: container-selinux-4:2.235.0-2.el10_0.noarch                             2/17
  Running scriptlet: passt-selinux-0^20260728.gf8df3f1-1.el10.noarch                         3/17
  Installing       : passt-selinux-0^20260728.gf8df3f1-1.el10.noarch                         3/17
  Running scriptlet: passt-selinux-0^20260728.gf8df3f1-1.el10.noarch                         3/17
libsemanage.semanage_pipe_data: Child process /usr/libexec/selinux/hll/pp failed with code: 255. (No such file or directory).
libsemanage.semanage_compile_module: passt: libsepol.policydb_read: policydb module version 24 does not match my version range 4-22
. (No such file or directory).
libsemanage.semanage_compile_module: passt: libsepol.sepol_module_package_read: invalid module in module package (at section 0)
. (No such file or directory).
libsemanage.semanage_compile_module: passt: libsepol.sepol_ppfile_to_module_package: Failed to read policy package
. (No such file or directory).
libsemanage.semanage_direct_commit: Failed to compile hll files into cil files. (No such file or directory).
semodule:  Failed!

  Installing       : passt-0^20260728.gf8df3f1-1.el10.aarch64                                4/17
  Installing       : shadow-utils-subid-2:4.15.0-13.el10.aarch64                             5/17
  Installing       : conmon-3:2.2.1-4.el10.aarch64                                           6/17
  Installing       : catatonit-5:0.2.1-4.el10.aarch64                                        7/17
  Installing       : aardvark-dns-4:2.1.0-1.el10.aarch64                                     8/17
  Installing       : netavark-4:2.1.0-1.el10.aarch64                                         9/17
  Installing       : containers-common-6:6.0-3.el10.noarch                                  10/17
  Installing       : podman-sequoia-0.4.0~pqc.3-1.el10_2.aarch64                            11/17
  Installing       : libnet-1.3-7.el10.aarch64                                              12/17
  Installing       : criu-4.2-9.el10.aarch64                                                13/17
  Installing       : criu-libs-4.2-9.el10.aarch64                                           14/17
  Installing       : crun-1.30.1-2.el10_2.aarch64                                           15/17
  Installing       : containers-common-extra-6:6.0-3.el10.noarch                            16/17
  Installing       : podman-7:6.1.0-2.el10.aarch64                                          17/17
  Running scriptlet: container-selinux-4:2.235.0-2.el10_0.noarch                            17/17
  Running scriptlet: passt-selinux-0^20260728.gf8df3f1-1.el10.noarch                        17/17
  Running scriptlet: containers-common-6:6.0-3.el10.noarch                                  17/17
  Running scriptlet: podman-7:6.1.0-2.el10.aarch64                                          17/17
Installed products updated.

Installed:
  aardvark-dns-4:2.1.0-1.el10.aarch64               catatonit-5:0.2.1-4.el10.aarch64
  conmon-3:2.2.1-4.el10.aarch64                     container-selinux-4:2.235.0-2.el10_0.noarch
  containers-common-6:6.0-3.el10.noarch             containers-common-extra-6:6.0-3.el10.noarch
  criu-4.2-9.el10.aarch64                           criu-libs-4.2-9.el10.aarch64
  crun-1.30.1-2.el10_2.aarch64                      libnet-1.3-7.el10.aarch64
  netavark-4:2.1.0-1.el10.aarch64                   passt-0^20260728.gf8df3f1-1.el10.aarch64
  passt-selinux-0^20260728.gf8df3f1-1.el10.noarch   podman-7:6.1.0-2.el10.aarch64
  podman-sequoia-0.4.0~pqc.3-1.el10_2.aarch64       protobuf-c-1.5.0-6.el10.aarch64
  shadow-utils-subid-2:4.15.0-13.el10.aarch64

Complete!
[e@producer1 ~]$
```

### Space

```sh
[e@producer1 ~]$ ls -l /opt/kafka-lab/producer/
total 8
-rw-r--r--. 1 root root  257 Nov 10 09:40 Dockerfile
-rw-r--r--. 1 root root 3306 Nov 10 09:52 producer.py
[e@producer1 ~]$
[e@producer1 ~]$ sudo mkdir -vp /opt/kafka-lab/producer /etc/containers/systemd
[sudo] password for e:
mkdir: created directory '/opt/kafka-lab'
mkdir: created directory '/opt/kafka-lab/producer'
[e@producer1 ~]$
```

### Build the image

Needs `/opt/kafka-lab/producer/producer.py` and `/opt/kafka-lab/producer/Dockerfile` in place.

```sh
[e@producer1 ~]$ sudo podman build --cgroup-manager=cgroupfs -t localhost/kafka-lab-producer:latest /opt/kafka-lab/producer/          -cgroup-manager=cgroupfs -t localhost/kafka-lab-producer:latest /opt/kafka-lab/producer/
STEP 1/6: FROM registry.access.redhat.com/ubi9/python-311:latest
STEP 2/6: WORKDIR /app
--> Using cache e91ca2002526ebd5d78357552785632a01fab1f49c10b59de14dfe0c23377e30
--> e91ca2002526
STEP 3/6: RUN pip install --no-cache-dir --upgrade pip &&     pip install --no-cache-dir confluent-kafka Faker
--> Using cache b46fbc9e62e102409aa0bdd601e82a0c29260caa38fe1f45aa9ba0c31746d300
--> b46fbc9e62e1
STEP 4/6: COPY producer.py /app/producer.py
--> 23a11708f755
STEP 5/6: USER 1001
--> ea5aac5bfc85
STEP 6/6: CMD ["python3", "/app/producer.py"]
COMMIT localhost/kafka-lab-producer:latest
--> 8158209f6494
Successfully tagged localhost/kafka-lab-producer:latest
8158209f649491ec6091c5b152446eb34ccfc52f96e836f06ba28072c7d2e5d5
[e@producer1 ~]$
[e@producer1 ~]$ sudo podman image ls
REPOSITORY                    TAG         IMAGE ID      CREATED             SIZE
localhost/kafka-lab-producer  latest      8158209f6494  About a minute ago  1.16 GB
[e@producer1 ~]$
```

### Create the service to run the container

```sh
[e@producer1 ~]$ cat /etc/containers/systemd/kafka-producer@.container
[Unit]
Description=Kafka Telemetry Producer Instance %i
After=network-online.target
Wants=network-online.target

[Container]
Image=localhost/kafka-lab-producer:latest
ContainerName=kafka-producer-%i
Environment=PRODUCER_ID=producer-%i
Environment=BOOTSTRAP_SERVERS=192.168.25.221:9092,192.168.25.103:9092,192.168.25.104:9092
Environment=TARGET_TOPIC=lab-telemetry
Environment=MAX_SMALL_SLEEP=0.5

[Service]
Restart=always
RestartSec=5s

[Install]
WantedBy=multi-user.target
[e@producer1 ~]$
```

### Start two containers

```sh
[e@producer1 ~]$ sudo systemctl start kafka-producer@alpha.service
[e@producer1 ~]$ sudo systemctl start kafka-producer@beta.service
[e@producer1 ~]$ sudo systemctl status kafka-producer@alpha.service
● kafka-producer@alpha.service - Kafka Telemetry Producer Instance alpha
     Loaded: loaded (/etc/containers/systemd/kafka-producer@.container; generated)
     Active: active (running) since Tue 2026-11-10 10:01:31 -05; 9s ago
 Invocation: 0e2bc62c4a144e42a9530562b946994e
   Main PID: 3688 (conmon)
      Tasks: 7 (limit: 24779)
     Memory: 23.5M (peak: 24M)
        CPU: 306ms
     CGroup: /system.slice/system-kafka\x2dproducer.slice/kafka-producer@alpha.service
             ├─libpod-payload-b69a168a3390161045acbfb93414f49d016b35163f9526cdb9419bbfea1b8db7
             │ └─3690 python3 /app/producer.py
             └─runtime
               └─3688 /usr/bin/conmon --api-version 1 -c b69a168a3390161045acbfb93414f49d016b3516>

Nov 10 10:01:31 producer1.local systemd[1]: Starting kafka-producer@alpha.service - Kafka Telemet>
Nov 10 10:01:31 producer1.local podman[3649]: 2026-11-10 10:01:31.622906751 -0500 -05 m=+0.056206>
Nov 10 10:01:31 producer1.local podman[3649]: 2026-11-10 10:01:31.613659165 -0500 -05 m=+0.046958>
Nov 10 10:01:31 producer1.local podman[3649]: 2026-11-10 10:01:31.760143986 -0500 -05 m=+0.193443>
Nov 10 10:01:31 producer1.local systemd[1]: Started kafka-producer@alpha.service - Kafka Telemetr>
Nov 10 10:01:31 producer1.local podman[3649]: 2026-11-10 10:01:31.762211736 -0500 -05 m=+0.195511>
Nov 10 10:01:31 producer1.local kafka-producer@alpha[3649]: b69a168a3390161045acbfb93414f49d016b3>
Nov 10 10:01:31 producer1.local kafka-producer-alpha[3688]: 2026-11-10 15:01:31,853 [INFO] [PRODU>
Nov 10 10:01:31 producer1.local kafka-producer-alpha[3688]: 2026-11-10 15:01:31,853 [INFO] [PRODU>
[e@producer1 ~]$ sudo systemctl status kafka-producer@beta.service
● kafka-producer@beta.service - Kafka Telemetry Producer Instance beta
     Loaded: loaded (/etc/containers/systemd/kafka-producer@.container; generated)
     Active: active (running) since Tue 2026-11-10 10:01:36 -05; 11s ago
 Invocation: 343506f177d04d41ab789ae56dc7a58c
   Main PID: 3730 (conmon)
      Tasks: 7 (limit: 24779)
     Memory: 23.7M (peak: 25.3M)
        CPU: 288ms
     CGroup: /system.slice/system-kafka\x2dproducer.slice/kafka-producer@beta.service
             ├─libpod-payload-c67608a059e654302627d1a7cba4ab47c85e114d3543c8527fa07f65039e8b30
             │ └─3733 python3 /app/producer.py
             └─runtime
               └─3730 /usr/bin/conmon --api-version 1 -c c67608a059e654302627d1a7cba4ab47c85e114d>

Nov 10 10:01:36 producer1.local systemd[1]: Starting kafka-producer@beta.service - Kafka Telemetr>
Nov 10 10:01:36 producer1.local podman[3706]: 2026-11-10 10:01:36.521033536 -0500 -05 m=+0.055192>
Nov 10 10:01:36 producer1.local podman[3706]: 2026-11-10 10:01:36.512185918 -0500 -05 m=+0.046345>
Nov 10 10:01:36 producer1.local podman[3706]: 2026-11-10 10:01:36.691023498 -0500 -05 m=+0.225182>
Nov 10 10:01:36 producer1.local systemd[1]: Started kafka-producer@beta.service - Kafka Telemetry>
Nov 10 10:01:36 producer1.local podman[3706]: 2026-11-10 10:01:36.69493364 -0500 -05 m=+0.2290928>
Nov 10 10:01:36 producer1.local kafka-producer@beta[3706]: c67608a059e654302627d1a7cba4ab47c85e11>
Nov 10 10:01:36 producer1.local kafka-producer-beta[3730]: 2026-11-10 15:01:36,774 [INFO] [PRODUC>
Nov 10 10:01:36 producer1.local kafka-producer-beta[3730]: 2026-11-10 15:01:36,774 [INFO] [PRODUC>
[e@producer1 ~]$
```

### Follow a producer to see the effect

```sh
sudo journalctl -u kafka-producer@alpha.service -u kafka-producer@beta.service -f -n 20
```

### Pushed info validation

#### Topics

```sh
[0 e@Gunther:~/Dropbox/Documents/Code/Kafka/kafka-lab-1]
▶︎ ansible kafka1 -m command -a "/opt/kafka/bin/kafka-topics.sh --bootstrap-server 192.168.25.221:9092 --list"
kafka1 | CHANGED | rc=0 >>
__consumer_offsets
lab-healthcheck
lab-telemetry
[0 e@Gunther:~/Dropbox/Documents/Code/Kafka/kafka-lab-1]
▶︎
[0 e@Gunther:~/Dropbox/Documents/Code/Kafka/kafka-lab-1]
▶︎ ansible kafka1 -m command -a "/opt/kafka/bin/kafka-topics.sh --bootstrap-server 192.168.25.221:9092 --describe"
kafka1 | CHANGED | rc=0 >>
Topic: __consumer_offsets	TopicId: 4LlKbCsvSU6ZVdG9IgzNfA	PartitionCount: 50	ReplicationFactor: 3	Configs: compression.type=producer,min.insync.replicas=2,cleanup.policy=compact,segment.bytes=104857600
	Topic: __consumer_offsets	Partition: 0	Leader: 1	Replicas: 1,2,3	Isr: 1,2,3
	Topic: __consumer_offsets	Partition: 1	Leader: 2	Replicas: 2,3,1	Isr: 2,3,1
	Topic: __consumer_offsets	Partition: 2	Leader: 3	Replicas: 3,1,2	Isr: 3,1,2
	Topic: __consumer_offsets	Partition: 3	Leader: 1	Replicas: 1,2,3	Isr: 1,2,3
	Topic: __consumer_offsets	Partition: 4	Leader: 2	Replicas: 2,3,1	Isr: 2,3,1
	Topic: __consumer_offsets	Partition: 5	Leader: 3	Replicas: 3,1,2	Isr: 3,1,2
	Topic: __consumer_offsets	Partition: 6	Leader: 2	Replicas: 2,1,3	Isr: 2,1,3
	Topic: __consumer_offsets	Partition: 7	Leader: 1	Replicas: 1,3,2	Isr: 1,3,2
	Topic: __consumer_offsets	Partition: 8	Leader: 3	Replicas: 3,2,1	Isr: 3,2,1
	Topic: __consumer_offsets	Partition: 9	Leader: 3	Replicas: 3,2,1	Isr: 3,2,1
	Topic: __consumer_offsets	Partition: 10	Leader: 2	Replicas: 2,1,3	Isr: 2,1,3
	Topic: __consumer_offsets	Partition: 11	Leader: 1	Replicas: 1,3,2	Isr: 1,3,2
	Topic: __consumer_offsets	Partition: 12	Leader: 3	Replicas: 3,2,1	Isr: 3,2,1
	Topic: __consumer_offsets	Partition: 13	Leader: 2	Replicas: 2,1,3	Isr: 2,1,3
	Topic: __consumer_offsets	Partition: 14	Leader: 1	Replicas: 1,3,2	Isr: 1,3,2
	Topic: __consumer_offsets	Partition: 15	Leader: 3	Replicas: 3,2,1	Isr: 3,2,1
	Topic: __consumer_offsets	Partition: 16	Leader: 2	Replicas: 2,1,3	Isr: 2,1,3
	Topic: __consumer_offsets	Partition: 17	Leader: 1	Replicas: 1,3,2	Isr: 1,3,2
	Topic: __consumer_offsets	Partition: 18	Leader: 2	Replicas: 2,3,1	Isr: 2,3,1
	Topic: __consumer_offsets	Partition: 19	Leader: 3	Replicas: 3,1,2	Isr: 3,1,2
	Topic: __consumer_offsets	Partition: 20	Leader: 1	Replicas: 1,2,3	Isr: 1,2,3
	Topic: __consumer_offsets	Partition: 21	Leader: 2	Replicas: 2,3,1	Isr: 2,3,1
	Topic: __consumer_offsets	Partition: 22	Leader: 3	Replicas: 3,1,2	Isr: 3,1,2
	Topic: __consumer_offsets	Partition: 23	Leader: 1	Replicas: 1,2,3	Isr: 1,2,3
	Topic: __consumer_offsets	Partition: 24	Leader: 2	Replicas: 2,1,3	Isr: 2,1,3
	Topic: __consumer_offsets	Partition: 25	Leader: 1	Replicas: 1,3,2	Isr: 1,3,2
	Topic: __consumer_offsets	Partition: 26	Leader: 3	Replicas: 3,2,1	Isr: 3,2,1
	Topic: __consumer_offsets	Partition: 27	Leader: 2	Replicas: 2,3,1	Isr: 2,3,1
	Topic: __consumer_offsets	Partition: 28	Leader: 3	Replicas: 3,1,2	Isr: 3,1,2
	Topic: __consumer_offsets	Partition: 29	Leader: 1	Replicas: 1,2,3	Isr: 1,2,3
	Topic: __consumer_offsets	Partition: 30	Leader: 1	Replicas: 1,2,3	Isr: 1,2,3
	Topic: __consumer_offsets	Partition: 31	Leader: 2	Replicas: 2,3,1	Isr: 2,3,1
	Topic: __consumer_offsets	Partition: 32	Leader: 3	Replicas: 3,1,2	Isr: 3,1,2
	Topic: __consumer_offsets	Partition: 33	Leader: 3	Replicas: 3,1,2	Isr: 3,1,2
	Topic: __consumer_offsets	Partition: 34	Leader: 1	Replicas: 1,2,3	Isr: 1,2,3
	Topic: __consumer_offsets	Partition: 35	Leader: 2	Replicas: 2,3,1	Isr: 2,3,1
	Topic: __consumer_offsets	Partition: 36	Leader: 1	Replicas: 1,3,2	Isr: 1,3,2
	Topic: __consumer_offsets	Partition: 37	Leader: 3	Replicas: 3,2,1	Isr: 3,2,1
	Topic: __consumer_offsets	Partition: 38	Leader: 2	Replicas: 2,1,3	Isr: 2,1,3
	Topic: __consumer_offsets	Partition: 39	Leader: 3	Replicas: 3,2,1	Isr: 3,2,1
	Topic: __consumer_offsets	Partition: 40	Leader: 2	Replicas: 2,1,3	Isr: 2,1,3
	Topic: __consumer_offsets	Partition: 41	Leader: 1	Replicas: 1,3,2	Isr: 1,3,2
	Topic: __consumer_offsets	Partition: 42	Leader: 2	Replicas: 2,1,3	Isr: 2,1,3
	Topic: __consumer_offsets	Partition: 43	Leader: 1	Replicas: 1,3,2	Isr: 1,3,2
	Topic: __consumer_offsets	Partition: 44	Leader: 3	Replicas: 3,2,1	Isr: 3,2,1
	Topic: __consumer_offsets	Partition: 45	Leader: 2	Replicas: 2,1,3	Isr: 2,1,3
	Topic: __consumer_offsets	Partition: 46	Leader: 1	Replicas: 1,3,2	Isr: 1,3,2
	Topic: __consumer_offsets	Partition: 47	Leader: 3	Replicas: 3,2,1	Isr: 3,2,1
	Topic: __consumer_offsets	Partition: 48	Leader: 2	Replicas: 2,1,3	Isr: 2,1,3
	Topic: __consumer_offsets	Partition: 49	Leader: 1	Replicas: 1,3,2	Isr: 1,3,2
Topic: lab-healthcheck	TopicId: uG3KrhFqQ7mwPzmFU3wlIw	PartitionCount: 3	ReplicationFactor: 3	Configs: min.insync.replicas=2
	Topic: lab-healthcheck	Partition: 0	Leader: 2	Replicas: 2,3,1	Isr: 2,3,1
	Topic: lab-healthcheck	Partition: 1	Leader: 3	Replicas: 3,1,2	Isr: 3,1,2
	Topic: lab-healthcheck	Partition: 2	Leader: 1	Replicas: 1,2,3	Isr: 1,2,3
Topic: lab-telemetry	TopicId: eHLvxdOHTdW6GZkpTgt91Q	PartitionCount: 3	ReplicationFactor: 3	Configs: min.insync.replicas=2
	Topic: lab-telemetry	Partition: 0	Leader: 2	Replicas: 2,3,1	Isr: 2,3,1
	Topic: lab-telemetry	Partition: 1	Leader: 3	Replicas: 3,1,2	Isr: 3,1,2
	Topic: lab-telemetry	Partition: 2	Leader: 1	Replicas: 1,2,3	Isr: 1,2,3
[0 e@Gunther:~/Dropbox/Documents/Code/Kafka/kafka-lab-1]
▶︎
[0 e@Gunther:~/Dropbox/Documents/Code/Kafka/kafka-lab-1]
▶︎ ansible kafka1 -m command -a "/opt/kafka/bin/kafka-topics.sh --bootstrap-server 192.168.25.221:9092 --describe --exclude-internal"
kafka1 | CHANGED | rc=0 >>
Topic: lab-healthcheck	TopicId: uG3KrhFqQ7mwPzmFU3wlIw	PartitionCount: 3	ReplicationFactor: 3	Configs: min.insync.replicas=2
	Topic: lab-healthcheck	Partition: 0	Leader: 2	Replicas: 2,3,1	Isr: 2,3,1
	Topic: lab-healthcheck	Partition: 1	Leader: 3	Replicas: 3,1,2	Isr: 3,1,2
	Topic: lab-healthcheck	Partition: 2	Leader: 1	Replicas: 1,2,3	Isr: 1,2,3
Topic: lab-telemetry	TopicId: eHLvxdOHTdW6GZkpTgt91Q	PartitionCount: 3	ReplicationFactor: 3	Configs: min.insync.replicas=2
	Topic: lab-telemetry	Partition: 0	Leader: 2	Replicas: 2,3,1	Isr: 2,3,1
	Topic: lab-telemetry	Partition: 1	Leader: 3	Replicas: 3,1,2	Isr: 3,1,2
	Topic: lab-telemetry	Partition: 2	Leader: 1	Replicas: 1,2,3	Isr: 1,2,3
[0 e@Gunther:~/Dropbox/Documents/Code/Kafka/kafka-lab-1]
▶︎
```

#### List use of one of the topics (`lab-telemetry`)

````sh
[0 e@Gunther:~/Dropbox/Documents/Code/Kafka/kafka-lab-1]
▶︎ ansible kafka1 -m command -a "/opt/kafka/bin/kafka-get-offsets.sh --bootstrap-server 192.168.25.221:9092 --topic lab-telemetry --time -1"
kafka1 | CHANGED | rc=0 >>
lab-telemetry:0:799
lab-telemetry:1:740
lab-telemetry:2:778
[0 e@Gunther:~/Dropbox/Documents/Code/Kafka/kafka-lab-1]
▶︎ sleep 10
ansible kafka1 -m command -a "/opt/kafka/bin/kafka-get-offsets.sh --bootstrap-server 192.168.25.221:9092 --topic lab-telemetry --time -1"
[0 e@Gunther:~/Dropbox/Documents/Code/Kafka/kafka-lab-1]
▶︎ ansible kafka1 -m command -a "/opt/kafka/bin/kafka-get-offsets.sh --bootstrap-server 192.168.25.221:9092 --topic lab-telemetry --time -1"
kafka1 | CHANGED | rc=0 >>
lab-telemetry:0:849
lab-telemetry:1:782
lab-telemetry:2:819
[0 e@Gunther:~/Dropbox/Documents/Code/Kafka/kafka-lab-1]
▶︎
### Validation that all producers are represented:
```sh
[0 e@Gunther:~/Dropbox/Documents/Code/Kafka/kafka-lab-1]
▶︎ ansible kafka1 -m command -a "/opt/kafka/bin/kafka-console-consumer.sh --bootstrap-server 192.168.25.221:9092 --topic lab-telemetry --max-messages 6 --property print.partition=true --property print.key=true"
kafka1 | CHANGED | rc=0 >>
Partition:0	1aa8da04-024c-44a8-bc88-5c8203e1bebc	{"producer_id": "producer-alpha", "event_id": "1aa8da04-024c-44a8-bc88-5c8203e1bebc", "timestamp_ns": 1794323261949334585, "user": "Adam Peterson", "email": "mark99@hotmail.com", "ipv4": "80.30.71.254", "transaction_value": 511.13}
Partition:2	9ae98c97-9a19-47a4-9d80-abce09870502	{"producer_id": "producer-beta", "event_id": "9ae98c97-9a19-47a4-9d80-abce09870502", "timestamp_ns": 1794323262215682741, "user": "Stephanie Alvarez", "email": "ryanjones@gmail.com", "ipv4": "75.126.69.109", "transaction_value": 15.44}
Partition:0	132fbc1c-8a2f-4de6-83a3-f9c96dc79bbf	{"producer_id": "producer-alpha", "event_id": "132fbc1c-8a2f-4de6-83a3-f9c96dc79bbf", "timestamp_ns": 1794323262440266868, "user": "Patricia Paul", "email": "gramos@gmail.com", "ipv4": "208.212.93.56", "transaction_value": 500.44}
Partition:1	1ca7a8be-a267-45c1-b5a8-a229d78f3858	{"producer_id": "producer-beta", "event_id": "1ca7a8be-a267-45c1-b5a8-a229d78f3858", "timestamp_ns": 1794323262660511431, "user": "Danielle Simmons", "email": "lisa36@yahoo.com", "ipv4": "77.126.83.107", "transaction_value": 618.51}
Partition:2	82a9872b-bf3b-4a30-8e81-9f34595376fa	{"producer_id": "producer-alpha", "event_id": "82a9872b-bf3b-4a30-8e81-9f34595376fa", "timestamp_ns": 1794323262883661744, "user": "Sarah Fisher", "email": "angel88@gmail.com", "ipv4": "24.20.129.226", "transaction_value": 451.79}
Partition:1	7065d7b6-8aa3-45d3-9aee-dfbcfe0dcf54	{"producer_id": "producer-beta", "event_id": "7065d7b6-8aa3-45d3-9aee-dfbcfe0dcf54", "timestamp_ns": 1794323262903050279, "user": "Julie Stewart", "email": "kevinhahn@yahoo.com", "ipv4": "150.41.84.157", "transaction_value": 132.56}Processed a total of 6 messages
[0 e@Gunther:~/Dropbox/Documents/Code/Kafka/kafka-lab-1]
▶︎
````

## On **producer2** (`192.168.209.134`)

### Install **podman**

```sh

```

## On **consumer** (QNAP)

### Deploy

```
╭[0 superuser🐧Hermes ~]
╰→ docker run -d \
╰→    --name kafka-qnap-consumer \
╰→    --restart unless-stopped \
╰→    --network bridge \
╰→    apache/kafka:3.7.0 \
╰→    /opt/kafka/bin/kafka-console-consumer.sh \
╰→      --bootstrap-server 192.168.25.221:9092,192.168.25.103:9092,192.168.25.104:9092 \
╰→      --topic lab-telemetry \
╰→      --group qnap-storage-analytics \
╰→      --property print.partition=true \
╰→      --property print.key=true
d1f8160c7f05b6da6c0b4f01718a6999f5f4e34a3520921c5d3881355432ca3d
╭[0 superuser🐧Hermes ~]
╰→
```

### Check it is running

```sh
╭[0 e@Hermes ~]
╰→ docker logs kafka-qnap-consumer --tail 10
Partition:0	479b0544-242b-46b5-8993-4fe3866bb8d0	{"producer_id": "producer-beta", "event_id": "479b0544-242b-46b5-8993-4fe3866bb8d0", "timestamp_ns": 1794325683993936221, "user": "Karen Stevenson", "email": "sarafernandez@yahoo.com", "ipv4": "126.108.73.196", "transaction_value": 576.1}
Partition:1	725edbfa-aed4-4eee-ab72-02713de3fb8f	{"producer_id": "producer-beta", "event_id": "725edbfa-aed4-4eee-ab72-02713de3fb8f", "timestamp_ns": 1794325684326466032, "user": "Edward Skinner", "email": "amanda64@yahoo.com", "ipv4": "187.158.67.76", "transaction_value": 522.46}
Partition:0	1dfece8f-3252-4ef7-b692-03b263d40caa	{"producer_id": "producer-alpha", "event_id": "1dfece8f-3252-4ef7-b692-03b263d40caa", "timestamp_ns": 1794325684450475760, "user": "Christopher Vasquez", "email": "catherine66@hotmail.com", "ipv4": "68.184.50.60", "transaction_value": 326.23}
Partition:0	b64f0e56-23f5-413d-b037-5757747facc3	{"producer_id": "producer-beta", "event_id": "b64f0e56-23f5-413d-b037-5757747facc3", "timestamp_ns": 1794325684613038650, "user": "Megan Thomas", "email": "ubrown@yahoo.com", "ipv4": "118.20.171.164", "transaction_value": 506.22}
Partition:1	c437084a-c08f-40b3-af66-75f5190fa9fd	{"producer_id": "producer-alpha", "event_id": "c437084a-c08f-40b3-af66-75f5190fa9fd", "timestamp_ns": 1794325684694274949, "user": "Sandra Rodriguez", "email": "hobrien@yahoo.com", "ipv4": "29.43.45.126", "transaction_value": 72.04}
Partition:1	3dd17c56-ad34-4c1e-b9d0-73cfe6ad90d2	{"producer_id": "producer-beta", "event_id": "3dd17c56-ad34-4c1e-b9d0-73cfe6ad90d2", "timestamp_ns": 1794325685075846674, "user": "James Barr", "email": "michaelsmith@hotmail.com", "ipv4": "207.226.250.44", "transaction_value": 552.58}
Partition:0	f9258918-88ec-4888-95fc-5f16a25b30b4	{"producer_id": "producer-beta", "event_id": "f9258918-88ec-4888-95fc-5f16a25b30b4", "timestamp_ns": 1794325685415204013, "user": "Paul Yang", "email": "cassandra39@gmail.com", "ipv4": "116.118.77.149", "transaction_value": 596.58}
Partition:0	72d575a6-10c3-469e-a9a9-8488f745d97a	{"producer_id": "producer-alpha", "event_id": "72d575a6-10c3-469e-a9a9-8488f745d97a", "timestamp_ns": 1794325685772382969, "user": "James Hampton", "email": "robertdickerson@hotmail.com", "ipv4": "118.33.91.49", "transaction_value": 460.4}
Partition:1	1e8d3b43-2155-4a67-8049-c108bc3a2658	{"producer_id": "producer-beta", "event_id": "1e8d3b43-2155-4a67-8049-c108bc3a2658", "timestamp_ns": 1794325685909746420, "user": "James Moore", "email": "davisalbert@hotmail.com", "ipv4": "22.221.207.97", "transaction_value": 555.24}
Partition:1	20e983f8-b2b7-40c7-aea4-810ffec807bc	{"producer_id": "producer-alpha", "event_id": "20e983f8-b2b7-40c7-aea4-810ffec807bc", "timestamp_ns": 1794325685982623434, "user": "Phillip Rich", "email": "ellisonrobert@gmail.com", "ipv4": "116.162.3.192", "transaction_value": 32.95}
╭[0 e@Hermes ~]
╰→
```

### On cluster, check consumer group commit and lag on the cluster

```sh
0 e@Gunther:~/Dropbox/Documents/Code/Kafka/kafka-lab-1]
▶︎ ansible kafka1 -m command -a "/opt/kafka/bin/kafka-consumer-groups.sh --bootstrap-server 192.168.25.221:9092 --describe --group qnap-storage-analytics"
kafka1 | CHANGED | rc=0 >>

GROUP                  TOPIC           PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG             CONSUMER-ID                                           HOST            CLIENT-ID
qnap-storage-analytics lab-telemetry   0          7527            7538            11              console-consumer-6bb513ea-5723-4066-9164-9577067ac6a7 /192.168.25.15  console-consumer
qnap-storage-analytics lab-telemetry   1          7396            7411            15              console-consumer-6bb513ea-5723-4066-9164-9577067ac6a7 /192.168.25.15  console-consumer
qnap-storage-analytics lab-telemetry   2          7364            7378            14              console-consumer-7f90609d-8416-4d74-a5d9-aedb0cc90981 /192.168.25.15  console-consumer
[0 e@Gunther:~/Dropbox/Documents/Code/Kafka/kafka-lab-1]
▶︎
```

There are three nodes with three partitions and two consumers, then there is an asymmetrical distribution onf consumption. A lag of 1 second is normal (there are 13 msg/s produced).

## Test shutting down a node

### Shut it down

```sh
[0 e@Gunther:~/Dropbox/Documents/Code/Kafka/kafka-lab-1]
▶︎ ansible kafka3 -b -m systemd -a "name=kafka state=stopped"
kafka3 | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3.12"
    },
    "changed": true,
    "name": "kafka",
    "state": "stopped",
    "status": {
        "AccessSELinuxContext": "system_u:object_r:systemd_unit_file_t:s0",
        "ActiveEnterTimestamp": "Tue 2026-11-10 07:48:45 -05",
        "ActiveEnterTimestampMonotonic": "41460958441",
        "ActiveExitTimestampMonotonic": "0",
        "ActiveState": "active",
        "After": "sysinit.target system.slice systemd-journald.socket network.target basic.target",
        "AllowIsolate": "no",
        "AssertResult": "yes",
        "AssertTimestamp": "Tue 2026-11-10 07:48:45 -05",
        "AssertTimestampMonotonic": "41460957108",
        "Before": "multi-user.target shutdown.target",
        "BindLogSockets": "no",
        "BlockIOAccounting": "no",
        "BlockIOWeight": "[not set]",
        "CPUAccounting": "yes",
        "CPUAffinityFromNUMA": "no",
        "CPUQuotaPerSecUSec": "infinity",
        "CPUQuotaPeriodUSec": "infinity",
        "CPUSchedulingPolicy": "0",
        "CPUSchedulingPriority": "0",
        "CPUSchedulingResetOnFork": "no",
        "CPUShares": "[not set]",
        "CPUUsageNSec": "379486608000",
        "CPUWeight": "[not set]",
        "CacheDirectoryMode": "0755",
        "CanFreeze": "yes",
        "CanIsolate": "no",
        "CanLiveMount": "no",
        "CanReload": "no",
        "CanStart": "yes",
        "CanStop": "yes",
        "CapabilityBoundingSet": "cap_chown cap_dac_override cap_dac_read_search cap_fowner cap_fsetid cap_kill cap_setgid cap_setuid cap_setpcap cap_linux_immutable cap_net_bind_service cap_net_broadcast cap_net_admin cap_net_raw cap_ipc_lock cap_ipc_owner cap_sys_module cap_sys_rawio cap_sys_chroot cap_sys_ptrace cap_sys_pacct cap_sys_admin cap_sys_boot cap_sys_nice cap_sys_resource cap_sys_time cap_sys_tty_config cap_mknod cap_lease cap_audit_write cap_audit_control cap_setfcap cap_mac_override cap_mac_admin cap_syslog cap_wake_alarm cap_block_suspend cap_audit_read cap_perfmon cap_bpf cap_checkpoint_restore",
        "CleanResult": "success",
        "CollectMode": "inactive",
        "ConditionResult": "yes",
        "ConditionTimestamp": "Tue 2026-11-10 07:48:45 -05",
        "ConditionTimestampMonotonic": "41460957104",
        "ConfigurationDirectoryMode": "0755",
        "Conflicts": "shutdown.target",
        "ControlGroup": "/system.slice/kafka.service",
        "ControlGroupId": "36099",
        "ControlPID": "0",
        "CoredumpFilter": "0x33",
        "CoredumpReceive": "no",
        "DebugInvocation": "no",
        "DefaultDependencies": "yes",
        "DefaultMemoryLow": "0",
        "DefaultMemoryMin": "0",
        "DefaultStartupMemoryLow": "0",
        "Delegate": "no",
        "Description": "Apache Kafka Distributed Message Broker (KRaft Mode)",
        "DevicePolicy": "auto",
        "Documentation": "https://kafka.apache.org/documentation/",
        "DynamicUser": "no",
        "EffectiveMemoryHigh": "4087136256",
        "EffectiveMemoryMax": "4087136256",
        "EffectiveTasksMax": "24779",
        "Environment": "JAVA_HOME=/etc/alternatives/jre LOG_DIR=/var/log/kafka",
        "ExecMainCode": "0",
        "ExecMainExitTimestampMonotonic": "0",
        "ExecMainHandoffTimestamp": "Tue 2026-11-10 07:48:45 -05",
        "ExecMainHandoffTimestampMonotonic": "41461001271",
        "ExecMainPID": "266999",
        "ExecMainStartTimestamp": "Tue 2026-11-10 07:48:45 -05",
        "ExecMainStartTimestampMonotonic": "41460957882",
        "ExecMainStatus": "0",
        "ExecStart": "{ path=/opt/kafka/bin/kafka-server-start.sh ; argv[]=/opt/kafka/bin/kafka-server-start.sh /opt/kafka/config/kraft/server.properties ; ignore_errors=no ; start_time=[n/a] ; stop_time=[n/a] ; pid=0 ; code=(null) ; status=0/0 }",
        "ExecStartEx": "{ path=/opt/kafka/bin/kafka-server-start.sh ; argv[]=/opt/kafka/bin/kafka-server-start.sh /opt/kafka/config/kraft/server.properties ; flags= ; start_time=[n/a] ; stop_time=[n/a] ; pid=0 ; code=(null) ; status=0/0 }",
        "ExecStop": "{ path=/opt/kafka/bin/kafka-server-stop.sh ; argv[]=/opt/kafka/bin/kafka-server-stop.sh ; ignore_errors=no ; start_time=[n/a] ; stop_time=[n/a] ; pid=0 ; code=(null) ; status=0/0 }",
        "ExecStopEx": "{ path=/opt/kafka/bin/kafka-server-stop.sh ; argv[]=/opt/kafka/bin/kafka-server-stop.sh ; flags= ; start_time=[n/a] ; stop_time=[n/a] ; pid=0 ; code=(null) ; status=0/0 }",
        "ExitType": "main",
        "ExtensionImagePolicy": "root=verity+signed+encrypted+unprotected+absent:usr=verity+signed+encrypted+unprotected+absent:home=encrypted+unprotected+absent:srv=encrypted+unprotected+absent:tmp=encrypted+unprotected+absent:var=encrypted+unprotected+absent",
        "FailureAction": "none",
        "FileDescriptorStoreMax": "0",
        "FileDescriptorStorePreserve": "restart",
        "FinalKillSignal": "9",
        "FragmentPath": "/etc/systemd/system/kafka.service",
        "FreezerState": "running",
        "GID": "992",
        "Group": "kafka",
        "GuessMainPID": "yes",
        "IOAccounting": "no",
        "IOReadBytes": "[not set]",
        "IOReadOperations": "[not set]",
        "IOSchedulingClass": "2",
        "IOSchedulingPriority": "4",
        "IOWeight": "[not set]",
        "IOWriteBytes": "[not set]",
        "IOWriteOperations": "[not set]",
        "IPAccounting": "no",
        "IPEgressBytes": "[no data]",
        "IPEgressPackets": "[no data]",
        "IPIngressBytes": "[no data]",
        "IPIngressPackets": "[no data]",
        "Id": "kafka.service",
        "IgnoreOnIsolate": "no",
        "IgnoreSIGPIPE": "yes",
        "InactiveEnterTimestampMonotonic": "0",
        "InactiveExitTimestamp": "Tue 2026-11-10 07:48:45 -05",
        "InactiveExitTimestampMonotonic": "41460958441",
        "InvocationID": "7d856e9fee044c8d8ecfd08604cade29",
        "JobRunningTimeoutUSec": "infinity",
        "JobTimeoutAction": "none",
        "JobTimeoutUSec": "infinity",
        "KeyringMode": "private",
        "KillMode": "control-group",
        "KillSignal": "15",
        "LimitAS": "infinity",
        "LimitASSoft": "infinity",
        "LimitCORE": "infinity",
        "LimitCORESoft": "infinity",
        "LimitCPU": "infinity",
        "LimitCPUSoft": "infinity",
        "LimitDATA": "infinity",
        "LimitDATASoft": "infinity",
        "LimitFSIZE": "infinity",
        "LimitFSIZESoft": "infinity",
        "LimitLOCKS": "infinity",
        "LimitLOCKSSoft": "infinity",
        "LimitMEMLOCK": "8388608",
        "LimitMEMLOCKSoft": "8388608",
        "LimitMSGQUEUE": "819200",
        "LimitMSGQUEUESoft": "819200",
        "LimitNICE": "0",
        "LimitNICESoft": "0",
        "LimitNOFILE": "65536",
        "LimitNOFILESoft": "65536",
        "LimitNPROC": "15487",
        "LimitNPROCSoft": "15487",
        "LimitRSS": "infinity",
        "LimitRSSSoft": "infinity",
        "LimitRTPRIO": "0",
        "LimitRTPRIOSoft": "0",
        "LimitRTTIME": "infinity",
        "LimitRTTIMESoft": "infinity",
        "LimitSIGPENDING": "15487",
        "LimitSIGPENDINGSoft": "15487",
        "LimitSTACK": "infinity",
        "LimitSTACKSoft": "8388608",
        "LiveMountResult": "success",
        "LoadState": "loaded",
        "LockPersonality": "no",
        "LogLevelMax": "-1",
        "LogRateLimitBurst": "0",
        "LogRateLimitIntervalUSec": "0",
        "LogsDirectoryMode": "0755",
        "MainPID": "266999",
        "ManagedOOMMemoryPressure": "auto",
        "ManagedOOMMemoryPressureDurationUSec": "[not set]",
        "ManagedOOMMemoryPressureLimit": "0",
        "ManagedOOMPreference": "none",
        "ManagedOOMSwap": "auto",
        "MemoryAccounting": "yes",
        "MemoryAvailable": "2964058112",
        "MemoryCurrent": "636235776",
        "MemoryDenyWriteExecute": "no",
        "MemoryHigh": "infinity",
        "MemoryKSM": "no",
        "MemoryLimit": "infinity",
        "MemoryLow": "0",
        "MemoryMax": "infinity",
        "MemoryMin": "0",
        "MemoryPeak": "636760064",
        "MemoryPressureThresholdUSec": "200ms",
        "MemoryPressureWatch": "auto",
        "MemorySwapCurrent": "0",
        "MemorySwapMax": "infinity",
        "MemorySwapPeak": "0",
        "MemoryZSwapCurrent": "0",
        "MemoryZSwapMax": "infinity",
        "MemoryZSwapWriteback": "yes",
        "MountAPIVFS": "no",
        "MountImagePolicy": "root=verity+signed+encrypted+unprotected+absent:usr=verity+signed+encrypted+unprotected+absent:home=encrypted+unprotected+absent:srv=encrypted+unprotected+absent:tmp=encrypted+unprotected+absent:var=encrypted+unprotected+absent",
        "NFileDescriptorStore": "0",
        "NRestarts": "0",
        "NUMAPolicy": "n/a",
        "Names": "kafka.service",
        "NeedDaemonReload": "no",
        "Nice": "0",
        "NoNewPrivileges": "no",
        "NonBlocking": "no",
        "NotifyAccess": "none",
        "OOMPolicy": "stop",
        "OOMScoreAdjust": "0",
        "OnFailureJobMode": "replace",
        "OnSuccessJobMode": "fail",
        "Perpetual": "no",
        "PrivateDevices": "no",
        "PrivateIPC": "no",
        "PrivateMounts": "no",
        "PrivateNetwork": "no",
        "PrivatePIDs": "no",
        "PrivateTmp": "no",
        "PrivateTmpEx": "no",
        "PrivateUsers": "no",
        "PrivateUsersEx": "no",
        "ProcSubset": "all",
        "ProtectClock": "no",
        "ProtectControlGroups": "no",
        "ProtectControlGroupsEx": "no",
        "ProtectHome": "no",
        "ProtectHostname": "no",
        "ProtectKernelLogs": "no",
        "ProtectKernelModules": "no",
        "ProtectKernelTunables": "no",
        "ProtectProc": "default",
        "ProtectSystem": "no",
        "RefuseManualStart": "no",
        "RefuseManualStop": "no",
        "ReloadResult": "success",
        "ReloadSignal": "1",
        "RemainAfterExit": "no",
        "RemoveIPC": "no",
        "Requires": "system.slice sysinit.target",
        "Restart": "on-failure",
        "RestartKillSignal": "15",
        "RestartMaxDelayUSec": "infinity",
        "RestartMode": "normal",
        "RestartSteps": "0",
        "RestartUSec": "10s",
        "RestartUSecNext": "10s",
        "RestrictNamespaces": "no",
        "RestrictRealtime": "no",
        "RestrictSUIDSGID": "no",
        "Result": "success",
        "RootDirectoryStartOnly": "no",
        "RootEphemeral": "no",
        "RootImagePolicy": "root=verity+signed+encrypted+unprotected+absent:usr=verity+signed+encrypted+unprotected+absent:home=encrypted+unprotected+absent:srv=encrypted+unprotected+absent:tmp=encrypted+unprotected+absent:var=encrypted+unprotected+absent",
        "RuntimeDirectoryMode": "0755",
        "RuntimeDirectoryPreserve": "no",
        "RuntimeMaxUSec": "infinity",
        "RuntimeRandomizedExtraUSec": "0",
        "SameProcessGroup": "no",
        "SecureBits": "0",
        "SendSIGHUP": "no",
        "SendSIGKILL": "yes",
        "SetLoginEnvironment": "yes",
        "Slice": "system.slice",
        "StandardError": "inherit",
        "StandardInput": "null",
        "StandardOutput": "journal",
        "StartLimitAction": "none",
        "StartLimitBurst": "5",
        "StartLimitIntervalUSec": "10s",
        "StartupBlockIOWeight": "[not set]",
        "StartupCPUShares": "[not set]",
        "StartupCPUWeight": "[not set]",
        "StartupIOWeight": "[not set]",
        "StartupMemoryHigh": "infinity",
        "StartupMemoryLow": "0",
        "StartupMemoryMax": "infinity",
        "StartupMemorySwapMax": "infinity",
        "StartupMemoryZSwapMax": "infinity",
        "StateChangeTimestamp": "Tue 2026-11-10 07:48:45 -05",
        "StateChangeTimestampMonotonic": "41460958441",
        "StateDirectoryMode": "0755",
        "StatusErrno": "0",
        "StopWhenUnneeded": "no",
        "SubState": "running",
        "SuccessAction": "none",
        "SurviveFinalKillSignal": "no",
        "SyslogFacility": "3",
        "SyslogLevel": "6",
        "SyslogLevelPrefix": "yes",
        "SyslogPriority": "30",
        "SystemCallErrorNumber": "2147483646",
        "TTYReset": "no",
        "TTYVHangup": "no",
        "TTYVTDisallocate": "no",
        "TasksAccounting": "yes",
        "TasksCurrent": "99",
        "TasksMax": "24779",
        "TimeoutAbortUSec": "1min 30s",
        "TimeoutCleanUSec": "infinity",
        "TimeoutStartFailureMode": "terminate",
        "TimeoutStartUSec": "1min 30s",
        "TimeoutStopFailureMode": "terminate",
        "TimeoutStopUSec": "1min 30s",
        "TimerSlackNSec": "50000",
        "Transient": "no",
        "Type": "simple",
        "UID": "992",
        "UMask": "0022",
        "UnitFilePreset": "disabled",
        "UnitFileState": "enabled",
        "User": "kafka",
        "UtmpMode": "init",
        "WantedBy": "multi-user.target",
        "WatchdogSignal": "6",
        "WatchdogTimestampMonotonic": "0",
        "WatchdogUSec": "0"
    }
}
[0 e@Gunther:~/Dropbox/Documents/Code/Kafka/kafka-lab-1]
▶︎
```

### Verify is down

```sh
[0 e@Gunther:~/Dropbox/Documents/Code/Kafka/kafka-lab-1]
▶︎ ansible kafka3 -b -m command -a "systemctl status kafka"
[ERROR]: Task failed: Module failed: non-zero return code
Origin: <adhoc 'command' task>

{'action': 'command', 'args': {'_raw_params': 'systemctl status kafka'}, 'timeout': 0, 'async_val': 0, 'poll': 15}

kafka3 | FAILED | rc=3 >>
× kafka.service - Apache Kafka Distributed Message Broker (KRaft Mode)
     Loaded: loaded (/etc/systemd/system/kafka.service; enabled; preset: disabled)
     Active: failed (Result: exit-code) since Tue 2026-11-10 10:54:28 -05; 1min 48s ago
   Duration: 3h 5min 40.828s
 Invocation: 7d856e9fee044c8d8ecfd08604cade29
       Docs: https://kafka.apache.org/documentation/
    Process: 266999 ExecStart=/opt/kafka/bin/kafka-server-start.sh /opt/kafka/config/kraft/server.properties (code=exited, status=143)
    Process: 271798 ExecStop=/opt/kafka/bin/kafka-server-stop.sh (code=exited, status=0/SUCCESS)
   Main PID: 266999 (code=exited, status=143)
   Mem peak: 611.2M
        CPU: 6min 19.903s

Nov 10 10:54:28 server3.lan kafka-server-start.sh[266999]: [2026-11-10 10:54:28,355] INFO [SnapshotGenerator id=3] closed event queue. (org.apache.kafka.queue.KafkaEventQueue)
Nov 10 10:54:28 server3.lan kafka-server-start.sh[266999]: [2026-11-10 10:54:28,355] INFO Metrics scheduler closed (org.apache.kafka.common.metrics.Metrics)
Nov 10 10:54:28 server3.lan kafka-server-start.sh[266999]: [2026-11-10 10:54:28,355] INFO Closing reporter org.apache.kafka.common.metrics.JmxReporter (org.apache.kafka.common.metrics.Metrics)
Nov 10 10:54:28 server3.lan kafka-server-start.sh[266999]: [2026-11-10 10:54:28,355] INFO Metrics reporters closed (org.apache.kafka.common.metrics.Metrics)
Nov 10 10:54:28 server3.lan kafka-server-start.sh[266999]: [2026-11-10 10:54:28,356] INFO App info kafka.server for 3 unregistered (org.apache.kafka.common.utils.AppInfoParser)
Nov 10 10:54:28 server3.lan kafka-server-start.sh[266999]: [2026-11-10 10:54:28,356] INFO App info kafka.server for 3 unregistered (org.apache.kafka.common.utils.AppInfoParser)
Nov 10 10:54:28 server3.lan systemd[1]: kafka.service: Main process exited, code=exited, status=143/n/a
Nov 10 10:54:28 server3.lan systemd[1]: kafka.service: Failed with result 'exit-code'.
Nov 10 10:54:28 server3.lan systemd[1]: Stopped kafka.service - Apache Kafka Distributed Message Broker (KRaft Mode).
Nov 10 10:54:28 server3.lan systemd[1]: kafka.service: Consumed 6min 19.903s CPU time, 611.2M memory peak.non-zero return code
[2 e@Gunther:~/Dropbox/Documents/Code/Kafka/kafka-lab-1]
▶︎
```

### Verify quorum health

```sh
[2 e@Gunther:~/Dropbox/Documents/Code/Kafka/kafka-lab-1]
▶︎ ansible kafka1 -m command -a "/opt/kafka/bin/kafka-metadata-quorum.sh --bootstrap-server 192.168.25.221:9092 describe --status"
kafka1 | CHANGED | rc=0 >>
ClusterId:              4L6g3nShT-eMCtK--X86sw
LeaderId:               1
LeaderEpoch:            1
HighWatermark:          23167
MaxFollowerLag:         282
MaxFollowerLagTimeMs:   141426
CurrentVoters:          [1,2,3]
CurrentObservers:       []
[0 e@Gunther:~/Dropbox/Documents/Code/Kafka/kafka-lab-1]
▶︎
[2 e@Gunther:~/Dropbox/Documents/Code/Kafka/kafka-lab-1]
▶︎ ansible kafka1 -m command -a "/opt/kafka/bin/kafka-metadata-quorum.sh --bootstrap-server 192.168.25.221:9092 describe --status"
kafka1 | CHANGED | rc=0 >>
ClusterId:              4L6g3nShT-eMCtK--X86sw
LeaderId:               1
LeaderEpoch:            1
HighWatermark:          23167
MaxFollowerLag:         282
MaxFollowerLagTimeMs:   141426
CurrentVoters:          [1,2,3]
CurrentObservers:       []
[0 e@Gunther:~/Dropbox/Documents/Code/Kafka/kafka-lab-1]
▶︎
```

### Start it again

```sh
[0 e@Gunther:~/Dropbox/Documents/Code/Kafka/kafka-lab-1]
▶︎ ansible kafka3 -b -m systemd -a "name=kafka state=started"
kafka3 | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3.12"
    },
    "changed": true,
    "name": "kafka",
    "state": "started",
    "status": {
        "AccessSELinuxContext": "system_u:object_r:systemd_unit_file_t:s0",
        "ActiveEnterTimestamp": "Tue 2026-11-10 07:48:45 -05",
        "ActiveEnterTimestampMonotonic": "41460958441",
        "ActiveExitTimestamp": "Tue 2026-11-10 10:54:25 -05",
        "ActiveExitTimestampMonotonic": "52601786658",
        "ActiveState": "failed",
        "After": "sysinit.target system.slice systemd-journald.socket network.target basic.target",
        "AllowIsolate": "no",
        "AssertResult": "yes",
        "AssertTimestamp": "Tue 2026-11-10 07:48:45 -05",
        "AssertTimestampMonotonic": "41460957108",
        "Before": "multi-user.target shutdown.target",
        "BindLogSockets": "no",
        "BlockIOAccounting": "no",
        "BlockIOWeight": "[not set]",
        "CPUAccounting": "yes",
        "CPUAffinityFromNUMA": "no",
        "CPUQuotaPerSecUSec": "infinity",
        "CPUQuotaPeriodUSec": "infinity",
        "CPUSchedulingPolicy": "0",
        "CPUSchedulingPriority": "0",
        "CPUSchedulingResetOnFork": "no",
        "CPUShares": "[not set]",
        "CPUUsageNSec": "379903817000",
        "CPUWeight": "[not set]",
        "CacheDirectoryMode": "0755",
        "CanFreeze": "yes",
        "CanIsolate": "no",
        "CanLiveMount": "no",
        "CanReload": "no",
        "CanStart": "yes",
        "CanStop": "yes",
        "CapabilityBoundingSet": "cap_chown cap_dac_override cap_dac_read_search cap_fowner cap_fsetid cap_kill cap_setgid cap_setuid cap_setpcap cap_linux_immutable cap_net_bind_service cap_net_broadcast cap_net_admin cap_net_raw cap_ipc_lock cap_ipc_owner cap_sys_module cap_sys_rawio cap_sys_chroot cap_sys_ptrace cap_sys_pacct cap_sys_admin cap_sys_boot cap_sys_nice cap_sys_resource cap_sys_time cap_sys_tty_config cap_mknod cap_lease cap_audit_write cap_audit_control cap_setfcap cap_mac_override cap_mac_admin cap_syslog cap_wake_alarm cap_block_suspend cap_audit_read cap_perfmon cap_bpf cap_checkpoint_restore",
        "CleanResult": "success",
        "CollectMode": "inactive",
        "ConditionResult": "yes",
        "ConditionTimestamp": "Tue 2026-11-10 07:48:45 -05",
        "ConditionTimestampMonotonic": "41460957104",
        "ConfigurationDirectoryMode": "0755",
        "Conflicts": "shutdown.target",
        "ControlGroupId": "36099",
        "ControlPID": "0",
        "CoredumpFilter": "0x33",
        "CoredumpReceive": "no",
        "DebugInvocation": "no",
        "DefaultDependencies": "yes",
        "DefaultMemoryLow": "0",
        "DefaultMemoryMin": "0",
        "DefaultStartupMemoryLow": "0",
        "Delegate": "no",
        "Description": "Apache Kafka Distributed Message Broker (KRaft Mode)",
        "DevicePolicy": "auto",
        "Documentation": "https://kafka.apache.org/documentation/",
        "DynamicUser": "no",
        "EffectiveMemoryHigh": "4087136256",
        "EffectiveMemoryMax": "4087136256",
        "EffectiveTasksMax": "24779",
        "Environment": "JAVA_HOME=/etc/alternatives/jre LOG_DIR=/var/log/kafka",
        "ExecMainCode": "1",
        "ExecMainExitTimestamp": "Tue 2026-11-10 10:54:28 -05",
        "ExecMainExitTimestampMonotonic": "52604629907",
        "ExecMainHandoffTimestamp": "Tue 2026-11-10 07:48:45 -05",
        "ExecMainHandoffTimestampMonotonic": "41461001271",
        "ExecMainPID": "266999",
        "ExecMainStartTimestamp": "Tue 2026-11-10 07:48:45 -05",
        "ExecMainStartTimestampMonotonic": "41460957882",
        "ExecMainStatus": "143",
        "ExecStart": "{ path=/opt/kafka/bin/kafka-server-start.sh ; argv[]=/opt/kafka/bin/kafka-server-start.sh /opt/kafka/config/kraft/server.properties ; ignore_errors=no ; start_time=[Tue 2026-11-10 07:48:45 -05] ; stop_time=[Tue 2026-11-10 10:54:28 -05] ; pid=266999 ; code=exited ; status=143 }",
        "ExecStartEx": "{ path=/opt/kafka/bin/kafka-server-start.sh ; argv[]=/opt/kafka/bin/kafka-server-start.sh /opt/kafka/config/kraft/server.properties ; flags= ; start_time=[Tue 2026-11-10 07:48:45 -05] ; stop_time=[Tue 2026-11-10 10:54:28 -05] ; pid=266999 ; code=exited ; status=143 }",
        "ExecStop": "{ path=/opt/kafka/bin/kafka-server-stop.sh ; argv[]=/opt/kafka/bin/kafka-server-stop.sh ; ignore_errors=no ; start_time=[Tue 2026-11-10 10:54:25 -05] ; stop_time=[Tue 2026-11-10 10:54:25 -05] ; pid=271798 ; code=exited ; status=0 }",
        "ExecStopEx": "{ path=/opt/kafka/bin/kafka-server-stop.sh ; argv[]=/opt/kafka/bin/kafka-server-stop.sh ; flags= ; start_time=[Tue 2026-11-10 10:54:25 -05] ; stop_time=[Tue 2026-11-10 10:54:25 -05] ; pid=271798 ; code=exited ; status=0 }",
        "ExitType": "main",
        "ExtensionImagePolicy": "root=verity+signed+encrypted+unprotected+absent:usr=verity+signed+encrypted+unprotected+absent:home=encrypted+unprotected+absent:srv=encrypted+unprotected+absent:tmp=encrypted+unprotected+absent:var=encrypted+unprotected+absent",
        "FailureAction": "none",
        "FileDescriptorStoreMax": "0",
        "FileDescriptorStorePreserve": "restart",
        "FinalKillSignal": "9",
        "FragmentPath": "/etc/systemd/system/kafka.service",
        "FreezerState": "running",
        "GID": "[not set]",
        "Group": "kafka",
        "GuessMainPID": "yes",
        "IOAccounting": "no",
        "IOReadBytes": "[not set]",
        "IOReadOperations": "[not set]",
        "IOSchedulingClass": "2",
        "IOSchedulingPriority": "4",
        "IOWeight": "[not set]",
        "IOWriteBytes": "[not set]",
        "IOWriteOperations": "[not set]",
        "IPAccounting": "no",
        "IPEgressBytes": "[no data]",
        "IPEgressPackets": "[no data]",
        "IPIngressBytes": "[no data]",
        "IPIngressPackets": "[no data]",
        "Id": "kafka.service",
        "IgnoreOnIsolate": "no",
        "IgnoreSIGPIPE": "yes",
        "InactiveEnterTimestamp": "Tue 2026-11-10 10:54:28 -05",
        "InactiveEnterTimestampMonotonic": "52604630630",
        "InactiveExitTimestamp": "Tue 2026-11-10 07:48:45 -05",
        "InactiveExitTimestampMonotonic": "41460958441",
        "InvocationID": "7d856e9fee044c8d8ecfd08604cade29",
        "JobRunningTimeoutUSec": "infinity",
        "JobTimeoutAction": "none",
        "JobTimeoutUSec": "infinity",
        "KeyringMode": "private",
        "KillMode": "control-group",
        "KillSignal": "15",
        "LimitAS": "infinity",
        "LimitASSoft": "infinity",
        "LimitCORE": "infinity",
        "LimitCORESoft": "infinity",
        "LimitCPU": "infinity",
        "LimitCPUSoft": "infinity",
        "LimitDATA": "infinity",
        "LimitDATASoft": "infinity",
        "LimitFSIZE": "infinity",
        "LimitFSIZESoft": "infinity",
        "LimitLOCKS": "infinity",
        "LimitLOCKSSoft": "infinity",
        "LimitMEMLOCK": "8388608",
        "LimitMEMLOCKSoft": "8388608",
        "LimitMSGQUEUE": "819200",
        "LimitMSGQUEUESoft": "819200",
        "LimitNICE": "0",
        "LimitNICESoft": "0",
        "LimitNOFILE": "65536",
        "LimitNOFILESoft": "65536",
        "LimitNPROC": "15487",
        "LimitNPROCSoft": "15487",
        "LimitRSS": "infinity",
        "LimitRSSSoft": "infinity",
        "LimitRTPRIO": "0",
        "LimitRTPRIOSoft": "0",
        "LimitRTTIME": "infinity",
        "LimitRTTIMESoft": "infinity",
        "LimitSIGPENDING": "15487",
        "LimitSIGPENDINGSoft": "15487",
        "LimitSTACK": "infinity",
        "LimitSTACKSoft": "8388608",
        "LiveMountResult": "success",
        "LoadState": "loaded",
        "LockPersonality": "no",
        "LogLevelMax": "-1",
        "LogRateLimitBurst": "0",
        "LogRateLimitIntervalUSec": "0",
        "LogsDirectoryMode": "0755",
        "MainPID": "0",
        "ManagedOOMMemoryPressure": "auto",
        "ManagedOOMMemoryPressureDurationUSec": "[not set]",
        "ManagedOOMMemoryPressureLimit": "0",
        "ManagedOOMPreference": "none",
        "ManagedOOMSwap": "auto",
        "MemoryAccounting": "yes",
        "MemoryAvailable": "3572301824",
        "MemoryCurrent": "[not set]",
        "MemoryDenyWriteExecute": "no",
        "MemoryHigh": "infinity",
        "MemoryKSM": "no",
        "MemoryLimit": "infinity",
        "MemoryLow": "0",
        "MemoryMax": "infinity",
        "MemoryMin": "0",
        "MemoryPeak": "640978944",
        "MemoryPressureThresholdUSec": "200ms",
        "MemoryPressureWatch": "auto",
        "MemorySwapCurrent": "[not set]",
        "MemorySwapMax": "infinity",
        "MemorySwapPeak": "0",
        "MemoryZSwapCurrent": "[not set]",
        "MemoryZSwapMax": "infinity",
        "MemoryZSwapWriteback": "yes",
        "MountAPIVFS": "no",
        "MountImagePolicy": "root=verity+signed+encrypted+unprotected+absent:usr=verity+signed+encrypted+unprotected+absent:home=encrypted+unprotected+absent:srv=encrypted+unprotected+absent:tmp=encrypted+unprotected+absent:var=encrypted+unprotected+absent",
        "NFileDescriptorStore": "0",
        "NRestarts": "0",
        "NUMAPolicy": "n/a",
        "Names": "kafka.service",
        "NeedDaemonReload": "no",
        "Nice": "0",
        "NoNewPrivileges": "no",
        "NonBlocking": "no",
        "NotifyAccess": "none",
        "OOMPolicy": "stop",
        "OOMScoreAdjust": "0",
        "OnFailureJobMode": "replace",
        "OnSuccessJobMode": "fail",
        "Perpetual": "no",
        "PrivateDevices": "no",
        "PrivateIPC": "no",
        "PrivateMounts": "no",
        "PrivateNetwork": "no",
        "PrivatePIDs": "no",
        "PrivateTmp": "no",
        "PrivateTmpEx": "no",
        "PrivateUsers": "no",
        "PrivateUsersEx": "no",
        "ProcSubset": "all",
        "ProtectClock": "no",
        "ProtectControlGroups": "no",
        "ProtectControlGroupsEx": "no",
        "ProtectHome": "no",
        "ProtectHostname": "no",
        "ProtectKernelLogs": "no",
        "ProtectKernelModules": "no",
        "ProtectKernelTunables": "no",
        "ProtectProc": "default",
        "ProtectSystem": "no",
        "RefuseManualStart": "no",
        "RefuseManualStop": "no",
        "ReloadResult": "success",
        "ReloadSignal": "1",
        "RemainAfterExit": "no",
        "RemoveIPC": "no",
        "Requires": "system.slice sysinit.target",
        "Restart": "on-failure",
        "RestartKillSignal": "15",
        "RestartMaxDelayUSec": "infinity",
        "RestartMode": "normal",
        "RestartSteps": "0",
        "RestartUSec": "10s",
        "RestartUSecNext": "10s",
        "RestrictNamespaces": "no",
        "RestrictRealtime": "no",
        "RestrictSUIDSGID": "no",
        "Result": "exit-code",
        "RootDirectoryStartOnly": "no",
        "RootEphemeral": "no",
        "RootImagePolicy": "root=verity+signed+encrypted+unprotected+absent:usr=verity+signed+encrypted+unprotected+absent:home=encrypted+unprotected+absent:srv=encrypted+unprotected+absent:tmp=encrypted+unprotected+absent:var=encrypted+unprotected+absent",
        "RuntimeDirectoryMode": "0755",
        "RuntimeDirectoryPreserve": "no",
        "RuntimeMaxUSec": "infinity",
        "RuntimeRandomizedExtraUSec": "0",
        "SameProcessGroup": "no",
        "SecureBits": "0",
        "SendSIGHUP": "no",
        "SendSIGKILL": "yes",
        "SetLoginEnvironment": "yes",
        "Slice": "system.slice",
        "StandardError": "inherit",
        "StandardInput": "null",
        "StandardOutput": "journal",
        "StartLimitAction": "none",
        "StartLimitBurst": "5",
        "StartLimitIntervalUSec": "10s",
        "StartupBlockIOWeight": "[not set]",
        "StartupCPUShares": "[not set]",
        "StartupCPUWeight": "[not set]",
        "StartupIOWeight": "[not set]",
        "StartupMemoryHigh": "infinity",
        "StartupMemoryLow": "0",
        "StartupMemoryMax": "infinity",
        "StartupMemorySwapMax": "infinity",
        "StartupMemoryZSwapMax": "infinity",
        "StateChangeTimestamp": "Tue 2026-11-10 10:54:28 -05",
        "StateChangeTimestampMonotonic": "52604630630",
        "StateDirectoryMode": "0755",
        "StatusErrno": "0",
        "StopWhenUnneeded": "no",
        "SubState": "failed",
        "SuccessAction": "none",
        "SurviveFinalKillSignal": "no",
        "SyslogFacility": "3",
        "SyslogLevel": "6",
        "SyslogLevelPrefix": "yes",
        "SyslogPriority": "30",
        "SystemCallErrorNumber": "2147483646",
        "TTYReset": "no",
        "TTYVHangup": "no",
        "TTYVTDisallocate": "no",
        "TasksAccounting": "yes",
        "TasksCurrent": "[not set]",
        "TasksMax": "24779",
        "TimeoutAbortUSec": "1min 30s",
        "TimeoutCleanUSec": "infinity",
        "TimeoutStartFailureMode": "terminate",
        "TimeoutStartUSec": "1min 30s",
        "TimeoutStopFailureMode": "terminate",
        "TimeoutStopUSec": "1min 30s",
        "TimerSlackNSec": "50000",
        "Transient": "no",
        "Type": "simple",
        "UID": "[not set]",
        "UMask": "0022",
        "UnitFilePreset": "disabled",
        "UnitFileState": "enabled",
        "User": "kafka",
        "UtmpMode": "init",
        "WantedBy": "multi-user.target",
        "WatchdogSignal": "6",
        "WatchdogTimestampMonotonic": "0",
        "WatchdogUSec": "0"
    }
}
[0 e@Gunther:~/Dropbox/Documents/Code/Kafka/kafka-lab-1]
▶︎
```

### Verify again

Cluster normalize and rebalance automatically.

```sh
[0 e@Gunther:~/Dropbox/Documents/Code/Kafka/kafka-lab-1]
▶︎ ansible kafka1 -m command -a "/opt/kafka/bin/kafka-topics.sh --bootstrap-server 192.168.25.221:9092 --describe --topic lab-telemetry"
kafka1 | CHANGED | rc=0 >>
Topic: lab-telemetry	TopicId: eHLvxdOHTdW6GZkpTgt91Q	PartitionCount: 3	ReplicationFactor: 3	Configs: min.insync.replicas=2
	Topic: lab-telemetry	Partition: 0	Leader: 2	Replicas: 2,3,1	Isr: 2,1,3
	Topic: lab-telemetry	Partition: 1	Leader: 1	Replicas: 3,1,2	Isr: 1,2,3
	Topic: lab-telemetry	Partition: 2	Leader: 1	Replicas: 1,2,3	Isr: 1,2,3
[0 e@Gunther:~/Dropbox/Documents/Code/Kafka/kafka-lab-1]
▶︎
```

# Backups

## Prepare backups

A NFS resource from `192.168.25.15` named `/Public` will be used and was prepared beforehand. The kafka nodes will need to use that space:

### Prepare playbook

Named `prepared-cluster-backup.yml`. It needs the script template on `templates/full-broker-backup.sh.j2`:

```yaml
---
- name: Prepare backup NFS space on nodes
  hosts: kafka
  become: true
  tasks:
    - name: Ensure NFS client is installed
      ansible.builtin.dnf:
        name: nfs-utils
        state: present
    - name: Ensure NFS mount directory is present
      ansible.builtin.file:
        state: directory
        mode: "a=rwx,g+s,+t"
        path: /mnt/Public
    - name: Ensure NFS mount is ready
      ansible.posix.mount:
        src: "192.168.25.15:/Public"
        path: /mnt/Public
        fstype: nfs
        opts: "soft,bg,intr,noatime"
        state: mounted
    - name: Ensure backup script is in /usr/local/bin
      ansible.builtin.template:
        src: full-broker-backup.sh.j2
        dest: /usr/local/bin/full-brocker-backup.sh
        mode: "u=rwx,go=rx"
```

### Backup script

And the needed script template `templates/full-broker-backup.sh.j2`:

```sh
#!/usr/bin/env bash
set -euo pipefail

# Node identity dynamically injected from Ansible
NODE_ID="{{ node_id | default(inventory_hostname) }}"
NODE_NAME="{{ inventory_hostname }}"

# Backup destination with fallback
DEF_DEST="${KAFKA_BACKUP_DEST:-/mnt/Public}"
BACKUP_DEST="${DEF_DEST}/${NODE_NAME}/$(date +%Y%m%d_%H%M%S)"
mkdir -p "${BACKUP_DEST}"

# Stop the Kafka service to flush page cache, checkpoint files, and log segments
echo "Stopping Kafka on ${NODE_NAME}..."
sudo systemctl stop kafka.service

# Archive log segments, metadata, and checkpoints (excluding file locks)
echo "Archiving storage directory..."
tar -czvf "${BACKUP_DEST}/kafka-broker-${NODE_NAME}-storage.tar.gz" \
    --exclude="lost+found" \
    --exclude="*.lock" \
    --exclude=".lock" \
    -C /var/lib/kafka data

# Resume Kafka broker operations
echo "Restarting Kafka on ${NODE_NAME}..."
sudo systemctl start kafka.service

logger -t KAFKA_BACKUP "Node ${NODE_NAME} (ID: ${NODE_ID}) storage successfully archived to ${BACKUP_DEST}"
echo "Backup completed successfully at ${BACKUP_DEST}"
```

### Preparation with the playbook

The preparation includes creating a mounting point on each kafka nodes, validating the required software packages are present (and adding them if not), mounting the NFS resource, and copying the backup script were useable.

```sh
[0 e@Gunther:~/Dropbox/Documents/Code/Kafka/kafka-lab-1]
▶︎ ansible-playbook prepare-cluster-backup.yml

PLAY [Prepare backup NFS space on nodes] *********************************************************

TASK [Gathering Facts] ***************************************************************************
ok: [kafka1]
ok: [kafka2]
ok: [kafka3]

TASK [Ensure NFS client is installed] ************************************************************
ok: [kafka3]
ok: [kafka2]
ok: [kafka1]

TASK [Ensure NFS mount directory is present] *****************************************************
ok: [kafka3]
ok: [kafka2]
ok: [kafka1]

TASK [Ensure NFS mount is ready] *****************************************************************
[WARNING]: Deprecation warnings can be disabled by setting `deprecation_warnings=False` in ansible.cfg.
[DEPRECATION WARNING]: Importing 'to_bytes' from 'ansible.module_utils._text' is deprecated. This feature will be removed from ansible-core version 2.24. Use ansible.module_utils.common.text.converters instead.
[DEPRECATION WARNING]: Importing 'to_native' from 'ansible.module_utils._text' is deprecated. This feature will be removed from ansible-core version 2.24. Use ansible.module_utils.common.text.converters instead.
[DEPRECATION WARNING]: Passing `warnings` to `exit_json` or `fail_json` is deprecated. This feature will be removed from ansible-core version 2.23. Use `AnsibleModule.warn` instead.
ok: [kafka3]
ok: [kafka1]
ok: [kafka2]

TASK [Ensure backup script is in /usr/local/bin] *************************************************
ok: [kafka3]
ok: [kafka1]
ok: [kafka2]

PLAY RECAP ***************************************************************************************
kafka1                     : ok=5    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
kafka2                     : ok=5    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
kafka3                     : ok=5    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

[0 e@Gunther:~/Dropbox/Documents/Code/Kafka/kafka-lab-1]
▶︎
```

## Run the backups

Use ansible to try to parallelize the backups:

```sh
[0 e@Gunther:~/Dropbox/Documents/Code/Kafka/kafka-lab-1]
▶︎ ansible kafka -b -m command -a "/usr/local/bin/full-brocker-backup.sh"
kafka3 | CHANGED | rc=0 >>
Stopping Kafka on kafka3...
Archiving storage directory...
data/
data/bootstrap.checkpoint
data/meta.properties
data/__cluster_metadata-0/
data/__cluster_metadata-0/leader-epoch-checkpoint
data/__cluster_metadata-0/00000000000000000000.log
data/__cluster_metadata-0/00000000000000000000.index
data/__cluster_metadata-0/00000000000000000000.timeindex
data/__cluster_metadata-0/partition.metadata
data/__cluster_metadata-0/quorum-state
data/__cluster_metadata-0/00000000000000007747-0000000001.checkpoint
data/__cluster_metadata-0/00000000000000014948-0000000001.checkpoint
data/__cluster_metadata-0/00000000000000022145-0000000001.checkpoint
data/__cluster_metadata-0/00000000000000027756.snapshot
data/__cluster_metadata-0/00000000000000028254.snapshot
data/recovery-point-offset-checkpoint
data/log-start-offset-checkpoint
data/replication-offset-checkpoint
data/cleaner-offset-checkpoint
data/lab-healthcheck-1/
data/lab-healthcheck-1/leader-epoch-checkpoint
data/lab-healthcheck-1/00000000000000000000.log
data/lab-healthcheck-1/00000000000000000000.index
data/lab-healthcheck-1/00000000000000000000.timeindex
data/lab-healthcheck-1/partition.metadata
data/lab-healthcheck-2/
data/lab-healthcheck-2/leader-epoch-checkpoint
data/lab-healthcheck-2/00000000000000000000.log
data/lab-healthcheck-2/00000000000000000000.index
data/lab-healthcheck-2/00000000000000000000.timeindex
data/lab-healthcheck-2/partition.metadata
data/lab-healthcheck-0/
data/lab-healthcheck-0/leader-epoch-checkpoint
data/lab-healthcheck-0/00000000000000000000.log
data/lab-healthcheck-0/00000000000000000000.index
data/lab-healthcheck-0/00000000000000000000.timeindex
data/lab-healthcheck-0/partition.metadata
data/lab-healthcheck-0/00000000000000000001.snapshot
data/__consumer_offsets-15/
data/__consumer_offsets-15/leader-epoch-checkpoint
data/__consumer_offsets-15/00000000000000000000.log
data/__consumer_offsets-15/00000000000000000000.index
data/__consumer_offsets-15/00000000000000000000.timeindex
data/__consumer_offsets-15/partition.metadata
data/__consumer_offsets-47/
data/__consumer_offsets-47/leader-epoch-checkpoint
data/__consumer_offsets-47/00000000000000000000.log
data/__consumer_offsets-47/00000000000000000000.index
data/__consumer_offsets-47/00000000000000000000.timeindex
data/__consumer_offsets-47/partition.metadata
data/__consumer_offsets-12/
data/__consumer_offsets-12/leader-epoch-checkpoint
data/__consumer_offsets-12/00000000000000000000.log
data/__consumer_offsets-12/00000000000000000000.index
data/__consumer_offsets-12/00000000000000000000.timeindex
data/__consumer_offsets-12/partition.metadata
data/__consumer_offsets-44/
data/__consumer_offsets-44/leader-epoch-checkpoint
data/__consumer_offsets-44/00000000000000000000.log
data/__consumer_offsets-44/00000000000000000000.index
data/__consumer_offsets-44/00000000000000000000.timeindex
data/__consumer_offsets-44/partition.metadata
data/__consumer_offsets-9/
data/__consumer_offsets-9/leader-epoch-checkpoint
data/__consumer_offsets-9/00000000000000000000.log
data/__consumer_offsets-9/00000000000000000000.index
data/__consumer_offsets-9/00000000000000000000.timeindex
data/__consumer_offsets-9/partition.metadata
data/__consumer_offsets-22/
data/__consumer_offsets-22/leader-epoch-checkpoint
data/__consumer_offsets-22/00000000000000000000.log
data/__consumer_offsets-22/00000000000000000000.index
data/__consumer_offsets-22/00000000000000000000.timeindex
data/__consumer_offsets-22/partition.metadata
data/__consumer_offsets-19/
data/__consumer_offsets-19/leader-epoch-checkpoint
data/__consumer_offsets-19/00000000000000000000.log
data/__consumer_offsets-19/00000000000000000000.index
data/__consumer_offsets-19/00000000000000000000.timeindex
data/__consumer_offsets-19/partition.metadata
data/__consumer_offsets-32/
data/__consumer_offsets-32/leader-epoch-checkpoint
data/__consumer_offsets-32/00000000000000000000.log
data/__consumer_offsets-32/00000000000000000000.index
data/__consumer_offsets-32/00000000000000000000.timeindex
data/__consumer_offsets-32/partition.metadata
data/__consumer_offsets-28/
data/__consumer_offsets-28/leader-epoch-checkpoint
data/__consumer_offsets-28/00000000000000000000.log
data/__consumer_offsets-28/00000000000000000000.index
data/__consumer_offsets-28/00000000000000000000.timeindex
data/__consumer_offsets-28/partition.metadata
data/__consumer_offsets-26/
data/__consumer_offsets-26/leader-epoch-checkpoint
data/__consumer_offsets-26/00000000000000000000.log
data/__consumer_offsets-26/00000000000000000000.index
data/__consumer_offsets-26/00000000000000000000.timeindex
data/__consumer_offsets-26/partition.metadata
data/__consumer_offsets-39/
data/__consumer_offsets-39/leader-epoch-checkpoint
data/__consumer_offsets-39/00000000000000000000.log
data/__consumer_offsets-39/00000000000000000000.index
data/__consumer_offsets-39/00000000000000000000.timeindex
data/__consumer_offsets-39/partition.metadata
data/__consumer_offsets-8/
data/__consumer_offsets-8/leader-epoch-checkpoint
data/__consumer_offsets-8/00000000000000000000.log
data/__consumer_offsets-8/00000000000000000000.index
data/__consumer_offsets-8/00000000000000000000.timeindex
data/__consumer_offsets-8/partition.metadata
data/__consumer_offsets-5/
data/__consumer_offsets-5/leader-epoch-checkpoint
data/__consumer_offsets-5/00000000000000000000.log
data/__consumer_offsets-5/00000000000000000000.index
data/__consumer_offsets-5/00000000000000000000.timeindex
data/__consumer_offsets-5/partition.metadata
data/__consumer_offsets-37/
data/__consumer_offsets-37/leader-epoch-checkpoint
data/__consumer_offsets-37/00000000000000000000.log
data/__consumer_offsets-37/00000000000000000000.index
data/__consumer_offsets-37/00000000000000000000.timeindex
data/__consumer_offsets-37/partition.metadata
data/__consumer_offsets-33/
data/__consumer_offsets-33/leader-epoch-checkpoint
data/__consumer_offsets-33/00000000000000000000.log
data/__consumer_offsets-33/00000000000000000000.index
data/__consumer_offsets-33/00000000000000000000.timeindex
data/__consumer_offsets-33/partition.metadata
data/__consumer_offsets-2/
data/__consumer_offsets-2/leader-epoch-checkpoint
data/__consumer_offsets-2/00000000000000000000.log
data/__consumer_offsets-2/00000000000000000000.index
data/__consumer_offsets-2/00000000000000000000.timeindex
data/__consumer_offsets-2/partition.metadata
data/__consumer_offsets-2/00000000000000000003.snapshot
data/__consumer_offsets-48/
data/__consumer_offsets-48/leader-epoch-checkpoint
data/__consumer_offsets-48/00000000000000000000.log
data/__consumer_offsets-48/00000000000000000000.index
data/__consumer_offsets-48/00000000000000000000.timeindex
data/__consumer_offsets-48/partition.metadata
data/__consumer_offsets-13/
data/__consumer_offsets-13/leader-epoch-checkpoint
data/__consumer_offsets-13/00000000000000000000.log
data/__consumer_offsets-13/00000000000000000000.index
data/__consumer_offsets-13/00000000000000000000.timeindex
data/__consumer_offsets-13/partition.metadata
data/__consumer_offsets-46/
data/__consumer_offsets-46/leader-epoch-checkpoint
data/__consumer_offsets-46/00000000000000000000.log
data/__consumer_offsets-46/00000000000000000000.index
data/__consumer_offsets-46/00000000000000000000.timeindex
data/__consumer_offsets-46/partition.metadata
data/__consumer_offsets-11/
data/__consumer_offsets-11/leader-epoch-checkpoint
data/__consumer_offsets-11/00000000000000000000.log
data/__consumer_offsets-11/00000000000000000000.index
data/__consumer_offsets-11/00000000000000000000.timeindex
data/__consumer_offsets-11/partition.metadata
data/__consumer_offsets-42/
data/__consumer_offsets-42/leader-epoch-checkpoint
data/__consumer_offsets-42/00000000000000000000.log
data/__consumer_offsets-42/00000000000000000000.index
data/__consumer_offsets-42/00000000000000000000.timeindex
data/__consumer_offsets-42/partition.metadata
data/__consumer_offsets-23/
data/__consumer_offsets-23/leader-epoch-checkpoint
data/__consumer_offsets-23/00000000000000000000.log
data/__consumer_offsets-23/00000000000000000000.index
data/__consumer_offsets-23/00000000000000000000.timeindex
data/__consumer_offsets-23/partition.metadata
data/__consumer_offsets-21/
data/__consumer_offsets-21/leader-epoch-checkpoint
data/__consumer_offsets-21/00000000000000000000.log
data/__consumer_offsets-21/00000000000000000000.index
data/__consumer_offsets-21/00000000000000000000.timeindex
data/__consumer_offsets-21/partition.metadata
data/__consumer_offsets-17/
data/__consumer_offsets-17/leader-epoch-checkpoint
data/__consumer_offsets-17/00000000000000000000.log
data/__consumer_offsets-17/00000000000000000000.index
data/__consumer_offsets-17/00000000000000000000.timeindex
data/__consumer_offsets-17/partition.metadata
data/__consumer_offsets-30/
data/__consumer_offsets-30/leader-epoch-checkpoint
data/__consumer_offsets-30/00000000000000000000.log
data/__consumer_offsets-30/00000000000000000000.index
data/__consumer_offsets-30/00000000000000000000.timeindex
data/__consumer_offsets-30/partition.metadata
data/__consumer_offsets-7/
data/__consumer_offsets-7/leader-epoch-checkpoint
data/__consumer_offsets-7/00000000000000000000.log
data/__consumer_offsets-7/00000000000000000000.index
data/__consumer_offsets-7/00000000000000000000.timeindex
data/__consumer_offsets-7/partition.metadata
data/__consumer_offsets-40/
data/__consumer_offsets-40/leader-epoch-checkpoint
data/__consumer_offsets-40/00000000000000000000.log
data/__consumer_offsets-40/00000000000000000000.index
data/__consumer_offsets-40/00000000000000000000.timeindex
data/__consumer_offsets-40/partition.metadata
data/__consumer_offsets-38/
data/__consumer_offsets-38/leader-epoch-checkpoint
data/__consumer_offsets-38/00000000000000000000.log
data/__consumer_offsets-38/00000000000000000000.index
data/__consumer_offsets-38/00000000000000000000.timeindex
data/__consumer_offsets-38/partition.metadata
data/__consumer_offsets-3/
data/__consumer_offsets-3/leader-epoch-checkpoint
data/__consumer_offsets-3/00000000000000000000.log
data/__consumer_offsets-3/00000000000000000000.index
data/__consumer_offsets-3/00000000000000000000.timeindex
data/__consumer_offsets-3/partition.metadata
data/__consumer_offsets-36/
data/__consumer_offsets-36/leader-epoch-checkpoint
data/__consumer_offsets-36/00000000000000000000.log
data/__consumer_offsets-36/00000000000000000000.index
data/__consumer_offsets-36/00000000000000000000.timeindex
data/__consumer_offsets-36/partition.metadata
data/__consumer_offsets-1/
data/__consumer_offsets-1/leader-epoch-checkpoint
data/__consumer_offsets-1/00000000000000000000.log
data/__consumer_offsets-1/00000000000000000000.index
data/__consumer_offsets-1/00000000000000000000.timeindex
data/__consumer_offsets-1/partition.metadata
data/__consumer_offsets-34/
data/__consumer_offsets-34/leader-epoch-checkpoint
data/__consumer_offsets-34/00000000000000000000.log
data/__consumer_offsets-34/00000000000000000000.index
data/__consumer_offsets-34/00000000000000000000.timeindex
data/__consumer_offsets-34/partition.metadata
data/__consumer_offsets-16/
data/__consumer_offsets-16/leader-epoch-checkpoint
data/__consumer_offsets-16/00000000000000000000.log
data/__consumer_offsets-16/00000000000000000000.index
data/__consumer_offsets-16/00000000000000000000.timeindex
data/__consumer_offsets-16/partition.metadata
data/__consumer_offsets-45/
data/__consumer_offsets-45/leader-epoch-checkpoint
data/__consumer_offsets-45/00000000000000000000.log
data/__consumer_offsets-45/00000000000000000000.index
data/__consumer_offsets-45/00000000000000000000.timeindex
data/__consumer_offsets-45/partition.metadata
data/__consumer_offsets-14/
data/__consumer_offsets-14/leader-epoch-checkpoint
data/__consumer_offsets-14/00000000000000000000.log
data/__consumer_offsets-14/00000000000000000000.index
data/__consumer_offsets-14/00000000000000000000.timeindex
data/__consumer_offsets-14/partition.metadata
data/__consumer_offsets-43/
data/__consumer_offsets-43/leader-epoch-checkpoint
data/__consumer_offsets-43/00000000000000000000.log
data/__consumer_offsets-43/00000000000000000000.index
data/__consumer_offsets-43/00000000000000000000.timeindex
data/__consumer_offsets-43/partition.metadata
data/__consumer_offsets-43/00000000000000001856.snapshot
data/__consumer_offsets-43/00000000000000001919.snapshot
data/__consumer_offsets-41/
data/__consumer_offsets-41/leader-epoch-checkpoint
data/__consumer_offsets-41/00000000000000000000.log
data/__consumer_offsets-41/00000000000000000000.index
data/__consumer_offsets-41/00000000000000000000.timeindex
data/__consumer_offsets-41/partition.metadata
data/__consumer_offsets-10/
data/__consumer_offsets-10/leader-epoch-checkpoint
data/__consumer_offsets-10/00000000000000000000.log
data/__consumer_offsets-10/00000000000000000000.index
data/__consumer_offsets-10/00000000000000000000.timeindex
data/__consumer_offsets-10/partition.metadata
data/__consumer_offsets-24/
data/__consumer_offsets-24/leader-epoch-checkpoint
data/__consumer_offsets-24/00000000000000000000.log
data/__consumer_offsets-24/00000000000000000000.index
data/__consumer_offsets-24/00000000000000000000.timeindex
data/__consumer_offsets-24/partition.metadata
data/__consumer_offsets-20/
data/__consumer_offsets-20/leader-epoch-checkpoint
data/__consumer_offsets-20/00000000000000000000.log
data/__consumer_offsets-20/00000000000000000000.index
data/__consumer_offsets-20/00000000000000000000.timeindex
data/__consumer_offsets-20/partition.metadata
data/__consumer_offsets-49/
data/__consumer_offsets-49/leader-epoch-checkpoint
data/__consumer_offsets-49/00000000000000000000.log
data/__consumer_offsets-49/00000000000000000000.index
data/__consumer_offsets-49/00000000000000000000.timeindex
data/__consumer_offsets-49/partition.metadata
data/__consumer_offsets-18/
data/__consumer_offsets-18/leader-epoch-checkpoint
data/__consumer_offsets-18/00000000000000000000.log
data/__consumer_offsets-18/00000000000000000000.index
data/__consumer_offsets-18/00000000000000000000.timeindex
data/__consumer_offsets-18/partition.metadata
data/__consumer_offsets-31/
data/__consumer_offsets-31/leader-epoch-checkpoint
data/__consumer_offsets-31/00000000000000000000.log
data/__consumer_offsets-31/00000000000000000000.index
data/__consumer_offsets-31/00000000000000000000.timeindex
data/__consumer_offsets-31/partition.metadata
data/__consumer_offsets-31/00000000000000000003.snapshot
data/__consumer_offsets-0/
data/__consumer_offsets-0/leader-epoch-checkpoint
data/__consumer_offsets-0/00000000000000000000.log
data/__consumer_offsets-0/00000000000000000000.index
data/__consumer_offsets-0/00000000000000000000.timeindex
data/__consumer_offsets-0/partition.metadata
data/__consumer_offsets-29/
data/__consumer_offsets-29/leader-epoch-checkpoint
data/__consumer_offsets-29/00000000000000000000.log
data/__consumer_offsets-29/00000000000000000000.index
data/__consumer_offsets-29/00000000000000000000.timeindex
data/__consumer_offsets-29/partition.metadata
data/__consumer_offsets-27/
data/__consumer_offsets-27/leader-epoch-checkpoint
data/__consumer_offsets-27/00000000000000000000.log
data/__consumer_offsets-27/00000000000000000000.index
data/__consumer_offsets-27/00000000000000000000.timeindex
data/__consumer_offsets-27/partition.metadata
data/__consumer_offsets-25/
data/__consumer_offsets-25/leader-epoch-checkpoint
data/__consumer_offsets-25/00000000000000000000.log
data/__consumer_offsets-25/00000000000000000000.index
data/__consumer_offsets-25/00000000000000000000.timeindex
data/__consumer_offsets-25/partition.metadata
data/__consumer_offsets-6/
data/__consumer_offsets-6/leader-epoch-checkpoint
data/__consumer_offsets-6/00000000000000000000.log
data/__consumer_offsets-6/00000000000000000000.index
data/__consumer_offsets-6/00000000000000000000.timeindex
data/__consumer_offsets-6/partition.metadata
data/__consumer_offsets-35/
data/__consumer_offsets-35/leader-epoch-checkpoint
data/__consumer_offsets-35/00000000000000000000.log
data/__consumer_offsets-35/00000000000000000000.index
data/__consumer_offsets-35/00000000000000000000.timeindex
data/__consumer_offsets-35/partition.metadata
data/__consumer_offsets-4/
data/__consumer_offsets-4/leader-epoch-checkpoint
data/__consumer_offsets-4/00000000000000000000.log
data/__consumer_offsets-4/00000000000000000000.index
data/__consumer_offsets-4/00000000000000000000.timeindex
data/__consumer_offsets-4/partition.metadata
data/lab-telemetry-1/
data/lab-telemetry-1/leader-epoch-checkpoint
data/lab-telemetry-1/00000000000000000000.log
data/lab-telemetry-1/00000000000000000000.index
data/lab-telemetry-1/00000000000000000000.timeindex
data/lab-telemetry-1/partition.metadata
data/lab-telemetry-1/00000000000000014472.snapshot
data/lab-telemetry-1/00000000000000014751.snapshot
data/lab-telemetry-0/
data/lab-telemetry-0/leader-epoch-checkpoint
data/lab-telemetry-0/00000000000000000000.log
data/lab-telemetry-0/00000000000000000000.index
data/lab-telemetry-0/00000000000000000000.timeindex
data/lab-telemetry-0/partition.metadata
data/lab-telemetry-0/00000000000000014559.snapshot
data/lab-telemetry-0/00000000000000014848.snapshot
data/lab-telemetry-2/
data/lab-telemetry-2/leader-epoch-checkpoint
data/lab-telemetry-2/00000000000000000000.log
data/lab-telemetry-2/00000000000000000000.index
data/lab-telemetry-2/00000000000000000000.timeindex
data/lab-telemetry-2/partition.metadata
data/lab-telemetry-2/00000000000000014606.snapshot
data/lab-telemetry-2/00000000000000014895.snapshot
data/.kafka_cleanshutdown
Restarting Kafka on kafka3...
Backup completed successfully at /mnt/Public/kafka3/20261110_113513
kafka1 | CHANGED | rc=0 >>
Stopping Kafka on kafka1...
Archiving storage directory...
data/
data/recovery-point-offset-checkpoint
data/__consumer_offsets-16/
data/__consumer_offsets-16/00000000000000000000.timeindex
data/__consumer_offsets-16/00000000000000000000.index
data/__consumer_offsets-16/leader-epoch-checkpoint
data/__consumer_offsets-16/partition.metadata
data/__consumer_offsets-16/00000000000000000000.log
data/__consumer_offsets-24/
data/__consumer_offsets-24/00000000000000000000.timeindex
data/__consumer_offsets-24/00000000000000000000.index
data/__consumer_offsets-24/leader-epoch-checkpoint
data/__consumer_offsets-24/partition.metadata
data/__consumer_offsets-24/00000000000000000000.log
data/__consumer_offsets-42/
data/__consumer_offsets-42/00000000000000000000.timeindex
data/__consumer_offsets-42/00000000000000000000.index
data/__consumer_offsets-42/leader-epoch-checkpoint
data/__consumer_offsets-42/partition.metadata
data/__consumer_offsets-42/00000000000000000000.log
data/__consumer_offsets-48/
data/__consumer_offsets-48/00000000000000000000.timeindex
data/__consumer_offsets-48/00000000000000000000.index
data/__consumer_offsets-48/leader-epoch-checkpoint
data/__consumer_offsets-48/partition.metadata
data/__consumer_offsets-48/00000000000000000000.log
data/lab-telemetry-0/
data/lab-telemetry-0/00000000000000000000.timeindex
data/lab-telemetry-0/00000000000000000000.index
data/lab-telemetry-0/leader-epoch-checkpoint
data/lab-telemetry-0/partition.metadata
data/lab-telemetry-0/00000000000000000000.log
data/lab-telemetry-0/00000000000000014848.snapshot
data/lab-telemetry-0/00000000000000014559.snapshot
data/__consumer_offsets-29/
data/__consumer_offsets-29/00000000000000000000.timeindex
data/__consumer_offsets-29/00000000000000000000.index
data/__consumer_offsets-29/leader-epoch-checkpoint
data/__consumer_offsets-29/partition.metadata
data/__consumer_offsets-29/00000000000000000000.log
data/__consumer_offsets-23/
data/__consumer_offsets-23/00000000000000000000.timeindex
data/__consumer_offsets-23/00000000000000000000.index
data/__consumer_offsets-23/leader-epoch-checkpoint
data/__consumer_offsets-23/partition.metadata
data/__consumer_offsets-23/00000000000000000000.log
data/lab-healthcheck-0/
data/lab-healthcheck-0/00000000000000000000.timeindex
data/lab-healthcheck-0/00000000000000000000.index
data/lab-healthcheck-0/leader-epoch-checkpoint
data/lab-healthcheck-0/partition.metadata
data/lab-healthcheck-0/00000000000000000000.log
data/lab-healthcheck-0/00000000000000000001.snapshot
data/bootstrap.checkpoint
data/__consumer_offsets-36/
data/__consumer_offsets-36/00000000000000000000.timeindex
data/__consumer_offsets-36/00000000000000000000.index
data/__consumer_offsets-36/leader-epoch-checkpoint
data/__consumer_offsets-36/partition.metadata
data/__consumer_offsets-36/00000000000000000000.log
data/lab-healthcheck-2/
data/lab-healthcheck-2/00000000000000000000.timeindex
data/lab-healthcheck-2/00000000000000000000.index
data/lab-healthcheck-2/leader-epoch-checkpoint
data/lab-healthcheck-2/partition.metadata
data/lab-healthcheck-2/00000000000000000000.log
data/__consumer_offsets-35/
data/__consumer_offsets-35/00000000000000000000.timeindex
data/__consumer_offsets-35/00000000000000000000.index
data/__consumer_offsets-35/leader-epoch-checkpoint
data/__consumer_offsets-35/partition.metadata
data/__consumer_offsets-35/00000000000000000000.log
data/__consumer_offsets-32/
data/__consumer_offsets-32/00000000000000000000.timeindex
data/__consumer_offsets-32/00000000000000000000.index
data/__consumer_offsets-32/leader-epoch-checkpoint
data/__consumer_offsets-32/partition.metadata
data/__consumer_offsets-32/00000000000000000000.log
data/__consumer_offsets-5/
data/__consumer_offsets-5/00000000000000000000.timeindex
data/__consumer_offsets-5/00000000000000000000.index
data/__consumer_offsets-5/leader-epoch-checkpoint
data/__consumer_offsets-5/partition.metadata
data/__consumer_offsets-5/00000000000000000000.log
data/__consumer_offsets-2/
data/__consumer_offsets-2/00000000000000000003.snapshot
data/__consumer_offsets-2/00000000000000000000.timeindex
data/__consumer_offsets-2/00000000000000000000.index
data/__consumer_offsets-2/leader-epoch-checkpoint
data/__consumer_offsets-2/partition.metadata
data/__consumer_offsets-2/00000000000000000000.log
data/.kafka_cleanshutdown
data/__consumer_offsets-18/
data/__consumer_offsets-18/00000000000000000000.timeindex
data/__consumer_offsets-18/00000000000000000000.index
data/__consumer_offsets-18/leader-epoch-checkpoint
data/__consumer_offsets-18/partition.metadata
data/__consumer_offsets-18/00000000000000000000.log
data/lab-healthcheck-1/
data/lab-healthcheck-1/00000000000000000000.timeindex
data/lab-healthcheck-1/00000000000000000000.index
data/lab-healthcheck-1/leader-epoch-checkpoint
data/lab-healthcheck-1/partition.metadata
data/lab-healthcheck-1/00000000000000000000.log
data/__consumer_offsets-7/
data/__consumer_offsets-7/00000000000000000000.timeindex
data/__consumer_offsets-7/00000000000000000000.index
data/__consumer_offsets-7/leader-epoch-checkpoint
data/__consumer_offsets-7/partition.metadata
data/__consumer_offsets-7/00000000000000000000.log
data/__consumer_offsets-30/
data/__consumer_offsets-30/00000000000000000000.timeindex
data/__consumer_offsets-30/00000000000000000000.index
data/__consumer_offsets-30/leader-epoch-checkpoint
data/__consumer_offsets-30/partition.metadata
data/__consumer_offsets-30/00000000000000000000.log
data/__consumer_offsets-43/
data/__consumer_offsets-43/00000000000000000000.timeindex
data/__consumer_offsets-43/00000000000000000000.index
data/__consumer_offsets-43/leader-epoch-checkpoint
data/__consumer_offsets-43/partition.metadata
data/__consumer_offsets-43/00000000000000001919.snapshot
data/__consumer_offsets-43/00000000000000001856.snapshot
data/__consumer_offsets-43/00000000000000000000.log
data/__consumer_offsets-10/
data/__consumer_offsets-10/00000000000000000000.timeindex
data/__consumer_offsets-10/00000000000000000000.index
data/__consumer_offsets-10/leader-epoch-checkpoint
data/__consumer_offsets-10/partition.metadata
data/__consumer_offsets-10/00000000000000000000.log
data/__consumer_offsets-15/
data/__consumer_offsets-15/00000000000000000000.timeindex
data/__consumer_offsets-15/00000000000000000000.index
data/__consumer_offsets-15/leader-epoch-checkpoint
data/__consumer_offsets-15/partition.metadata
data/__consumer_offsets-15/00000000000000000000.log
data/__consumer_offsets-27/
data/__consumer_offsets-27/00000000000000000000.timeindex
data/__consumer_offsets-27/00000000000000000000.index
data/__consumer_offsets-27/leader-epoch-checkpoint
data/__consumer_offsets-27/partition.metadata
data/__consumer_offsets-27/00000000000000000000.log
data/__consumer_offsets-25/
data/__consumer_offsets-25/00000000000000000000.timeindex
data/__consumer_offsets-25/00000000000000000000.index
data/__consumer_offsets-25/leader-epoch-checkpoint
data/__consumer_offsets-25/partition.metadata
data/__consumer_offsets-25/00000000000000000000.log
data/__consumer_offsets-17/
data/__consumer_offsets-17/00000000000000000000.timeindex
data/__consumer_offsets-17/00000000000000000000.index
data/__consumer_offsets-17/leader-epoch-checkpoint
data/__consumer_offsets-17/partition.metadata
data/__consumer_offsets-17/00000000000000000000.log
data/meta.properties
data/__consumer_offsets-45/
data/__consumer_offsets-45/00000000000000000000.timeindex
data/__consumer_offsets-45/00000000000000000000.index
data/__consumer_offsets-45/leader-epoch-checkpoint
data/__consumer_offsets-45/partition.metadata
data/__consumer_offsets-45/00000000000000000000.log
data/__consumer_offsets-34/
data/__consumer_offsets-34/00000000000000000000.timeindex
data/__consumer_offsets-34/00000000000000000000.index
data/__consumer_offsets-34/leader-epoch-checkpoint
data/__consumer_offsets-34/partition.metadata
data/__consumer_offsets-34/00000000000000000000.log
data/__consumer_offsets-0/
data/__consumer_offsets-0/00000000000000000000.timeindex
data/__consumer_offsets-0/00000000000000000000.index
data/__consumer_offsets-0/leader-epoch-checkpoint
data/__consumer_offsets-0/partition.metadata
data/__consumer_offsets-0/00000000000000000000.log
data/lab-telemetry-2/
data/lab-telemetry-2/00000000000000014895.snapshot
data/lab-telemetry-2/00000000000000000000.timeindex
data/lab-telemetry-2/00000000000000000000.index
data/lab-telemetry-2/leader-epoch-checkpoint
data/lab-telemetry-2/partition.metadata
data/lab-telemetry-2/00000000000000014606.snapshot
data/lab-telemetry-2/00000000000000000000.log
data/__consumer_offsets-49/
data/__consumer_offsets-49/00000000000000000000.timeindex
data/__consumer_offsets-49/00000000000000000000.index
data/__consumer_offsets-49/leader-epoch-checkpoint
data/__consumer_offsets-49/partition.metadata
data/__consumer_offsets-49/00000000000000000000.log
data/__consumer_offsets-1/
data/__consumer_offsets-1/00000000000000000000.timeindex
data/__consumer_offsets-1/00000000000000000000.index
data/__consumer_offsets-1/leader-epoch-checkpoint
data/__consumer_offsets-1/partition.metadata
data/__consumer_offsets-1/00000000000000000000.log
data/__consumer_offsets-9/
data/__consumer_offsets-9/00000000000000000000.timeindex
data/__consumer_offsets-9/00000000000000000000.index
data/__consumer_offsets-9/leader-epoch-checkpoint
data/__consumer_offsets-9/partition.metadata
data/__consumer_offsets-9/00000000000000000000.log
data/__consumer_offsets-44/
data/__consumer_offsets-44/00000000000000000000.timeindex
data/__consumer_offsets-44/00000000000000000000.index
data/__consumer_offsets-44/leader-epoch-checkpoint
data/__consumer_offsets-44/partition.metadata
data/__consumer_offsets-44/00000000000000000000.log
data/__cluster_metadata-0/
data/__cluster_metadata-0/00000000000000014477-0000000001.checkpoint
data/__cluster_metadata-0/00000000000000028257.snapshot
data/__cluster_metadata-0/00000000000000000000.timeindex
data/__cluster_metadata-0/00000000000000000000.index
data/__cluster_metadata-0/leader-epoch-checkpoint
data/__cluster_metadata-0/partition.metadata
data/__cluster_metadata-0/00000000000000000000.log
data/__cluster_metadata-0/00000000000000027756.snapshot
data/__cluster_metadata-0/quorum-state
data/__cluster_metadata-0/00000000000000007276-0000000001.checkpoint
data/__cluster_metadata-0/00000000000000021674-0000000001.checkpoint
data/__consumer_offsets-41/
data/__consumer_offsets-41/00000000000000000000.timeindex
data/__consumer_offsets-41/00000000000000000000.index
data/__consumer_offsets-41/leader-epoch-checkpoint
data/__consumer_offsets-41/partition.metadata
data/__consumer_offsets-41/00000000000000000000.log
data/__consumer_offsets-26/
data/__consumer_offsets-26/00000000000000000000.timeindex
data/__consumer_offsets-26/00000000000000000000.index
data/__consumer_offsets-26/leader-epoch-checkpoint
data/__consumer_offsets-26/partition.metadata
data/__consumer_offsets-26/00000000000000000000.log
data/log-start-offset-checkpoint
data/__consumer_offsets-11/
data/__consumer_offsets-11/00000000000000000000.timeindex
data/__consumer_offsets-11/00000000000000000000.index
data/__consumer_offsets-11/leader-epoch-checkpoint
data/__consumer_offsets-11/partition.metadata
data/__consumer_offsets-11/00000000000000000000.log
data/__consumer_offsets-13/
data/__consumer_offsets-13/00000000000000000000.timeindex
data/__consumer_offsets-13/00000000000000000000.index
data/__consumer_offsets-13/leader-epoch-checkpoint
data/__consumer_offsets-13/partition.metadata
data/__consumer_offsets-13/00000000000000000000.log
data/cleaner-offset-checkpoint
data/__consumer_offsets-39/
data/__consumer_offsets-39/00000000000000000000.timeindex
data/__consumer_offsets-39/00000000000000000000.index
data/__consumer_offsets-39/leader-epoch-checkpoint
data/__consumer_offsets-39/partition.metadata
data/__consumer_offsets-39/00000000000000000000.log
data/__consumer_offsets-3/
data/__consumer_offsets-3/00000000000000000000.timeindex
data/__consumer_offsets-3/00000000000000000000.index
data/__consumer_offsets-3/leader-epoch-checkpoint
data/__consumer_offsets-3/partition.metadata
data/__consumer_offsets-3/00000000000000000000.log
data/__consumer_offsets-8/
data/__consumer_offsets-8/00000000000000000000.timeindex
data/__consumer_offsets-8/00000000000000000000.index
data/__consumer_offsets-8/leader-epoch-checkpoint
data/__consumer_offsets-8/partition.metadata
data/__consumer_offsets-8/00000000000000000000.log
data/__consumer_offsets-28/
data/__consumer_offsets-28/00000000000000000000.timeindex
data/__consumer_offsets-28/00000000000000000000.index
data/__consumer_offsets-28/leader-epoch-checkpoint
data/__consumer_offsets-28/partition.metadata
data/__consumer_offsets-28/00000000000000000000.log
data/__consumer_offsets-20/
data/__consumer_offsets-20/00000000000000000000.timeindex
data/__consumer_offsets-20/00000000000000000000.index
data/__consumer_offsets-20/leader-epoch-checkpoint
data/__consumer_offsets-20/partition.metadata
data/__consumer_offsets-20/00000000000000000000.log
data/__consumer_offsets-21/
data/__consumer_offsets-21/00000000000000000000.timeindex
data/__consumer_offsets-21/00000000000000000000.index
data/__consumer_offsets-21/leader-epoch-checkpoint
data/__consumer_offsets-21/partition.metadata
data/__consumer_offsets-21/00000000000000000000.log
data/__consumer_offsets-31/
data/__consumer_offsets-31/00000000000000000003.snapshot
data/__consumer_offsets-31/00000000000000000000.timeindex
data/__consumer_offsets-31/00000000000000000000.index
data/__consumer_offsets-31/leader-epoch-checkpoint
data/__consumer_offsets-31/partition.metadata
data/__consumer_offsets-31/00000000000000000000.log
data/__consumer_offsets-33/
data/__consumer_offsets-33/00000000000000000000.timeindex
data/__consumer_offsets-33/00000000000000000000.index
data/__consumer_offsets-33/leader-epoch-checkpoint
data/__consumer_offsets-33/partition.metadata
data/__consumer_offsets-33/00000000000000000000.log
data/__consumer_offsets-22/
data/__consumer_offsets-22/00000000000000000000.timeindex
data/__consumer_offsets-22/00000000000000000000.index
data/__consumer_offsets-22/leader-epoch-checkpoint
data/__consumer_offsets-22/partition.metadata
data/__consumer_offsets-22/00000000000000000000.log
data/__consumer_offsets-37/
data/__consumer_offsets-37/00000000000000000000.timeindex
data/__consumer_offsets-37/00000000000000000000.index
data/__consumer_offsets-37/leader-epoch-checkpoint
data/__consumer_offsets-37/partition.metadata
data/__consumer_offsets-37/00000000000000000000.log
data/replication-offset-checkpoint
data/lab-telemetry-1/
data/lab-telemetry-1/00000000000000000000.timeindex
data/lab-telemetry-1/00000000000000000000.index
data/lab-telemetry-1/leader-epoch-checkpoint
data/lab-telemetry-1/partition.metadata
data/lab-telemetry-1/00000000000000000000.log
data/lab-telemetry-1/00000000000000014471.snapshot
data/lab-telemetry-1/00000000000000014751.snapshot
data/__consumer_offsets-19/
data/__consumer_offsets-19/00000000000000000000.timeindex
data/__consumer_offsets-19/00000000000000000000.index
data/__consumer_offsets-19/leader-epoch-checkpoint
data/__consumer_offsets-19/partition.metadata
data/__consumer_offsets-19/00000000000000000000.log
data/__consumer_offsets-47/
data/__consumer_offsets-47/00000000000000000000.timeindex
data/__consumer_offsets-47/00000000000000000000.index
data/__consumer_offsets-47/leader-epoch-checkpoint
data/__consumer_offsets-47/partition.metadata
data/__consumer_offsets-47/00000000000000000000.log
data/__consumer_offsets-46/
data/__consumer_offsets-46/00000000000000000000.timeindex
data/__consumer_offsets-46/00000000000000000000.index
data/__consumer_offsets-46/leader-epoch-checkpoint
data/__consumer_offsets-46/partition.metadata
data/__consumer_offsets-46/00000000000000000000.log
data/__consumer_offsets-40/
data/__consumer_offsets-40/00000000000000000000.timeindex
data/__consumer_offsets-40/00000000000000000000.index
data/__consumer_offsets-40/leader-epoch-checkpoint
data/__consumer_offsets-40/partition.metadata
data/__consumer_offsets-40/00000000000000000000.log
data/__consumer_offsets-14/
data/__consumer_offsets-14/00000000000000000000.timeindex
data/__consumer_offsets-14/00000000000000000000.index
data/__consumer_offsets-14/leader-epoch-checkpoint
data/__consumer_offsets-14/partition.metadata
data/__consumer_offsets-14/00000000000000000000.log
data/__consumer_offsets-4/
data/__consumer_offsets-4/00000000000000000000.timeindex
data/__consumer_offsets-4/00000000000000000000.index
data/__consumer_offsets-4/leader-epoch-checkpoint
data/__consumer_offsets-4/partition.metadata
data/__consumer_offsets-4/00000000000000000000.log
data/__consumer_offsets-12/
data/__consumer_offsets-12/00000000000000000000.timeindex
data/__consumer_offsets-12/00000000000000000000.index
data/__consumer_offsets-12/leader-epoch-checkpoint
data/__consumer_offsets-12/partition.metadata
data/__consumer_offsets-12/00000000000000000000.log
data/__consumer_offsets-6/
data/__consumer_offsets-6/00000000000000000000.timeindex
data/__consumer_offsets-6/00000000000000000000.index
data/__consumer_offsets-6/leader-epoch-checkpoint
data/__consumer_offsets-6/partition.metadata
data/__consumer_offsets-6/00000000000000000000.log
data/__consumer_offsets-38/
data/__consumer_offsets-38/00000000000000000000.timeindex
data/__consumer_offsets-38/00000000000000000000.index
data/__consumer_offsets-38/leader-epoch-checkpoint
data/__consumer_offsets-38/partition.metadata
data/__consumer_offsets-38/00000000000000000000.log
Restarting Kafka on kafka1...
Backup completed successfully at /mnt/Public/kafka1/20261110_113513
kafka2 | CHANGED | rc=0 >>
Stopping Kafka on kafka2...
Archiving storage directory...
data/
data/lab-healthcheck-1/
data/lab-healthcheck-1/partition.metadata
data/lab-healthcheck-1/leader-epoch-checkpoint
data/lab-healthcheck-1/00000000000000000000.index
data/lab-healthcheck-1/00000000000000000000.timeindex
data/lab-healthcheck-1/00000000000000000000.log
data/__consumer_offsets-0/
data/__consumer_offsets-0/partition.metadata
data/__consumer_offsets-0/leader-epoch-checkpoint
data/__consumer_offsets-0/00000000000000000000.index
data/__consumer_offsets-0/00000000000000000000.timeindex
data/__consumer_offsets-0/00000000000000000000.log
data/cleaner-offset-checkpoint
data/__consumer_offsets-16/
data/__consumer_offsets-16/partition.metadata
data/__consumer_offsets-16/leader-epoch-checkpoint
data/__consumer_offsets-16/00000000000000000000.index
data/__consumer_offsets-16/00000000000000000000.timeindex
data/__consumer_offsets-16/00000000000000000000.log
data/lab-healthcheck-2/
data/lab-healthcheck-2/partition.metadata
data/lab-healthcheck-2/leader-epoch-checkpoint
data/lab-healthcheck-2/00000000000000000000.index
data/lab-healthcheck-2/00000000000000000000.timeindex
data/lab-healthcheck-2/00000000000000000000.log
data/__consumer_offsets-38/
data/__consumer_offsets-38/partition.metadata
data/__consumer_offsets-38/leader-epoch-checkpoint
data/__consumer_offsets-38/00000000000000000000.index
data/__consumer_offsets-38/00000000000000000000.timeindex
data/__consumer_offsets-38/00000000000000000000.log
data/__consumer_offsets-8/
data/__consumer_offsets-8/partition.metadata
data/__consumer_offsets-8/leader-epoch-checkpoint
data/__consumer_offsets-8/00000000000000000000.index
data/__consumer_offsets-8/00000000000000000000.timeindex
data/__consumer_offsets-8/00000000000000000000.log
data/__consumer_offsets-4/
data/__consumer_offsets-4/partition.metadata
data/__consumer_offsets-4/leader-epoch-checkpoint
data/__consumer_offsets-4/00000000000000000000.index
data/__consumer_offsets-4/00000000000000000000.timeindex
data/__consumer_offsets-4/00000000000000000000.log
data/__consumer_offsets-25/
data/__consumer_offsets-25/partition.metadata
data/__consumer_offsets-25/leader-epoch-checkpoint
data/__consumer_offsets-25/00000000000000000000.index
data/__consumer_offsets-25/00000000000000000000.timeindex
data/__consumer_offsets-25/00000000000000000000.log
data/__consumer_offsets-48/
data/__consumer_offsets-48/partition.metadata
data/__consumer_offsets-48/leader-epoch-checkpoint
data/__consumer_offsets-48/00000000000000000000.index
data/__consumer_offsets-48/00000000000000000000.timeindex
data/__consumer_offsets-48/00000000000000000000.log
data/__consumer_offsets-31/
data/__consumer_offsets-31/00000000000000000003.snapshot
data/__consumer_offsets-31/partition.metadata
data/__consumer_offsets-31/leader-epoch-checkpoint
data/__consumer_offsets-31/00000000000000000000.index
data/__consumer_offsets-31/00000000000000000000.timeindex
data/__consumer_offsets-31/00000000000000000000.log
data/__consumer_offsets-49/
data/__consumer_offsets-49/partition.metadata
data/__consumer_offsets-49/leader-epoch-checkpoint
data/__consumer_offsets-49/00000000000000000000.index
data/__consumer_offsets-49/00000000000000000000.timeindex
data/__consumer_offsets-49/00000000000000000000.log
data/__consumer_offsets-6/
data/__consumer_offsets-6/partition.metadata
data/__consumer_offsets-6/leader-epoch-checkpoint
data/__consumer_offsets-6/00000000000000000000.index
data/__consumer_offsets-6/00000000000000000000.timeindex
data/__consumer_offsets-6/00000000000000000000.log
data/__consumer_offsets-28/
data/__consumer_offsets-28/partition.metadata
data/__consumer_offsets-28/leader-epoch-checkpoint
data/__consumer_offsets-28/00000000000000000000.index
data/__consumer_offsets-28/00000000000000000000.timeindex
data/__consumer_offsets-28/00000000000000000000.log
data/__consumer_offsets-3/
data/__consumer_offsets-3/partition.metadata
data/__consumer_offsets-3/leader-epoch-checkpoint
data/__consumer_offsets-3/00000000000000000000.index
data/__consumer_offsets-3/00000000000000000000.timeindex
data/__consumer_offsets-3/00000000000000000000.log
data/__consumer_offsets-36/
data/__consumer_offsets-36/partition.metadata
data/__consumer_offsets-36/leader-epoch-checkpoint
data/__consumer_offsets-36/00000000000000000000.index
data/__consumer_offsets-36/00000000000000000000.timeindex
data/__consumer_offsets-36/00000000000000000000.log
data/__consumer_offsets-35/
data/__consumer_offsets-35/partition.metadata
data/__consumer_offsets-35/leader-epoch-checkpoint
data/__consumer_offsets-35/00000000000000000000.index
data/__consumer_offsets-35/00000000000000000000.timeindex
data/__consumer_offsets-35/00000000000000000000.log
data/__consumer_offsets-21/
data/__consumer_offsets-21/partition.metadata
data/__consumer_offsets-21/leader-epoch-checkpoint
data/__consumer_offsets-21/00000000000000000000.index
data/__consumer_offsets-21/00000000000000000000.timeindex
data/__consumer_offsets-21/00000000000000000000.log
data/lab-telemetry-1/
data/lab-telemetry-1/00000000000000014751.snapshot
data/lab-telemetry-1/partition.metadata
data/lab-telemetry-1/leader-epoch-checkpoint
data/lab-telemetry-1/00000000000000000000.index
data/lab-telemetry-1/00000000000000000000.timeindex
data/lab-telemetry-1/00000000000000008363.snapshot
data/lab-telemetry-1/00000000000000000000.log
data/__consumer_offsets-1/
data/__consumer_offsets-1/partition.metadata
data/__consumer_offsets-1/leader-epoch-checkpoint
data/__consumer_offsets-1/00000000000000000000.index
data/__consumer_offsets-1/00000000000000000000.timeindex
data/__consumer_offsets-1/00000000000000000000.log
data/lab-telemetry-2/
data/lab-telemetry-2/00000000000000014895.snapshot
data/lab-telemetry-2/partition.metadata
data/lab-telemetry-2/leader-epoch-checkpoint
data/lab-telemetry-2/00000000000000000000.index
data/lab-telemetry-2/00000000000000000000.timeindex
data/lab-telemetry-2/00000000000000000000.log
data/__consumer_offsets-19/
data/__consumer_offsets-19/partition.metadata
data/__consumer_offsets-19/leader-epoch-checkpoint
data/__consumer_offsets-19/00000000000000000000.index
data/__consumer_offsets-19/00000000000000000000.timeindex
data/__consumer_offsets-19/00000000000000000000.log
data/__consumer_offsets-9/
data/__consumer_offsets-9/partition.metadata
data/__consumer_offsets-9/leader-epoch-checkpoint
data/__consumer_offsets-9/00000000000000000000.index
data/__consumer_offsets-9/00000000000000000000.timeindex
data/__consumer_offsets-9/00000000000000000000.log
data/__consumer_offsets-29/
data/__consumer_offsets-29/partition.metadata
data/__consumer_offsets-29/leader-epoch-checkpoint
data/__consumer_offsets-29/00000000000000000000.index
data/__consumer_offsets-29/00000000000000000000.timeindex
data/__consumer_offsets-29/00000000000000000000.log
data/__consumer_offsets-37/
data/__consumer_offsets-37/partition.metadata
data/__consumer_offsets-37/leader-epoch-checkpoint
data/__consumer_offsets-37/00000000000000000000.index
data/__consumer_offsets-37/00000000000000000000.timeindex
data/__consumer_offsets-37/00000000000000000000.log
data/meta.properties
data/__consumer_offsets-41/
data/__consumer_offsets-41/partition.metadata
data/__consumer_offsets-41/leader-epoch-checkpoint
data/__consumer_offsets-41/00000000000000000000.index
data/__consumer_offsets-41/00000000000000000000.timeindex
data/__consumer_offsets-41/00000000000000000000.log
data/__consumer_offsets-22/
data/__consumer_offsets-22/partition.metadata
data/__consumer_offsets-22/leader-epoch-checkpoint
data/__consumer_offsets-22/00000000000000000000.index
data/__consumer_offsets-22/00000000000000000000.timeindex
data/__consumer_offsets-22/00000000000000000000.log
data/log-start-offset-checkpoint
data/__consumer_offsets-24/
data/__consumer_offsets-24/partition.metadata
data/__consumer_offsets-24/leader-epoch-checkpoint
data/__consumer_offsets-24/00000000000000000000.index
data/__consumer_offsets-24/00000000000000000000.timeindex
data/__consumer_offsets-24/00000000000000000000.log
data/lab-healthcheck-0/
data/lab-healthcheck-0/00000000000000000001.snapshot
data/lab-healthcheck-0/partition.metadata
data/lab-healthcheck-0/leader-epoch-checkpoint
data/lab-healthcheck-0/00000000000000000000.index
data/lab-healthcheck-0/00000000000000000000.timeindex
data/lab-healthcheck-0/00000000000000000000.log
data/.kafka_cleanshutdown
data/__consumer_offsets-5/
data/__consumer_offsets-5/partition.metadata
data/__consumer_offsets-5/leader-epoch-checkpoint
data/__consumer_offsets-5/00000000000000000000.index
data/__consumer_offsets-5/00000000000000000000.timeindex
data/__consumer_offsets-5/00000000000000000000.log
data/bootstrap.checkpoint
data/__consumer_offsets-14/
data/__consumer_offsets-14/partition.metadata
data/__consumer_offsets-14/leader-epoch-checkpoint
data/__consumer_offsets-14/00000000000000000000.index
data/__consumer_offsets-14/00000000000000000000.timeindex
data/__consumer_offsets-14/00000000000000000000.log
data/__consumer_offsets-10/
data/__consumer_offsets-10/partition.metadata
data/__consumer_offsets-10/leader-epoch-checkpoint
data/__consumer_offsets-10/00000000000000000000.index
data/__consumer_offsets-10/00000000000000000000.timeindex
data/__consumer_offsets-10/00000000000000000000.log
data/__consumer_offsets-44/
data/__consumer_offsets-44/partition.metadata
data/__consumer_offsets-44/leader-epoch-checkpoint
data/__consumer_offsets-44/00000000000000000000.index
data/__consumer_offsets-44/00000000000000000000.timeindex
data/__consumer_offsets-44/00000000000000000000.log
data/__consumer_offsets-33/
data/__consumer_offsets-33/partition.metadata
data/__consumer_offsets-33/leader-epoch-checkpoint
data/__consumer_offsets-33/00000000000000000000.index
data/__consumer_offsets-33/00000000000000000000.timeindex
data/__consumer_offsets-33/00000000000000000000.log
data/__consumer_offsets-42/
data/__consumer_offsets-42/partition.metadata
data/__consumer_offsets-42/leader-epoch-checkpoint
data/__consumer_offsets-42/00000000000000000000.index
data/__consumer_offsets-42/00000000000000000000.timeindex
data/__consumer_offsets-42/00000000000000000000.log
data/__consumer_offsets-43/
data/__consumer_offsets-43/00000000000000001919.snapshot
data/__consumer_offsets-43/partition.metadata
data/__consumer_offsets-43/leader-epoch-checkpoint
data/__consumer_offsets-43/00000000000000000000.index
data/__consumer_offsets-43/00000000000000000000.timeindex
data/__consumer_offsets-43/00000000000000000000.log
data/__consumer_offsets-7/
data/__consumer_offsets-7/partition.metadata
data/__consumer_offsets-7/leader-epoch-checkpoint
data/__consumer_offsets-7/00000000000000000000.index
data/__consumer_offsets-7/00000000000000000000.timeindex
data/__consumer_offsets-7/00000000000000000000.log
data/__consumer_offsets-39/
data/__consumer_offsets-39/partition.metadata
data/__consumer_offsets-39/leader-epoch-checkpoint
data/__consumer_offsets-39/00000000000000000000.index
data/__consumer_offsets-39/00000000000000000000.timeindex
data/__consumer_offsets-39/00000000000000000000.log
data/__consumer_offsets-15/
data/__consumer_offsets-15/partition.metadata
data/__consumer_offsets-15/leader-epoch-checkpoint
data/__consumer_offsets-15/00000000000000000000.index
data/__consumer_offsets-15/00000000000000000000.timeindex
data/__consumer_offsets-15/00000000000000000000.log
data/__consumer_offsets-13/
data/__consumer_offsets-13/partition.metadata
data/__consumer_offsets-13/leader-epoch-checkpoint
data/__consumer_offsets-13/00000000000000000000.index
data/__consumer_offsets-13/00000000000000000000.timeindex
data/__consumer_offsets-13/00000000000000000000.log
data/recovery-point-offset-checkpoint
data/__consumer_offsets-17/
data/__consumer_offsets-17/partition.metadata
data/__consumer_offsets-17/leader-epoch-checkpoint
data/__consumer_offsets-17/00000000000000000000.index
data/__consumer_offsets-17/00000000000000000000.timeindex
data/__consumer_offsets-17/00000000000000000000.log
data/__consumer_offsets-23/
data/__consumer_offsets-23/partition.metadata
data/__consumer_offsets-23/leader-epoch-checkpoint
data/__consumer_offsets-23/00000000000000000000.index
data/__consumer_offsets-23/00000000000000000000.timeindex
data/__consumer_offsets-23/00000000000000000000.log
data/__consumer_offsets-47/
data/__consumer_offsets-47/partition.metadata
data/__consumer_offsets-47/leader-epoch-checkpoint
data/__consumer_offsets-47/00000000000000000000.index
data/__consumer_offsets-47/00000000000000000000.timeindex
data/__consumer_offsets-47/00000000000000000000.log
data/lab-telemetry-0/
data/lab-telemetry-0/partition.metadata
data/lab-telemetry-0/leader-epoch-checkpoint
data/lab-telemetry-0/00000000000000000000.index
data/lab-telemetry-0/00000000000000000000.timeindex
data/lab-telemetry-0/00000000000000014848.snapshot
data/lab-telemetry-0/00000000000000000000.log
data/__consumer_offsets-40/
data/__consumer_offsets-40/partition.metadata
data/__consumer_offsets-40/leader-epoch-checkpoint
data/__consumer_offsets-40/00000000000000000000.index
data/__consumer_offsets-40/00000000000000000000.timeindex
data/__consumer_offsets-40/00000000000000000000.log
data/__consumer_offsets-11/
data/__consumer_offsets-11/partition.metadata
data/__consumer_offsets-11/leader-epoch-checkpoint
data/__consumer_offsets-11/00000000000000000000.index
data/__consumer_offsets-11/00000000000000000000.timeindex
data/__consumer_offsets-11/00000000000000000000.log
data/__consumer_offsets-30/
data/__consumer_offsets-30/partition.metadata
data/__consumer_offsets-30/leader-epoch-checkpoint
data/__consumer_offsets-30/00000000000000000000.index
data/__consumer_offsets-30/00000000000000000000.timeindex
data/__consumer_offsets-30/00000000000000000000.log
data/__consumer_offsets-32/
data/__consumer_offsets-32/partition.metadata
data/__consumer_offsets-32/leader-epoch-checkpoint
data/__consumer_offsets-32/00000000000000000000.index
data/__consumer_offsets-32/00000000000000000000.timeindex
data/__consumer_offsets-32/00000000000000000000.log
data/__consumer_offsets-46/
data/__consumer_offsets-46/partition.metadata
data/__consumer_offsets-46/leader-epoch-checkpoint
data/__consumer_offsets-46/00000000000000000000.index
data/__consumer_offsets-46/00000000000000000000.timeindex
data/__consumer_offsets-46/00000000000000000000.log
data/__consumer_offsets-20/
data/__consumer_offsets-20/partition.metadata
data/__consumer_offsets-20/leader-epoch-checkpoint
data/__consumer_offsets-20/00000000000000000000.index
data/__consumer_offsets-20/00000000000000000000.timeindex
data/__consumer_offsets-20/00000000000000000000.log
data/replication-offset-checkpoint
data/__cluster_metadata-0/
data/__cluster_metadata-0/00000000000000007275-0000000001.checkpoint
data/__cluster_metadata-0/00000000000000014476-0000000001.checkpoint
data/__cluster_metadata-0/partition.metadata
data/__cluster_metadata-0/00000000000000028257.snapshot
data/__cluster_metadata-0/quorum-state
data/__cluster_metadata-0/00000000000000021673-0000000001.checkpoint
data/__cluster_metadata-0/leader-epoch-checkpoint
data/__cluster_metadata-0/00000000000000000000.index
data/__cluster_metadata-0/00000000000000000000.timeindex
data/__cluster_metadata-0/00000000000000000000.log
data/__consumer_offsets-26/
data/__consumer_offsets-26/partition.metadata
data/__consumer_offsets-26/leader-epoch-checkpoint
data/__consumer_offsets-26/00000000000000000000.index
data/__consumer_offsets-26/00000000000000000000.timeindex
data/__consumer_offsets-26/00000000000000000000.log
data/__consumer_offsets-34/
data/__consumer_offsets-34/partition.metadata
data/__consumer_offsets-34/leader-epoch-checkpoint
data/__consumer_offsets-34/00000000000000000000.index
data/__consumer_offsets-34/00000000000000000000.timeindex
data/__consumer_offsets-34/00000000000000000000.log
data/__consumer_offsets-18/
data/__consumer_offsets-18/partition.metadata
data/__consumer_offsets-18/leader-epoch-checkpoint
data/__consumer_offsets-18/00000000000000000000.index
data/__consumer_offsets-18/00000000000000000000.timeindex
data/__consumer_offsets-18/00000000000000000000.log
data/__consumer_offsets-27/
data/__consumer_offsets-27/partition.metadata
data/__consumer_offsets-27/leader-epoch-checkpoint
data/__consumer_offsets-27/00000000000000000000.index
data/__consumer_offsets-27/00000000000000000000.timeindex
data/__consumer_offsets-27/00000000000000000000.log
data/__consumer_offsets-12/
data/__consumer_offsets-12/partition.metadata
data/__consumer_offsets-12/leader-epoch-checkpoint
data/__consumer_offsets-12/00000000000000000000.index
data/__consumer_offsets-12/00000000000000000000.timeindex
data/__consumer_offsets-12/00000000000000000000.log
data/__consumer_offsets-45/
data/__consumer_offsets-45/partition.metadata
data/__consumer_offsets-45/leader-epoch-checkpoint
data/__consumer_offsets-45/00000000000000000000.index
data/__consumer_offsets-45/00000000000000000000.timeindex
data/__consumer_offsets-45/00000000000000000000.log
data/__consumer_offsets-2/
data/__consumer_offsets-2/00000000000000000003.snapshot
data/__consumer_offsets-2/partition.metadata
data/__consumer_offsets-2/leader-epoch-checkpoint
data/__consumer_offsets-2/00000000000000000000.index
data/__consumer_offsets-2/00000000000000000000.timeindex
data/__consumer_offsets-2/00000000000000000000.log
Restarting Kafka on kafka2...
Backup completed successfully at /mnt/Public/kafka2/20261110_113513
[0 e@Gunther:~/Dropbox/Documents/Code/Kafka/kafka-lab-1]
▶︎
```

Full successful execution of the backup process directed by ansible.

### Check backup file existence

```sh
[0 e@Gunther:~/Dropbox/Documents/Code/Kafka/kafka-lab-1]
▶︎ ls -ld /Volumes/Public/kafka*
drwx------ - e 10 Nov 11:37  /Volumes/Public/kafka1
drwx------ - e 10 Nov 11:37  /Volumes/Public/kafka2
drwx------ - e 10 Nov 11:37  /Volumes/Public/kafka3
[0 e@Gunther:~/Dropbox/Documents/Code/Kafka/kafka-lab-1]
▶︎ ls -l /Volumes/Public/kafka*
/Volumes/Public/kafka1:
drwx------ - e 10 Nov 11:38  20261110_113807

/Volumes/Public/kafka2:
drwx------ - e 10 Nov 11:37  20261110_113807

/Volumes/Public/kafka3:
drwx------ - e 10 Nov 11:37  20261110_113807
[0 e@Gunther:~/Dropbox/Documents/Code/Kafka/kafka-lab-1]
▶︎
```

All expected files are present. Note (by the name) all the backups were started at the same time.
---

(c) Ramón Barrios Láscar, 2026
