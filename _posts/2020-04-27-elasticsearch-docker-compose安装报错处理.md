---
layout: post
title:  "elasticsearch-docker-compose安装报错处理"
date:   2020-04-27
categories: elasticsearch
---

	在使用docker-compose安装elasticsearch时报如下错误：

	es01    | {"type": "server", "timestamp": "2020-04-27T10:12:26,394Z", "level": "INFO", "component": "o.e.t.TransportService", "cluster.name": "es-docker-cluster", "node.name": "es01", "message": "publish_address {172.23.0.2:9300}, bound_addresses {0.0.0.0:9300}" }
	es01    | {"type": "server", "timestamp": "2020-04-27T10:12:27,352Z", "level": "INFO", "component": "o.e.b.BootstrapChecks", "cluster.name": "es-docker-cluster", "node.name": "es01", "message": "bound or publishing to a non-loopback address, enforcing bootstrap checks" }
	es01    | ERROR: [1] bootstrap checks failed
	
<!--more-->


----------------

我的配置如下：
	
	version: '3.1'
	services:
	  es01:
	    image: elasticsearch:7.6.2
	    container_name: es01
	    restart: always
	    environment:
	      - node.name=es01
	      - cluster.name=es-docker-cluster
	      - bootstrap.memory_lock=true
	      - cluster.initial_master_nodes=es01
	      - "ES_JAVA_OPTS=-Xms300m -Xmx300m"
	    ports:
	      - 9200:9200
	    ulimits:
	      memlock:
	        soft: -1
	        hard: -1
	    volumes:
	      - ./elasticsearch/data:/usr/share/elasticsearch/data
	      - ./elasticsearch/plugins:/usr/share/elasticsearch/plugins
	      - ./elasticsearch/logs:/usr/share/elasticsearch/logs

"bound or publishing to a non-loopback address "这个错误的解决方法，就是增加

	network.host=127.0.0.1
	http.host=0.0.0.0
	transport.host=localhost

最终配置如下：


	version: '3.1'
	services:
	  es01:
	    image: elasticsearch:7.6.2
	    container_name: es01
	    restart: always
	    environment:
	      - node.name=es01
	      - network.host=127.0.0.1
	      - http.host=0.0.0.0
	      - transport.host=localhost
	      - cluster.name=es-docker-cluster
	      - bootstrap.memory_lock=true
	      - cluster.initial_master_nodes=es01
	      - "ES_JAVA_OPTS=-Xms300m -Xmx300m"
	    ports:
	      - 9200:9200
	    ulimits:
	      memlock:
	        soft: -1
	        hard: -1
	    volumes:
	      - ./elasticsearch/data:/usr/share/elasticsearch/data
	      - ./elasticsearch/plugins:/usr/share/elasticsearch/plugins
	      - ./elasticsearch/logs:/usr/share/elasticsearch/logs