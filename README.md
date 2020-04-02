helm-cleanup-on-fail-flag
=========================

Testing the --cleanup-on-fail flag. This is a flag for `helm upgrade`, that 

Rather than specify this on a project-by-project basis in helmfile.yaml, I think we'd specify it in our pipeline's 
helmfile command with `--args "--cleanup-on-fail"`.

The gist of our testing revealed that:
1. It works great for allowing devs to redeploy after a failed _upgrade_.
2. It doesn't do anything for us when the first deploy (a helm _install_) fails. Incidentally, `--atomic` seemed to 
help out there, but I don't know what other side effects that might have.

Before you do anything:

1. Ensure you have Helm 2.16.5 and Helmfile 0.106.3 installed
   ```
   $ helm version
   Client: &version.Version{SemVer:"v2.16.5", GitCommit:"89bd14c1541fa93a09492010030fd3699ca65a97", GitTreeState:"clean"}
   Server: &version.Version{SemVer:"v2.16.5", GitCommit:"89bd14c1541fa93a09492010030fd3699ca65a97", GitTreeState:"clean"}

   $ helmfile --version
   helmfile version v0.106.3
   ```
1. Configure your client to point at the `default` namespace (or any other namespace you'd like).

Scenario 1: Failed Upgrade
------------------------

New resources created during failed upgrade get rolled back, it is possible to upgrade from there:

1. Install: 
   ```
   $ helmfile sync --args "--cleanup-on-fail"
   ...
   
   $ helm ls my-release
   NAME            REVISION        UPDATED                         STATUS          CHART                   APP VERSION     NAMESPACE
   my-release      1               Wed Apr  1 23:03:41 2020        DEPLOYED        local-chart-0.1.0       1.0             default  
   ```
1. Validate serviceaccount resource created and deploy not created:
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
    1. You can try running the kubectl get command in another shell, while Helm's waiting for the deploy to try to finish:
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
   serviceaccount/my-release-local-chart   1         100s
   
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


Scenario 2: Failed Install
--------------------------

All resources created during install get left - no rollbacks. Helm delete required to run `helmfile sync` successfully:

1. Ensure old release gets deleted:
   ```
   $ helm delete --purge my-release
   release "my-release" deleted

   $ helm ls -a my-release
   <no results>  
   ```

1. Install with deployment and service enabled: 
   ```
   $ helmfile sync --set deploymentEnabled=true --set serviceEnabled=true --args "--cleanup-on-fail"
   ...
   
   $ helm ls my-release
   NAME            REVISION        UPDATED                         STATUS  CHART                   APP VERSION     NAMESPACE
   my-release      1               Wed Apr  1 23:25:01 2020        FAILED  local-chart-0.1.0       1.0             default    
   ```
1. Validate all resources are created ☹️:
   ```   
   $ kubectl get service,serviceaccount,deploy -l app.kubernetes.io/instance=my-release
   NAME                             TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
   service/my-release-local-chart   ClusterIP   10.103.181.15   <none>        80/TCP    75s
   
   NAME                                    SECRETS   AGE
   serviceaccount/my-release-local-chart   1         75s
   
   NAME                                           READY   UP-TO-DATE   AVAILABLE   AGE
   deployment.extensions/my-release-local-chart   0/1     1            0           75s

   ```
1. Try to upgrade, enabling service but no deployment
   ```
   $ helmfile sync --set serviceEnabled=true --args "--cleanup-on-fail"
   Building dependency release=my-release, chart=local-chart
   No requirements found in local-chart/charts.
   
   Affected releases are:
     my-release (./local-chart) UPDATED
   
   Upgrading release=my-release, chart=local-chart
   UPGRADE FAILED
   Error: "my-release" has no deployed releases
   
   
   FAILED RELEASES:
   NAME
   my-release
   in ./helmfile.yaml: failed processing release my-release: helm exited with status 1:
     Error: UPGRADE FAILED: "my-release" has no deployed releases

   
   $ helm ls my-release
   NAME            REVISION        UPDATED                         STATUS          CHART                   APP VERSION     NAMESPACE
   my-release      1               Wed Apr  1 23:25:01 2020        FAILED          local-chart-0.1.0       1.0             default    
   ```
1. Purge the Helm release and validate the resources are removed:
   ```
   $ helm delete --purge my-release
   release "my-release" deleted

   $ kubectl get service,serviceaccount,deploy -l app.kubernetes.io/instance=my-release
   No resources found in default namespace.
   ```
1. Try install again, but without deployment enabled:
   ```
   $ helmfile sync --set serviceEnabled=true --args "--cleanup-on-fail"
   ...
   UPDATED RELEASES:
   NAME         CHART           VERSION
   my-release   ./local-chart          

   $ helm ls my-release                                                    
   NAME            REVISION        UPDATED                         STATUS          CHART                   APP VERSION     NAMESPACE
   my-release      1               Wed Apr  1 23:30:06 2020        DEPLOYED        local-chart-0.1.0       1.0             default

   $ kubectl get service,serviceaccount,deploy -l app.kubernetes.io/instance=my-release
   NAME                             TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
   service/my-release-local-chart   ClusterIP   10.105.231.55   <none>        80/TCP    22s

   NAME                                    SECRETS   AGE
   serviceaccount/my-release-local-chart   1         22s

   # ^^ NOTE: no deploy listed
   ```
   

Scenario 3: Failed Install With Atomic
--------------------------------------

All resources created during install get removed with `--atomic`, and users are able to carry on with future deploys without issue

1. Ensure old release gets deleted:
   ```
   $ helm delete --purge my-release
   release "my-release" deleted

   $ helm ls -a my-release
   <no results>  
   ```
1. Install with deployment and service enabled: 
   ```
   $ helmfile sync --set deploymentEnabled=true --set serviceEnabled=true --args "--cleanup-on-fail --atomic"
   ... <failure> ...
   INSTALL FAILED
   PURGING CHART
   Error: release my-release failed: timed out waiting for the condition
   Successfully purged a chart!
   ...
   
   $ helm ls -a my-release
   <no results>  
   ```
1. Validate all resources were deleted 😀️:
   ```   
   $ kubectl get service,serviceaccount,deploy -l app.kubernetes.io/instance=my-release
   No resources found in default namespace.
   ```
1. Try to install again:
   ```
   $ helmfile sync --args "--cleanup-on-fail --atomic"
   ... <success> ...
   
   $ helm ls my-release
   NAME            REVISION        UPDATED                         STATUS          CHART                   APP VERSION     NAMESPACE
   my-release      1               Thu Apr  2 14:03:10 2020        DEPLOYED        local-chart-0.1.0       1.0             default  
   ```
1. Try to upgrade again:
   ```
   $ helmfile sync --set serviceEnabled=true --args "--cleanup-on-fail --atomic"
   ... <success> ...
   
   $ helm ls my-release
   NAME            REVISION        UPDATED                         STATUS          CHART                   APP VERSION     NAMESPACE
   my-release      2               Thu Apr  2 14:13:36 2020        DEPLOYED        local-chart-0.1.0       1.0             default  
   ```