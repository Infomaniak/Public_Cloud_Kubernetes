# Heat

Create / Update / Delete the infrastructure

```bash
openstack stack create -t cluster.yaml --parameter whitelisted_ip_range="10.8.0.0/16" k8s --wait
openstack stack update -t cluster.yaml --parameter whitelisted_ip_range="10.8.0.0/16" k8s --wait
openstack stack delete k8s --wait
```

Check rating in CHF

```bash
openstack rating dataframes get -b 2021-08-25T00:00:00 | awk -F 'rating' '{print $2}' | awk -F "'" '{sum+=$3} END{print sum/50}'
```

Check your nodes security groups

```bash
openstack security group rule list k8s-masters
openstack security group rule list k8s-workers
```

## Sources

- https://github.com/kubernetes-sigs/kubespray/blob/master/docs/openstack.md
- https://github.com/kubernetes/cloud-provider-openstack/blob/master/docs/openstack-cloud-controller-manager/using-openstack-cloud-controller-manager.md#load-balancer
- https://docs.openstack.org/heat/latest/template_guide/openstack.html