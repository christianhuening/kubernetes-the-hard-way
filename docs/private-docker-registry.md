# How to use a private Docker Registry

Docker stores keys for private registries in the $HOME/.dockercfg or $HOME/.docker/config.json file. If you put this in 
the $HOME of user root on a kubelet, then docker will use it.

Here are the recommended steps to configuring your nodes to use a private registry. In this example, run these on your 
desktop/laptop:

1. Run docker login [server] for each set of credentials you want to use. This updates ```$HOME/.docker/config.json.```
2. View ```$HOME/.docker/config.json``` in an editor to ensure it contains just the credentials you want to use.
3. Get a list of your nodes, for example:
    1. If you want the names: ```nodes=$(kubectl get nodes -o jsonpath='{range.items[*].metadata}{.name} {end}')```
    2. If you want to get the IPs: ```nodes=$(kubectl get nodes -o jsonpath='{range .items[*].status.addresses[?(@.type=="ExternalIP")]}{.address} {end}')```
4. Copy your local ```.docker/config.json``` to the home directory of root on each node.
    1. For example: ```for n in $nodes; do scp ~/.docker/config.json root@$n:/root/.docker/config.json; done```
5. Finally, if you use systemd, make sure that you explicitly set the $HOME variable for the root user, as systemd deletes 
all ENV variables for security reasons:
```shell
...
[Service]
Environment=HOME=/root
...
```

### Special hint if using Puppet for Distribution:
DO NOT ADD A NEWLINE TO THE END OF THE DOCKER CONFIG FILE!

    
    