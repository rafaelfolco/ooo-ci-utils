
# Install 
Grafana w/ Graphite as data source on (CentOS) localhost 

## Graphite 

Start graphite in a container:

```sh
docker run -d\
 --name graphite\
 --restart=always\
 -p 80:80\
 -p 2003-2004:2003-2004\
 -p 2023-2024:2023-2024\
 -p 8125:8125/udp\
 -p 8126:8126\
 graphiteapp/graphite-statsd
```
## Grafana

Install Grafana (CentOS) and start service:

```sh
$ sudo yum install https://s3-us-west-2.amazonaws.com/grafana-releases/release/grafana-4.6.3-1.x86_64.rpm
$ sudo service grafana-server start
```

# Configure
1. Acess Grafana http://localhost:3000
1. Login as admin/admin
1. Check graphite IP (docker)
    ```sh
    ip a
    <...>
    9: docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
        link/ether 02:42:6c:f9:fd:28 brd ff:ff:ff:ff:ff:ff
        inet 172.17.0.1/16 scope global docker0
    <...>
    ```
1. Add a datasource to your dashboard: url=``http://172.17.0.1:80``
1. Click on Test and Save button
1. Add Metric: * | gauges | foo 
1. Insert data points in "\<metric\>:\<value\>|\<type\>" format: 
    ```sh
    echo "foo:20|g" | nc -u -w1 172.17.0.1 8125
    ```
1. Save your dashboard

# References
* https://graphite.readthedocs.io/en/latest/install.html
* http://docs.grafana.org/installation/rpm/
* http://docs.grafana.org/features/datasources/graphite/
