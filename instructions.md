# Install and Configure a CentOS 7 ElasticSearch 5 Cluster for Chef Automate

By: Seth Thoenen

This guide is a combination of knowledge from the following sources:

- [Digital Ocean's Guide for a 3-node Elasticsearch 2 Cluster](https://www.digitalocean.com/community/tutorials/how-to-set-up-a-production-elasticsearch-cluster-on-centos-7)

- Chef Automate Tuning White Paper

- My own experience administering Chef infrastructure

## Part 1 - Getting Started

Assumptions for this tutorial:

- All systems are CentOS 7

- All systems have a static IP address

- All systems are in the samme VLAN with the local firewall's disabled

- All systems run on solid state disks

The three nodes in this tutorial have the following name and IP address:

chefels01 - 10.1.1.1
chefels02 - 10.1.1.2
chefels03 - 10.1.1.3

## Part 2 - Install Elasticsearch

For each Elasticsearch cluster nodes, do the following:

1. Install Java 8. Java 8 update 141 was the latest version at the time of this writing.

```
sudo yum install java-1.8.0-openjdk.x86_64 1:1.8.0.141-1.b16.el7_3
```

2. Download Elasticsearch 5.1.4:

```
sudo wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-5.4.1.rpm --no-check-certificate
```

3. Install Elasticsearch 5.1.4:

```
sudo rpm --install elasticsearch-5.4.1.rpm
```

4. Run the following commands to start Elasticsearch:

```
sudo systemctl daemon-reload
sudo systemctl enable elasticsearch.service
sudo systemctl start elasticsearch.service
```

5. Ensure Elasticsearch is running properly with the following command:

```
curl -X GET 'http://localhost:9200'
```

You should get output that looks like this:

```
{
  "name" : "chefels01",
  "cluster_name" : "chefels",
  "cluster_uuid" : "eUPaWsLRQByAG3QbHGzRBQ",
  "version" : {
    "number" : "5.4.1",
    "build_hash" : "2cfe0df",
    "build_date" : "2017-05-29T16:05:51.443Z",
    "build_snapshot" : false,
    "lucene_version" : "6.5.1"
  },
  "tagline" : "You Know, for Search"
}
```

## Part 3 - Configure Elasticsearch

For each Elasticsearch cluster nodes, do the following:

1. Edit the Elasticsearch configuration file at `/etc/elasticsearch/elasticsearch.yml`:

  1. Un-comment the line that contains `cluster.name:`. Specify a name for your cluster.

  ```
  cluster.name: chefels
  ```

  2. Un-comment the line that contains `node.name`. Use the system's hostname for this value:

  ```
  node.name: ${HOSTNAME}
  ```

  3. Un-comment the line that contains `bootstrap.memory_lock`. This enables memory locking.

  ```
  bootstrap.memory_lock: true
  ```

  4. Un-comment the line that contains `network.host`. Set it to the IP address of the system, and to listen to the loopback address:

  ```
  network.host: [ 10.1.1.1, _local_ ]
  ```

  5. Specify cluster node IP addresses. Fine and un-comment the line that contains `discovery.zen.ping.unicast.hosts` and specify the IP address of each cluster node:

  ```
	discovery.zen.ping.unicast.hosts: ["10.1.1.1", "10.1.1.2", "10.1.1.3"]
  ```

  6. Specify the minimum master nodes. This should be the (number of cluster nodes) / 2 + 1. I.e., for a 3 node cluster, there should be 2 minimum master nodes. Un-comment the line that contains `discovery.zen.minimum_master_nodes` and set it accordingly:

  ```
  discovery.zen.minimum_master_nodes: 2
  ```

2. Edit `/etc/sysconfig/elasticsearch` and un-comment the `MAX_LOCKED_MEMORY` variable:

```
MAX_LOCKED_MEMORY=unlimited
```

3. Edit `/usr/lib/systemd/system/elasticsearch.service` file and un-comment the `LimitMEMLOCK` variable:

```
LimitMEMLOCK=infinity
```

4. Edit `/etc/elasticsearch/jvm.options` and set the JVM heap size to 25% of the system's available RAM. By default, this will be set to 2GB in the below lines. Update this to meet your configuration needs:

```
-Xms2g
-Xmx2g
```

5. [Ensure the open file descriptor limit is set to 64000+](https://www.digitalocean.com/community/tutorials/how-to-set-up-a-production-elasticsearch-cluster-on-centos-7#configure-open-file-descriptor-limit-(optional))
. This should be on by default in CentOS 7. 

6. Restart the Elasticsearch service and ensure it is running:

```
sudo systemctl daemon-reload
sudo systemctl restart elasticsearch.service
sudo systemctl status elasticsearch.service
```

7. Ensure Elasticsearch is still happy:

```
curl -X GET 'http://localhost:9200'
```

You should get JSON output similar to the first time you ran this command

## Part 4 - Check Elasticsearch Cluster State

1. Run the following command from one cluster node and make sure all of your cluster nodes show up under `nodes`:

```
curl -XGET 'http://localhost:9200/_cluster/state?pretty'
```

2. Ensure memory locking is configured properly. Run the following command and ensure the `mlockall` property is set to `true` for each cluster node:

```
curl http://localhost:9200/_nodes/process?pretty`
```

## Part 5 - Update Chef Automate Instance

1. Create a backup of the Chef Automate instance:

```
sudo automate-ctl create-backup
```

2. Update `/etc/delivery/delivery.rb` to point to the Elasticsearch cluster nodes:

```
elasticsearch['urls'] = ['http://10.1.1.1:9200', 'http://10.1.1.2:9200', 'http://10.1.1.3:9200']
```

3. Restore from the backup that was just created, omitting the `delivery.rb` configuration. This will populate the Elasticsearch cluster with data from the built in Elasticsearch instance

```
sudo -automate-ctl restore-backup (path) --no-config
```

4. Update the shared count to the number of Elasticsearch cluster nodes:

```
sudo curl -XPUT http://localhost:8080/elasticsearch/_template/index_defaults -d '{"template": "*", "settings": { "number_of_shards": 3}}'
```

You should receive JSON output that states the transaction was successful:

```
{"acknowledged":true}
```

5. Tune LogStash

  1. Create a file at `/etc/security/limits.d/90-chef-performance.conf` to configure limits tuning:

  ```
  delivery   soft   nproc   unlimited
  delivery   hard   nproc   unlimited
  delivery   soft   nofile  unlimited
  delivery   hard   nofile  unlimited
  ```

  2. Update `/etc/delivery/delivery.rb` to configure JVM settings, worker threads and batch size:

  ```
  logstash['heap_size'] = "2g"
  logstash['config'] = {
    "pipeline" => {
      "batch" => {
        "size" => 40
      },
      "workers" => 16
    }
  } 
  ```

  3. Reconfigure Chef Automate

  ```
  sudo automate-ctl reconfigure
  ```

Note: The RabbigMQ tuning commands did not work in CentOS 7.

## Part 6 - Tune All Linux Systems

1. Set SELinux to permissive

```
Setenforce 0
sed -i 's/SELINUX=enforcing/SELINUX=permissive/g' /etc/selinux/config
```

2. Configure RHEL boot-time kernel tuning by adding the following arguments to the `GRUB_CMDLINE_LINUX` variable in `/etc/default/grub`

```
scsi_mod.use_blk_mq=Y elevator=noop transparent_hugepage=never
```

3. Tune Linux kernel VM tuning by creating a file at `/etc/sysctl.d/chef-highperf.conf` and populate with the following contents:

```
vm.swappiness=10
vm.max_map_count=256000
vm.dirty_ratio=20
vm.dirty_background_ratio=30
vm.dirty_expire_centisecs=30000 
```

4. Ensure all filesystems are XFS (skipped in this tutorial)

5. Reboot