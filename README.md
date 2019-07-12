# Elastic stack (ELK) on Docker


＃＃ 要求

###主機設置

* [Docker](https://www.docker.com/community-edition#/download) version **17.05+**
* [Docker Compose](https://docs.docker.com/compose/install/) version **1.6.0+**
* 1 GB of RAM

默認情況下，公開以下端口
* 5000: Logstash TCP input
* 9200: Elasticsearch HTTP
* 9300: Elasticsearch TCP transport
* 5601: Kibana


### Docker for Desktop

#### Windows

確保為C：啟用 [Shared Drives] .

#### macOS

默認的Docker for Mac配置允許掛載來自`/ Users /`，`/ Volumes /`，`/ private /`和`/ tmp`的文件. 確保將存儲庫克隆到其中一個位置或按照說明中的說明進行操作[documentation][mac-mounts] 添加。.

## Usage

### Bringing up the stack

Clone this repository, then start the stack using Docker Compose:

```console
$ docker-compose up
```

你還可以通過在上面的命令中添加`-d`來在後台運行所有服務.

information_source：每當切換分支或更新基本映像時，必須首先運行`docker-compose build`


## Initial setup

### Setting up user authentication


The stack is pre-configured with the following **privileged** bootstrap user:

* user: *elastic*
* password: *changeme*

Although all stack components work out-of-the-box with this user, we strongly recommend using the unprivileged [built-in
users][builtin-users] instead for increased security. Passwords for these users must be initialized:

```console
$ docker-compose exec -T elasticsearch bin/elasticsearch-setup-passwords auto --batch
```

Passwords for all 6 built-in users will be randomly generated. Take note of them and replace the `elastic` username with
`kibana` and `logstash_system` inside the Kibana and Logstash configuration files respectively. See the
[Configuration](#configuration) section below.

> :information_source: Do not use the `logstash_system` user inside the Logstash *pipeline* file, it does not have
> sufficient permissions to create indices. Follow the instructions at [Configuring Security in Logstash][ls-security]
> to create a user with suitable roles.

Restart Kibana and Logstash to apply the passwords you just wrote to the configuration files.

```console
$ docker-compose restart kibana logstash
```



### Injecting data

Give Kibana about a minute to initialize, then access the Kibana web UI by hitting
[http://localhost:5601](http://localhost:5601) with a web browser and use the following default credentials to log in:

* user: *elastic*
* password: *\<your generated elastic password>*

Now that the stack is running, you can go ahead and inject some log entries. The shipped Logstash configuration allows
you to send content via TCP:

```console
$ nc localhost 5000 < /path/to/logfile.log
```

You can also load the sample data provided by your Kibana installation.


#### On the command line

Create an index pattern via the Kibana API:

```console
$ curl -XPOST -D- 'http://localhost:5601/api/saved_objects/index-pattern' \
    -H 'Content-Type: application/json' \
    -H 'kbn-version: 7.2.0' \
    -u elastic:<your generated elastic password> \
    -d '{"attributes":{"title":"logstash-*","timeFieldName":"@timestamp"}}'
```



### How to configure Elasticsearch

The Elasticsearch configuration is stored in [`elasticsearch/config/elasticsearch.yml`][config-es].


```yml
elasticsearch:

  environment:
    network.host: _non_loopback_
    cluster.name: my-cluster
```



### How to configure Kibana

The Kibana default configuration is stored in [`kibana/config/kibana.yml`][config-kbn].



### How to configure Logstash

The Logstash configuration is stored in [`logstash/config/logstash.yml`][config-ls].




## Storage

### How to persist Elasticsearch data

The data stored in Elasticsearch will be persisted after container reboot but not after container removal.

In order to persist Elasticsearch data even after removing the Elasticsearch container, you'll have to mount a volume on
your Docker host. Update the `elasticsearch` service declaration to:

```yml
elasticsearch:

  volumes:
    - /path/to/storage:/usr/share/elasticsearch/data
```

This will store Elasticsearch data inside `/path/to/storage`.


### Swarm mode

Experimental support for Docker [Swarm mode][swarm-mode] is provided in the form of a `docker-stack.yml` file, which can
be deployed in an existing Swarm cluster using the following command:

```console
$ docker stack deploy -c docker-stack.yml elk
```

If all components get deployed without any error, the following command will show 3 running services:

```console
$ docker stack services elk
```

> :information_source: To scale Elasticsearch in Swarm mode, configure *zen* to use the DNS name `tasks.elasticsearch`
instead of `elasticsearch`.
