# Demo Script
Now we can progress to setting up the environment. 
We need to start by logging in to openShift / Kubernetes and setting up all the RHAI / skupper routers and gateways.


## Deploy RHAI to all Namespaces and create links


### Remote Site 1 - APAC
Open a new command line and run the setup script
```
$ . ./setup-apac.sh 
APAC: demo-env$ 

```

Login to the OpenShift cluster (you can get the login command from the web console)
```
APAC: demo-env$ oc login --token=sha256~#### --server=https://api.cluster-####.####.sandbox2077.####.com:6443
W0214 09:59:05.701750   40968 loader.go:221] Config not found: /home/admin/.kube/config-nearest-prime-apac
The server uses a certificate signed by an unknown authority.
You can bypass the certificate check, but any data you send to the server could be intercepted by others.
Use insecure connections? (y/n): y

Logged into "https://api.cluster-####.####.sandbox2077.####.com:6443" as "#####" using the token provided.

You have access to 66 projects, the list has been suppressed. You can list all projects with 'oc projects'

Using project "default".
W0214 09:59:14.280453   40968 loader.go:221] Config not found: /home/admin/.kube/config-nearest-prime-apac
W0214 09:59:14.280505   40968 loader.go:221] Config not found: /home/admin/.kube/config-nearest-prime-apac

```

Create a new namespace where you will deploy the nearest prime microservices.
```
APAC: demo-env$ oc new-project skupper-apac
Now using project "skupper-apac" on server "https://api.cluster-####.####.sandbox2077.####.com:6443".

You can add applications to this project with the 'new-app' command. For example, try:

    oc new-app rails-postgresql-example

to build a new example application in Ruby. Or use kubectl to deploy a simple Kubernetes application:

    kubectl create deployment hello-node --image=k8s.gcr.io/serve_hostname

```

Create your skupper router instance
```
skupper init --site-name APAC --console-auth=internal --console-user=admin --console-password=password
```

Check that skupper is properly installed
```
APAC: demo-env$ skupper status
Skupper is enabled for namespace "skupper-apac" with site name "APAC" in interior mode. It is not connected to any other sites. It has no exposed services.
The site console url is:  https://skupper-skupper-apac.apps.cluster-####.####.sandbox2077.####.com
The credentials for internal console-auth mode are held in secret: 'skupper-console-users'

```

### Local environment
Open a new command line and run the setup script
```
$ . ./setup-onprem.sh 
ONPREM: demo-env$ 

```

On this new window, login to the remote site 1 cluster.
```
ONPREM: demo-env$ oc login --token=sha256~####-HpGZ8 --server=https://api.cluster-####.####.sandbox2077.opentlc.com:6443 --namespace=skupper-apac
W0214 10:21:12.008744   42024 loader.go:221] Config not found: /home/admin/.kube/config-nearest-prime-onprem
The server uses a certificate signed by an unknown authority.
You can bypass the certificate check, but any data you send to the server could be intercepted by others.
Use insecure connections? (y/n): y

Logged into "https://api.cluster-####.####.sandbox2077.opentlc.com:6443" as "#####" using the token provided.

You have access to 67 projects, the list has been suppressed. You can list all projects with 'oc projects'

Using project "skupper-apac".
W0214 10:21:15.338495   42024 loader.go:221] Config not found: /home/admin/.kube/config-nearest-prime-onprem
W0214 10:21:15.338560   42024 loader.go:221] Config not found: /home/admin/.kube/config-nearest-prime-onprem

```

Initialize the local skupper gateway which will link the previously created router and this gateway instance.
```
ONPREM: demo-env$ skupper gateway init --type podman --namespace skupper-apac
Skupper gateway: 'dev02-rhel.localdomain-admin'. Use 'skupper gateway status' to get more information.
```

Observe if the gateway has started and the remote skupper router reports an established link from the gateway.
You can also check the links status via the skupper router web console which should be located at something like https://skupper-skupper-apac.apps.cluster-####.####.sandbox2077.opentlc.com/
```
ONPREM: demo-env$ skupper gateway status
Gateway Definition:
╰─ dev02-rhel.localdomain-admin type:podman version:2.2.1

ONPREM: demo-env$ skupper link status

Links created from this site:
-------------------------------
There are no links configured or active

Currently active links from other sites:
----------------------------------------
A link from the namespace  on site 3be3123b1b162219b5a238f13756c7c249d493d4(0f4ea697-9ae9-4da8-9e32-8693c7e5b39e) is active 

```


### Remote Site 2 - US
TO BE ADDEDD...





## Expose the DB service to the skupper network
Finally you need to add the on-premises database to the network. 
```
ONPREM: demo-env$ skupper gateway expose db 0.0.0.0 5432 --type podman
2023/02/14 10:42:06 CREATE io.skupper.router.tcpConnector dev02-rhel.localdomain-admin-egress-db:5432 map[address:db:5432 host:0.0.0.0 name:dev02-rhel.localdomain-admin-egress-db:5432 port:5432 siteId:0f4ea697-9ae9-4da8-9e32-8693c7e5b39e]

```

Once the gateway service has been created successfully, look at the services and pods on each cluster.
Observe that the db service has not associated pods, and the ip address is a local ip adress of the cluster. 
The RHAI router handles routing a service request over the RHAI network to the on premises local ip address and port.   

```
ONPREM: demo-env$ oc get svc,pods
NAME                           TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)               AGE
service/db                     ClusterIP   172.30.206.88    <none>        5432/TCP              38s
service/skupper                ClusterIP   172.30.157.143   <none>        8080/TCP,8081/TCP     33m
service/skupper-router         ClusterIP   172.30.111.79    <none>        55671/TCP,45671/TCP   33m
service/skupper-router-local   ClusterIP   172.30.167.189   <none>        5671/TCP              33m

NAME                                              READY   STATUS    RESTARTS   AGE
pod/skupper-router-777cc66994-mg49r               2/2     Running   0          33m
pod/skupper-service-controller-769c5fbd66-d89bq   1/1     Running   0          33m

```



## Deploy the Remote Applications
Let's now deploy the application that will consume the database and observe the services and pods that got created
Observe that there is no service for the nearestprime deployment at the remote sites.
```
APAC: yaml$ oc apply -f apac-nearestprime.yaml --namespace=skupper-apac
deployment.apps/nearestprime created

APAC: yaml$ oc get svc,pods
NAME                           TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)               AGE
service/db                     ClusterIP   172.30.206.88    <none>        5432/TCP              5m14s
service/skupper                ClusterIP   172.30.157.143   <none>        8080/TCP,8081/TCP     38m
service/skupper-router         ClusterIP   172.30.111.79    <none>        55671/TCP,45671/TCP   38m
service/skupper-router-local   ClusterIP   172.30.167.189   <none>        5671/TCP              38m

NAME                                              READY   STATUS    RESTARTS   AGE
pod/nearestprime-595cb57c8f-h4mlw                 1/1     Running   0          77s
pod/skupper-router-777cc66994-mg49r               2/2     Running   0          38m
pod/skupper-service-controller-769c5fbd66-d89bq   1/1     Running   0          38m

```


Now, at each of the remote clusters, we want to expose the nearestprime deployment so that it is reachable from outside the clusters
At each remote site type `skupper expose deployment nearestprime --port 8000 --protocol http` and press Enter.
Then Examine the services and pods on the on-premises cluster. Type `oc get svc,pods` and press Enter.

```
APAC: yaml$ skupper expose deployment nearestprime --port 8000 --protocol http
deployment nearestprime exposed as nearestprime

APAC: yaml$ oc get svc,pods
NAME                           TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)               AGE
service/db                     ClusterIP   172.30.206.88    <none>        5432/TCP              36m
service/nearestprime           ClusterIP   172.30.83.253    <none>        8000/TCP              59s
service/skupper                ClusterIP   172.30.157.143   <none>        8080/TCP,8081/TCP     69m
service/skupper-router         ClusterIP   172.30.111.79    <none>        55671/TCP,45671/TCP   69m
service/skupper-router-local   ClusterIP   172.30.167.189   <none>        5671/TCP              69m

NAME                                              READY   STATUS    RESTARTS   AGE
pod/nearestprime-595cb57c8f-h4mlw                 1/1     Running   0          32m
pod/skupper-router-777cc66994-mg49r               2/2     Running   0          69m
pod/skupper-service-controller-769c5fbd66-d89bq   1/1     Running   0          69m

```

From the above you can see that the remote db service is published but there are no pods for the service. 
RHAI will handle routing any request to the service over the RHAI network to the remote service and pod managed by the skupper gateway.

No matter how many remote sites expose the nearestprime service, there will only be one instance on-premises. 
This is because the service is an any-cast address. RHAI will handle routing the service request across the remote implementations.

**Note:** If you want to expose them individually then you would use unique names (such as nearestprime-ibmdc, nearestprime-awseur).

Repeat this for each remote site and observe that the service exists everywhere.



## Set up Gateway Forwarding
One last thing to set up is the gateway forwarding. 
This enables an on premises application to access the services published by the gateway.
```
ONPREM: demo-env$ skupper gateway forward nearestprime 8000
2023/02/14 11:22:16 CREATE io.skupper.router.httpListener dev02-rhel.localdomain-admin-ingress-nearestprime:8000 map[address:nearestprime:8000 name:dev02-rhel.localdomain-admin-ingress-nearestprime:8000 port:8000 protocolVersion:HTTP1 siteId:0f4ea697-9ae9-4da8-9e32-8693c7e5b39e]

```

## Review all of the connectivity details
The final step is to review the entire network configuration. 
You run this at any site, but for this demonstartion we will use remote site 1 (APAC). 
Type the command `skupper network status` and press Enter.
```
APAC: yaml$ skupper network status
Sites:
╰─ [local] 02d0f05 - APAC 
   URL: skupper-inter-router-skupper-apac.apps.cluster-l9jgb.l9jgb.sandbox2077.opentlc.com
   mode: interior
   name: APAC
   namespace: skupper-apac
   version: 1.2.2
   ╰─ Services:
      ├─ name: nearestprime
      │  address: nearestprime: 8000
      │  protocol: http
      │  ╰─ Targets:
      │     ╰─ name: nearestprime-595cb57c8f-h4mlw
      ╰─ name: db
         address: db: 5432
         protocol: tcp
```

# Installation and Configuration Finished

**Congratulations, you have now:**
1. Deployed the RHAI routers to all the application namespaces.
2. Created the application network.
3. Deployed the application
4. Advertised the application's endpoint to all namspaces in the RHAI network.

At this point in the story you have set up the RHAI network amd deployed all the applications. As you did this you highlighted how you could integrate with unroutable networks and with applications outside OpenShift.

You are now ready to test the configuration.


## Load Generator

The load generator is located in under the `components` directory.  

```
cd components/load-gen
```

Build and run the load generaor. Note: The first time you run this it will take a while to download all the maven repositories.

```
ONPREM$ ./mvnw quarkus:dev
Warning: JAVA_HOME environment variable is not set.
[INFO] Scanning for projects...
[INFO] 
[INFO] ------------------------< io.skupper:load-gen >-------------------------
[INFO] Building load-gen 1.0-SNAPSHOT
[INFO] --------------------------------[ jar ]---------------------------------
[INFO] 
[INFO] --- quarkus-maven-plugin:1.2.1.Final:dev (default-cli) @ load-gen ---
[INFO] Nothing to compile - all classes are up to date
OpenJDK 64-Bit Server VM warning: Options -Xverify:none and -noverify were deprecated in JDK 13 and will likely be removed in a future release.
Listening for transport dt_socket at address: 5005
2022-07-17 21:14:24,094 INFO  [io.quarkus] (main) load-gen 1.0-SNAPSHOT (running on Quarkus 1.2.1.Final) started in 0.737s. Listening on: http://0.0.0.0:8080
2022-07-17 21:14:24,105 INFO  [io.quarkus] (main) Profile dev activated. Live Coding activated.
2022-07-17 21:14:24,105 INFO  [io.quarkus] (main) Installed features: [cdi, rest-client, resteasy, resteasy-jsonb, vertx]
```

The load generator is now running and listening for instruction on `http://localhost:8080`. 
You will need to open a new terminal and use curl to control the load generator:

### Generate some load

The load generator is controlled by curl requests.
```
curl http://localhost:8080/set_load/0

```


In a new terminal, set the tool to generate 10 parallel requests.
This will continue to generate 10 parallel requests until you set the load to zero. **Note:** You should never need to increase the load above 10.
```
curl http://localhost:8080/set_load/10

```


To stop the load type the following command:
```
curl http://localhost:8080/set_load/0

```


Query the database contents and observe that the load balancing is not round robin. In ```pgadmin4```, query the database:

```
APAC: yaml$ psql -h dev02-rhel.localdomain -p 5432 -U demo -W -d demo-db
Password for user demo: 
psql (10.23, server 13.7)

demo-db=> select * from work where nearprime is not null;
```

# End of Demo Script

Return to [Main Index](../README.md)
