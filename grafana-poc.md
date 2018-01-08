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

# grafyaml
1. Clone and configure grafyaml
```sh
https://github.com/openstack-infra/grafyaml.git
cd grafyaml
sudo pip install -e requirements.txt
sudo python setup.py install
```
1. Create an API key
http://localhost:3000/org/apikeys
1. Create your dashboard and save as YAML (test-ooo.yaml)
```
dashboard:
  title: 'TripleO CI Metrics'
  rows:
    - title: Folco's Row
      height: 100px
      panels:
        - title: Folco's Panel
          content: |
            **This is a Grafana PoC using Graphite as data source. That's All Folco's**
          type: text

    - title: Deployment Time
      showTitle: true
      height: 250px
      panels:
        - title: Undercloud Deployment Time
          type: graph
          span: 6
          leftYAxisLabel: "time"
          y_formats:
            - s
            - none
          targets:
            - target: stats.gauges.undercloud_deployment_time
        - title: Overcloud Deployment Time
          type: graph
          span: 6
          leftYAxisLabel: "time"
          y_formats:
            - s
            - none
          targets:
            - target: stats.gauges.overcloud_deployment_time
```
1. Edit your config file (etc/grafyaml.conf)
```
#...
[grafana]
# URL for grafana server. (string value)
url = http://localhost:3000

# API key for access grafana. (string value)
apikey = eyJrIjoiNmFGSDhYbW5EMXlYa0ZLQm5wdDVoQnMyRVp2Q2FtNHQiLCJuIjoidHJpcGxlbyIsImlkIjoxfQ==
#...
```
1. Test your dashboard
```sh
grafana-dashboard --config-file etc/grafyaml.conf validate test-ooo.yaml
INFO:grafana_dashboards.cmd:Validating schema in test-ooo.yaml
SUCCESS!

grafana-dashboard --config-file etc/grafyaml.conf update test-ooo.yaml
INFO:grafana_dashboards.cmd:Updating schema in test-ooo.yaml
INFO:grafana_dashboards.builder:Number of datasources to be updated: 0
INFO:grafana_dashboards.builder:Number of dashboards to be updated: 1
```
![Grafana Setup](/grafana-poc.png)

# References
* https://graphite.readthedocs.io/en/latest/install.html
* http://docs.grafana.org/installation/rpm/
* http://docs.grafana.org/features/datasources/graphite/
