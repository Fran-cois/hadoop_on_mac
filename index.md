## My first experience installing hadoop on a mac os machine (high Sierra)

I wanted to implement a mapreduce program using hadoop.

I first tried to use the recommanded guide :
[getting hadoop with a Pseudo distributed mode](http://hadoop.apache.org/docs/current/hadoop-project-dist/hadoop-common/SingleCluster.html)

While playing around the extracted archive I ran into some issue due to wrong env variables.

So I decided to use [brew](https://brew.sh/index_fr).

### Install brew if necessary

```
if [[ $(brew -v) ]]; then
    echo "you already have brew on your computer";
else
    /usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
fi
```

### Install java if necessary

```
if [[ $(java -version) ]]; then
    echo "you already have java on your computer";
else
    brew install java
fi
```

### Installation of hadoop
```
brew install hadoop
```

### Allow hadoop to communicate on your local machine:
This step has been really weird to me, you need to allow remote login to your localhost !
There are two ways to do it :  

Manualy:
```
Go to Settings, and allow Remote Login in the System Preferences.
In French that's : "session à distance" in "partage".
```
Or with a command line :
```
sudo systemsetup -setremotelogin on
```
I would recommand to use ssh key and turn off the sharing when the hadoop service is not used.

```
 ssh-keygen -t rsa # if no key has been already generated.
 cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
```
The command `ssh localhost` should work now.

###  Basic Configuration of hadoop
First Export your hadoop version in a var:
```
hadoop_version=3.1.1
hadoop_local_path=/usr/local/Cellar/hadoop/$hadoop_version/libexec/etc/hadoop/
```
Export the paths in `/usr/local/Cellar/hadoop/$hadoop_version/libexec/etc/hadoop/hadoop-env.sh`
### Get all the env variable for hadoop
```
echo '
export JAVA_HOME=$(/usr/libexec/java_home)
export HADOOP_CLASSPATH=$JAVA_HOME/lib/tools.jar
export HADOOP_OPTS="$HADOOP_OPTS
 -Djava.net.preferIPv4Stack=true
 -Djava.security.krb5.realm=
 -Djava.security.krb5.kdc="
 ' >>
 $hadoop_local_path/hadoop-env.sh
'
```
### Set the core-site.xml
This is where the hdfs (port and address)configurations are.
```
echo '
<configuration>
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://localhost:9000</value>
    </property>
</configuration>
'  >  $hadoop_local_path/core-site.xml
```

### Set the mapred-site.xml
```
echo'
<configuration>
    <property>
        <name>mapred.job.tracker</name>
        <value>localhost:8021</value>
    </property>
</configuration>
' > $hadoop_local_path/mapred-site.xml
```

### Set the  hdfs-site.xml
This will set the replication of the hdfs (by default 3, here we only need one).
```
echo'
<configuration>
    <property>
        <name>dfs.replication</name>
        <value>1</value>
    </property>
</configuration>
' > $hadoop_local_path/hdfs-site.xml
```
## Setup Hdfs
Run this command : `hdfs namenode -format` 
## Now we can start all the hadoop services :
```
cd /usr/local/Cellar/hadoop/$hadoop_version/libexec/
./sbin/start-all.sh
```
Now the yarn web ui  should be up, you should check on http://localhost:8088.

The "Simple" hadoop installation is now done, let's check it with the word count / map reduce example  !

First copy the code from http://hadoop.apache.org/docs/current/hadoop-mapreduce-client/hadoop-mapreduce-client-core/MapReduceTutorial.html
to a file named WordCount.java 

And then execute this script.
```
#!/bin/bash

export PERSO_HADOOP_PATH=/usr/local/Cellar/hadoop/$hadoop_version/libexec
mkdir input

touch input/file1
touch input/file2

echo "Hello World Bye World" >> input/file1
echo "Hello Hadoop Goodbye Hadoop" >> input/file2

# compile java
$PERSO_HADOOP_PATH/bin/hadoop com.sun.tools.javac.Main WordCount.java
jar cf wc.jar WordCount*.class


# remove old files
hadoop fs -rm -r input
hadoop fs -rm -r output

# copy new input
hadoop fs -put input input

$PERSO_HADOOP_PATH/bin/hadoop jar wc.jar WordCount  input output


# read output
$PERSO_HADOOP_PATH/bin/hadoop fs -cat output/*
#bin/hdfs dfs -get output output

rm *.class
rm -rf input

```
