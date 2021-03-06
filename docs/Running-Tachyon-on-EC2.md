---
layout: global
title: Running Tachyon on EC2
---

Tachyon can be launched on EC2 using the [Spark EC2
scripts](https://github.com/mesos/spark/wiki/EC2-Scripts) that come with Spark. These scripts let
you launch, pause and destroy clusters that come automatically configured with HDFS, Spark, Apache
Mesos, Shark, and Tachyon.

# Launching a Cluster

To run a Tachyon cluster on EC2, first sign up for an Amazon EC2 account
on the [Amazon Web Services site](http://aws.amazon.com/). Then,
download Spark to your local machine:

    $ wget https://github.com/downloads/mesos/spark/spark-0.6.0-sources.tar.gz
    $ tar xvfz spark-0.6.0-sources.tar.gz

The `ec2` directory contains the scripts to set up a cluster. Detailed instructions are available in
the [Spark EC2 guide](https://github.com/mesos/spark/wiki/EC2-Scripts). In a nutshell, you will need
to do:

    $ spark-0.6.0/ec2/spark-ec2 -a ami-691d9100 -k <keypair-name> -i <key-file> -s <num-slaves> launch <cluster-name>

Where `<keypair>` is the name of your EC2 key pair (that you gave it
when you created it), `<key-file>` is the private key file for your key
pair, `<num-slaves>` is the number of slave nodes to launch (try 1 at
first), and `<cluster-name>` is the name to give to your cluster. This
creates a cluster on EC2 using a pre-built machine image that has
Tachyon, Spark, and Shark.

Login to the master using `spark-ec2 login`:

    $ ./spark-ec2 -k key -i key.pem login <cluster-name>

Then, config Tachyon in `tachyon` folder

    $ cd /root/tachyon/conf
    $ cp tachyon-env.sh.template tachyon-env.sh

Edit `tachyon-env.sh` file. Setup `TACHYON_HDFS_ADDRESS=hdfs://HDFS_HOSTNAME:HDFS_PORT`.

Edit `slaves` file, add slaves' hostnames into it. Sync the configuration to all nodes.

    $ cd /root/tachyon/conf
    $ /root/mesos-ec2/copy-dir .

Edit Spark `/root/spark/conf/spark-env.sh`, add:

    export SPARK_CLASSPATH+=/root/tachyon/target/tachyon-1.0-SNAPSHOT-jar-with-dependencies.jar
    SPARK_JAVA_OPTS+=" -Dtachyon.hdfs.address=hdfs://HDFS_HOSTNAME:HDFS_PORT -Dspark.default.parallelism=1 "
    export SPARK_JAVA_OPTS

Edit Spark's `hdfs-site.xml`, add:

    <property>
      <name>fs.tachyon.impl</name>
      <value>tachyon.hadoop.TachyonFileSystem</value>
      <description></description>
    </property>

Sync Spark's new configuration to all nodes:

    $ cd /root/spark/conf
    $ /root/mesos-ec2/copy-dir .

Put a file X into HDFS. Run Spark Shell:

    $ ./spark-shell
    $ val s = sc.textFile("tachyon://tachyon_master_host:9999/X")
    $ s.count()

Take a look at `MasterMachineHostName:9998`, there should be a dataset info there. The Tachyon will
load `hdfs://HDFS_HOSTNAME:HDFS_PORT/X` into the system.