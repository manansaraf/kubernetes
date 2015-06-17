# Proposal: HTTP Cloudprovider

# Motivation 
Currently, the cloudproviders are designed so that they get directly compiled into kubernetes. It is very hard to add custom cloudproviders to the kubernetes platform while using  existing kube containers. Therefore, we would like to build an HTTP Cloudprovider which would be able to send a HTTP request by converting the cloudprovider interfaces to an HTTP API. The cloudprovider would need an HTTP endpoint so that kubernetes would be able to send a request to them and then the cloudproviders would be able to send the desired service information back to kubernetes. 

# Goals
* Provide an HTTP API layer so that custom cloud providers can easily communicate with kubernetes.(Eventually we want the current cloud providers to also be moved to this model)

# Specification

The user should specify a config file in which the **"client-url"** field is required and the value should be the IP Address or the DNS Hostname of the cloudprovider which the HTTP API can communicate with.

First the API checks for which interfaces in [cloud.go](https://github.com/GoogleCloudPlatform/kubernetes/blob/master/pkg/cloudprovider/cloud.go) are implemented.

A GET request for /v1/{Interface}/supported will return whether the interface is supported or not.

The following methods must be implemented by the HTTP cloudprovider. If a cloudprovider does not implement a resource, it can return false for a given GET /{interface}/supported call

If any of the functions are unimplemented or there is an error in processing the request, please send back a HTTP 404 Error with the error mesage as it's value.

For the GET requests all the examples shown is the body kubernetes expects as a return value.

## Provider Name

#### GET /v1/providerName
Returns the cloud provider name.

|Name|Description|Example Value|
|----|-----------|-------------|
|providerName|Name of the cloud provider.|"my-cloud"|
##### Example
Expected return value
```json
{
    "providerName": "my-cloud"
}
```
#### Properties

## Instances
The instances interface allows the kubernetes platform to query the cloud provider about the instances currently within the cloud provider system.

### Methods
#### GET /v1/instances/support
Returns whether the Instance interface is supported or not.

|Name|Description|Example Value|
|----|-----------|-------------|
|support|Whether the interface is supported or not.|true|
##### Example
Expected return value
```json
{
    "support": true
}
```

#### GET /v1/instances/{InstanceName}/nodeAddresses 
Returns all node addresses for the given instance.

|Name|Description|Example Value|
|----|-----------|-------------|
|nodeAddresses|List of node adress objects.||
|nodeAddresses:type|The IP type relate to the node address. Can only be either "InternalIP", "ExternalIP", "Hostname" or "LegacyHostIP".|"InternalIP"|
|nodeAddresses:address|The actual node's address.|"127.0.0.1"|
##### Example
Expected return value
```json
{
    "nodeAddresses": [
        {
            "type": "InternalIP",
            "address": "127.0.0.1"
        }
    ]
}
```

#### GET /v1/instances/{InstanceName}/ID 
Returns the Instance ID for the given instance.

|Name|Description|Example Value|
|----|-----------|-------------|
|instanceID|ID of the cloud provider.|"my-cloud-name1"|
##### Example
Expected return value
```json
{
    "instanceID": "my-cloud-name1"
}
```

#### GET /v1/instances/{filter} 
Returns all instances where the name matches the word "filter". Filter is a regular expression.

|Name|Description|Example Value|
|----|-----------|-------------|
|instances|List of all instances matching the filter.||
|instances:instanceName|Name of the instance.|"cloud-1"|
##### Example
Expected return value
```json
{
    "instances": [
        {
            "instanceName": "cloud-1"
        }
    ]
}
```

#### GET /v1/instances/{NodeName}/resources
Returns all the resources for the given node. 

|Name|Description|Example Value|
|----|-----------|-------------|
|resources|List of all resources in the instance.||
|resources:resourceName|Name of the resource. The type/name of resources are restricted to the list below.|"cpu"|
|resources:quantity|Object represent the quantity of the specified resource.||
|resources:quantity:amount|The actual amount of the resource. It should be in the units specified below in resource types.|4|
|resources:quantity:format|The format in which the amount is stored. Can only be either "DecimalExponent", "BinarySI" or "DecimalSI".|"DecimalSI"|

##### Resource Types
|Name|Description|
|----|-----------|
|cpu|The CPU amount in cores.|
|memory|The amount of memory allocated, in bytes.|
|storage|The Volume size, in bytes.|
|pods|The number of pods.|
|services|The number of services.|
|replicationcontrollers|The number of Replication Controllers.|
|resourequotas|The number of resource quotas.|
|secrets|The number of resource secrets.|
|persistentvolumeclaims|The number of resource persistent volume claims.|
##### Example
Expected return value
```json
{
    "resources": [
        {
            "resourceName": "cpu",
            "quantity":
                {
                    "amount": 4,
                    "format": "DecimalSI"
                }
        }
    ]
}
```


#### POST /v1/instances/AddSSHKey
Adds the SSHKey given to all the instances, it returns whether the key was added or not.
##### Sender body
|Name|Description|Example Value|
|----|-----------|-------------|
|user|The user's whose SSH key needs to be added.|"name"|
|keydata|The actual ssh key. Any format allowed not restricted to strings.|"zPjoihsswRTGIUHKLHIHO345@435"|
##### Example
```json
{
    "user": "name",
    "keyData": "zPjoihsswRTGIUHKLHIHO345@435"
}
```
##### Expected Return Value:
|Name|Description|Example Value|
|----|-----------|-------------|
|SSHKeyAdded|Boolean value regarding whether the SSH Key was added or not.|false|

##### Example
```json
{
    "SSHKeyAdded": false
}
```

## TCPLoadBalancer

The TCPLoadBalancer interface allows the kubernetes platform to query the cloud provider about the Load Balancer the cloud provider has and requests to change the load balancer as required.
### Methods

#### GET /v1/TCPLoadBalancer/support
Know whether the interface is supported or not.

|Name|Description|Example Value|
|----|-----------|-------------|
|support|Whether the interface is supported or not.|true|
##### Example
Expected return value
```json
{
    "support":false
}
```

#### GET /v1/TCPLoadBalancer/{region}/{name}
Returns the TCP Load Balancer if it exists in the region with the particular name.

|Name|Description|Example Value|
|----|-----------|-------------|
|loadBalancerStatus|List of ingress points for the load balancer.||
|loadBalancerStatus:IP|The IP address of the ingress point. This is a optional field as only one of IP or hostname is required.|"127.0.0.1"|
|loadBalancerStatus:hostname|The hostname of the ingress point. This is a optional field as only one of IP or hostname is required.|"my-cloud-host1"|
|exists|Whether the TCP Load Balancer exists or not.|true|
##### Example
Expected return value
```json
{
    "loadBalancerStatus": [
        {
            "IP":"127.0.0.1",
            "hostname":"my-cloud-host1"
        }
    ],
    "exists":true
}
```

#### POST /v1/TCPLoadBalancer/create/
Creates a new tcp load balancer and return the status of the balancer.
##### Sender body
|Name|Description|Example Value|
|----|-----------|-------------|
|loadBalancerName|The name of the TCP load balancer.|"my-tcp-balancer"|
|region|The location of the load balancer.|"USA"|
|externalIP|The external IP of the load balancer.|"127.0.0.1"|
|ports|List of all the service port objects.||
|ports:portName|Name of the port. If only one port is present then this field is **optional**.|"SMTP"|
|ports:protocol|The protocol the port works on. The value can only be either "UDP" or "TCP".|"TCP"|
|ports:port|The port that will be exposed on the service.|1234|
|ports:targetPort|The target port on pods selected by this service. This field is optional. It can be **string** or **int** type. If not provided, will use **ports:port** value.|1234|
|ports:nodePort|The port on each node on which this service is exposed.|1234|
|hosts|A list of all host names.||
|hosts:hostname|A host name.|"my-cloud-host1"|
|ServiceAffinity|The session affinity. The value can only be either "None" or "ClientIP".|"None"|
##### Example
```json
{
    "loadBalancerName":"my-tcp-balancer",
    "region":"USA",
    "externalIP":"127.0.0.1",
    "ports":[
        {
            "portName":"SMTP",
            "protocol":"TCP",
            "port":1234,
            "targetPort":1234,
            "nodePort":1234
        }
    ],
    "hosts":[
        {
            "hostname":"my-cloud-host1"
        }
    ],
    "ServiceAffinity":"None"
}
```
##### Expected return value
|Name|Description|Example Value|
|----|-----------|-------------|
|loadBalancerStatus|List of ingress points for the load balancer.||
|loadBalancerStatus:IP|The IP address of the ingress point. This is a optional field as only one of IP or hostname is required.|"127.0.0.1"|
|loadBalancerStatus:hostname|The hostname of the ingress point. This is a optional field as only one of IP or hostname is required.|"my-cloud-host1"|
|exists|Whether the TCP Load Balancer exists or not.|true|
##### Example
```json
{
    "loadBalancerStatus": [
        {
            "IP":"127.0.0.1",
            "hostname":"my-cloud-host1"
        }
    ],
    "exists":true
}
```

#### POST /v1/TCPLoadBalancer/update/{region}/{name}
Updates the hosts given in the Load Balancer specified.
##### Sender body
|Name|Description|Example Value|
|----|-----------|-------------|
|hosts|A list of all host names.||
|hosts:hostname|A host name.|"my-cloud-host1"|
##### Example
```json
{
    "hosts":[
        {
            "hostname":"my-cloud-host1"
        }
    ]
}
```
##### Expected return body:
|Name|Description|Example Value|
|----|-----------|-------------|
|TCPLoadBalancerUpdated|Whether the TCP Load Balancer has been updated or not.|true|
##### Example
```json
{
    "TCPLoadBalancerUpdated": true
}
```

#### DELETE /v1/TCPLoadBalancer/{region}/{name}
Deletes the specified Load Balancer.

|Name|Description|Example Value|
|----|-----------|-------------|
|TCPLoadBalancerDeleted|Whether the specified TCP Load Balancer is deleted or not.|true|
##### Example
Expected return value
```json
{
    "TCPLoadBalancerDeleted": true
}
```
**Note:** Return **true** if the Load Balancer was deleted or it didn't exist before. This construction is useful because many cloud providers' load balancers have multiple underlying components, meaning a Get could say that the LB doesn't exist even if some part of it is still laying around. 
## Zones
The Zones interface allows the kubernetes platform to query the cloud provider about the Zones and which areas are having a failed state. 

### Methods

#### GET /v1/zones/support
Know whether the interface is supported or not.

|Name|Description|Example Value|
|----|-----------|-------------|
|support|Whether the interface is supported or not.|true|
##### Example
Expected Return Value
```json
{
    "support":true
}
```

#### GET /v1/zones
Returns the Zone containing the current failure zone and locality region that the program is running in.

|Name|Description|Example Value|
|----|-----------|-------------|
|zone|The zone object, it represents the location of the machine.||
|zone:failureDomain|The failure zone.|"my-zone-2"|
|zone:region|Locality region that the program is running in.|"USA"|
##### Example
Expected Return Value
```json
{
    "zone":{
        "failureDomain":"my-zone-2",
        "region":"USA"
    }
}
```
## Clusters
The clusters interface allows the kubernetes platform to query information about the clusters inside the cloud provider.

### Methods

#### GET /v1/clusters/support
Know whether the interface is supported or not.

|Name|Description|Example Value|
|----|-----------|-------------|
|support|Whether the interface is supported or not.|true|
##### Example
Expected Return Value
```json
{
    "support":true
}
```

#### GET /v1/clusters/list
Returns a list of all clusters.

|Name|Description|Example Value|
|----|-----------|-------------|
|clusters|The list of all cluster objects.||
|clusters:clusterName|The name of the cluster.|"my-cluster"|
##### Example
Expected Return Value
```json
{
    "clusters": [
        {
            "clusterName":"my-cluster"
        }
    ]
}
```

#### GET /v1/clusters/{clusterName}/master
Returns the address of the master of the cluster provided.

|Name|Description|Example Value|
|----|-----------|-------------|
|masterAddress|The address of the master in the cluster. Can be a DNS Hostname or an IP Address.|"127.0.0.1"|
##### Example
Expected Return Value
```json
{
    "masterAddress":"127.0.0.1"
}
```

## Routes
The routes interface allows the kubernetes platform to query information about the routes and change them inside the cloud provider.

### Methods

#### GET /v1/routes/support
Know whether the interface is supported or not.

|Name|Description|Example Value|
|----|-----------|-------------|
|support|Whether the interface is supported or not.|true|
##### Example
Expected Return Value
```json
{
    "support":true
}
```
#### GET /v1/routes/list/{filter}
Returns a list of all the routes matching the word "filter".

|Name|Description|Example Value|
|----|-----------|-------------|
|routes|The list of route objects.||
|routes:routeName|The name of the routing rule.|"abc"|
|routes:targetInstance|The name of the instance as specified in routing rules for the cloud-provider.|"a1-small"|
|routes:destinationCIDR|The CIDR format IP range that this routing rule applies to.|"192.168.1.0/24"|
|routes:description|Description of the routing rule.|"abc"|
##### Example
Expected Return Value
```json
{
    "routes": [
        {
            "routeName":"abc",
            "targetInstance":"a1-small",
            "destinationCIDR":"192.168.1.0/24",
            "description":"abc"
        }
    ]
}
```
#### POST /v1/routes/create
Creates a route inside the clou provider and returns the route just created.
##### Sender Body
|Name|Description|Example Value|
|----|-----------|-------------|
|routes|The route object||
|route:routeName|The name of the routing rule.|"abc"|
|route:targetInstance|The name of the instance as specified in routing rules for the cloud-provider.|"a1-small"|
|route:destinationCIDR|The CIDR format IP range that this routing rule applies to.|"192.168.1.0/24"|
|route:description|Description of the routing rule.|"abc"|
##### Example
```json
{
    "route": 
    {
        "routeName":"abc",
        "targetInstance":"a1-small",
        "destinationCIDR":"192.168.1.0/24",
        "description":"abc"
    }
}
```
##### Expected Return Body:
|Name|Description|Example Value|
|----|-----------|-------------|
|routeCreated|Whether the route was created or not.|true|

##### Example
```json
{
    "routeCreated":true
}
```

#### DELETE /v1/routes/{routeName}
Delete the requested route and returns the route just deleted.

|Name|Description|Example Value|
|----|-----------|-------------|
|routeDeleted|Whether the route was deleted or not.|true|
##### Example
Expected Return Value
```json
{
    "routeDeleted":true
}
```

