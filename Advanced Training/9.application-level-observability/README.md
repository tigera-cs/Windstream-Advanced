# 9 Observability - Dashboards

In this lab, we will enable application layer data in flow logs.

Steps: \
9.1 Configure Felix for log data collection \
9.2 Configure ApplicationLayer CRD \
9.3 Select traffic for L7 log collection \
9.4 Test your configuration \
9.5 Check the Kibana Dashboard 


## 9.1 Configure Felix for log data collection

First we need to enable the Policy Sync API in Felix. 
To do this cluster-wide, modify the default FelixConfiguration to set the field policySyncPathPrefix to /var/run/nodeagent.

```
kubectl patch felixconfiguration default --type='merge' -p '{"spec":{"policySyncPathPrefix":"/var/run/nodeagent"}}'
```

## 9.2 Configure ApplicationLayer CRD

In this step, you configure ApplicationLayer resource to gather the L7 logs.

Create the ApplicationLayer resource named, tigera-secure: 
Ensure that collectLogs fields is set to Enabled.

```
kubectl apply -f -<<EOF
apiVersion: operator.tigera.io/v1
kind: ApplicationLayer
metadata:
  name: tigera-secure
spec:
  logCollection:
    collectLogs: Enabled
    logIntervalSeconds: 5
    logRequestsPerInterval: -1
EOF

```


## 9.3. Enable sidecar for targeted application pod

Next enable yaobank to use Envoy. This will give us layer 7 visibility into micro-services communication. Notice the deployment of 2 sidecars alongside the app container, envoy and l7-collector. The function of l7-collector is to integrate with Envoy and extract flowlog data. Also notice that one of the steps includes enabling flowlogs for hostendpoints which is disabled by default. 

```
kubectl patch felixconfiguration default --type='merge' -p '{"spec":{"policySyncPathPrefix":"/var/run/nodeagent"}}'
```
```
kubectl get deployment -n yaobank
```
```
NAME       READY   UP-TO-DATE   AVAILABLE   AGE
customer   1/1     1            1           2d16h
database   1/1     1            1           2d16h
summary    2/2     2            2           2d16h
```
```
kubectl patch deployment customer -n yaobank --patch "$(cat patch-envoy.yaml)"
```
```
kubectl patch deployment database -n yaobank --patch "$(cat patch-envoy.yaml)"
```
```
kubectl patch deployment summary -n yaobank --patch "$(cat patch-envoy.yaml)"
```
```
kubectl patch felixconfiguration default -p '{"spec":{"flowLogsEnableHostEndpoint":true}}'
```

Wait until the yaobank application is redeployed with the additional containers running as sidecars inside the pod 

```
watch kubectl get pod -n yaobank
```
```
NAME                        READY   STATUS        RESTARTS   AGE
customer-78647ff759-25p2z   3/3     Running       0          71s
database-78f6d54974-bjwcv   3/3     Running       0          66s
summary-77f9d5f98c-dbjkw    3/3     Running       0          54s
summary-77f9d5f98c-fhdmr    3/3     Running       0          57s
```

## 9.4. Verify The Application Level Dashboard

We need to retrieve the password for kibana you wrote down in previous labs. If you forgot to do so then execute the command below:

```
kubectl -n tigera-elasticsearch get secret tigera-secure-es-elastic-user -o go-template='{{.data.elastic | base64decode}}' && echo
```

Before Checking the Dashboards, let's generate some http traffic for our yaobank application:

```
touch /tmp/runscript ; while [ -f /tmp/runscript ] ; do curl -si $(kubectl get svc -n yaobank | grep customer | awk {'print $3'}) | head -1 ; sleep 2 ; done &
```

You will start seeing some HTTP responses displayed. This will keep running until we remove a file, but for now leave it running.

Access kibana from the left tool bar with the icon ![kibana](img/9.1-kib-icon.png). The default username is `elastic`.

Now select the Dashboards as indicated in the figure below, and then L7 HTTP Dashboard:

![menu](img/9.2-kib-menu.png)

There you will see the Application level Dashboard for the yaobank application

![l7dashboard](img/9.3-dnsdashboard.png)

## 9.5 Cleanup

In the terminal where the script is running, execute the following command to stop the script (copy and paste to your terminal):

```
rm /tmp/runscript
```

After hitting the `Enter` key a couple of times, you should see a message as the one below:

```
[1]+  Done                    while [ -f /tmp/runscript ]; do
    curl -si $(kubectl get svc -n yaobank | grep customer | awk {'print $3'}) | head -1; sleep 2;
done
```

