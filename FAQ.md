# FAQ

## Debugging TripleO

### check introspection failure (timeout)
```sh
sudo journalctl -u openstack-ironic-inspector
```

### check which resources failed to deploy the overcloud
```sh
openstack stack resource list overcloud -n 5 --filter status=FAILED -c physical_resource_id -c resource_name -c resource_type -c stack_name
```

### show the reason of a failed resource
```sh 
openstack stack resource show overcloud-Compute-mtgxddccdppm-0-ls3jxjtmk5je-NovaCompute-555ow6o7upus HostsEntryDeployment |grep resource_status_reason
```

### check pre-introspection failures
```sh
for i in $(mistral execution-list | grep tripleo.validations.*ERROR | awk '{print $2}'); do mistral execution-get-output $i; done
```

### delete introspection workflow 
```sh
for i in $(mistral execution-list | grep tripleo.validations.* | awk '{print $2}'); do mistral execution-delete $i; done
```

