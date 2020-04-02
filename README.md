helm-cleanup-on-fail-flag
=========================

Testing the --cleanup-on-fail flag

Before you do anything:

1. Ensure you have Helm 2.16.5 installed
1. Ensure you have Helmfile 0.106.3 installed
1. Configure your client to point at the `default` namespace.

Scenario: new resources created during upgrade get rolled back, it is possible to upgrade from there
----------------------------------------------------------------------------------------------------

1. Install: 
   ```
   $ helmfile sync --args "--cleanup-on-fail"
   ...
   
   $ helm ls my-release
   NAME            REVISION        UPDATED                         STATUS          CHART                   APP VERSION     NAMESPACE
   my-release      1               Wed Apr  1 23:03:41 2020        DEPLOYED        local-chart-0.1.0       1.0             default  
   ```
1. Validate service, serviceaccount resources created and deploy not created:
   ```   
   $ kubectl get service,serviceaccount,deploy -l app.kubernetes.io/instance=my-release
   NAME                                    SECRETS   AGE
   serviceaccount/my-release-local-chart   1         22s
   
   # ^^ NOTE: no service, deploy listed
   ```
1. Upgrade, enabling service and deployment (we have a bad image name, so this will fail the deploy and thus fail the Helm upgrade after our 20 second configured timeout):
   ```
   $ helmfile sync --set deploymentEnabled=true --set serviceEnabled=true --args "--cleanup-on-fail"
   ... ~20 second delay ...
   
   UPGRADE FAILED
   Error: timed out waiting for the condition
   
   
   FAILED RELEASES:
   NAME
   my-release
   in ./helmfile.yaml: failed processing release my-release: helm exited with status 1:
     Error: UPGRADE FAILED: timed out waiting for the condition
   
   $ helm ls my-release
   NAME            REVISION        UPDATED                         STATUS  CHART                   APP VERSION     NAMESPACE
   my-release      2               Wed Apr  1 23:04:41 2020        FAILED  local-chart-0.1.0       1.0             default  

   ```
    1. You can try this running the kubectl get command in another shell, while Helm's waiting for the deploy to try to finish:
       ```
       $ kubectl get service,serviceaccount,deploy -l app.kubernetes.io/instance=my-release
       NAME                             TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
       service/my-release-local-chart   ClusterIP   10.110.35.75   <none>        80/TCP    1s
       
       NAME                                    SECRETS   AGE
       serviceaccount/my-release-local-chart   1         17m
       
       NAME                                           READY   UP-TO-DATE   AVAILABLE   AGE
       deployment.extensions/my-release-local-chart   0/1     1            0           1s

       ```
1. Validate the new resources do not exist:
   ```
   $ kubectl get service,serviceaccount,deploy -l app.kubernetes.io/instance=my-release
     
   NAME                                    SECRETS   AGE
   serviceaccount/my-release-local-chart   1         22s
   
   # ^^ NOTE: no service, deploy listed
   ```
1. Upgrade, enabling service but no deployment
   ```
   $ helmfile sync --set serviceEnabled=true --args "--cleanup-on-fail"
   
   $ helm ls my-release
   NAME            REVISION        UPDATED                         STATUS          CHART                   APP VERSION     NAMESPACE
   my-release      3               Wed Apr  1 23:06:23 2020        DEPLOYED        local-chart-0.1.0       1.0             default  
   ```
1. Validate the service now exists, without a deploy:
   ```
   $ kubectl get service,serviceaccount,deploy -l app.kubernetes.io/instance=my-release
   NAME                             TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
   service/my-release-local-chart   ClusterIP   10.105.231.55   <none>        80/TCP    22s
   
   NAME                                    SECRETS   AGE
   serviceaccount/my-release-local-chart   1         22s
   
   # ^^ NOTE: no deploy listed
   ```
