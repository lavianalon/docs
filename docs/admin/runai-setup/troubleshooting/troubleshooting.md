
# Troubleshooting Run:ai 

## Dashboard Issues

??? "No Metrics are showing on Dashboard"

    __Symptom__: No metrics are not showing on dashboards at `https://<company-name>.run.ai/dashboards/now`

    __Typical root causes:__

    * Firewall-related issues.
    * Internal clock is not synced.
    * Prometheus pods are not running.

    __Firewall issues__

    Add verbosity to Prometheus as describe [here](diagnostics.md).Verify that there are no errors. If there are connectivity-related errors you may need to:

    * Check your firewall for outbound connections. See the required permitted URL list in [Network requirements](../cluster-setup/cluster-prerequisites.md#network-requirements.md).
    * If you need to set up an internet proxy or certificate, please contact Run:ai customer support. 


    __Machine Clocks are not synced__

    Run: `date` on cluster nodes and verify that date/time is correct.  If not,

    * Set the Linux time service (NTP).
    * Restart Run:ai services. Depending on the previous time gap between servers, you may need to reinstall the Run:ai cluster


    __Prometheus pods are not running__

    Run: `kubectl get pods -n monitoring -o wide`

    * Verify that all pods are running.
    * The default Prometheus installation is not built for high availability. If a node is down, the Prometheus pod may not recover by itself unless manually deleted. Delete the pod to see it start on a different node and consider adding a second replica to Prometheus.

??? "GPU Relates metrics not showing"
    __Symptom:__ GPU-related metrics such as `GPU Nodes` and `Total GPUs` are showing zero but other metrics, such as `Cluster load` are shown.

    __Root cause:__ An installation issue related to the NVIDIA stack.

    __Resolution:__ 

    Need to run through the NVIDIA stack and find the issue. The current NVIDIA stack looks as follows:

    1. NVIDIA Drivers (at the OS level, on every node)
    2. NVIDIA Docker (extension to Docker, on every node)
    3. Kubernetes Node feature discovery (mark node properties)
    4. NVIDIA GPU Feature discovery (mark nodes as “having GPUs”)
    5. NVIDIA Device plug-in (Exposes GPUs to Kubernetes)
    6. NVIDIA DCGM Exporter (Exposes metrics from GPUs in Kubernetes)

    Run:ai requires the installation of the NVIDIA GPU Operator which installs the entire stack above. However, there are two alternative methods for using the operator:

    * Use the default operator values to install 1 through 6.
    * If  NVIDIA Drivers (#1 above) are already installed on __all__ nodes, use the operator with a flag that disables drivers install. 
    
    For more information see [Cluster prerequisites](../cluster-setup/cluster-prerequisites.md#nvidia).

    __NVIDIA GPU Operator__

    Run: `kubectl get pods -n gpu-operator | grep nvidia` and verify that all pods are running.

    __Node and GPU feature discovery__
    
    _Kubernetes Node feature discovery_ identifies and annotates nodes. _NVIDIA GPU Feature Discovery_ identifies and annotates nodes with GPU properties. See that: 
    
    * All such pods are up, 
    * The GPU feature discovery pod is available for every node with a GPU.
    * And finally, when describing nodes, they show an active `gpu/nvidia` resource.

    __NVIDIA Drivers__
    
    * If NVIDIA drivers have been installed on the nodes themselves, ssh into each node and run `nvidia-smi`. Run `sudo systemctl status docker` and verify that docker is running. Run `nvidia-docker` and verify that it is installed and working.  Linux software upgrades may require a node restart.
    * If NVIDIA drivers are installed by the Operator, verify that the NVIDIA driver daemonset has created a pod for each node and that all nodes are running. Review the logs of all such pods. A typical problem may be the driver version which is too advanced for the GPU hardware. You can set the driver version via operator flags. 


    __NVIDIA DCGM Exporter__

    View the logs of the DCGM exporter and verify that there are no errors prohibiting the sending of metrics. 



??? "Allocation-related metrics not showing"
    __Symptom:__ GPU Allocation-related metrics such as `Allocated GPUs` are showing zero but other metrics, such as `Cluster load` are shown.

    __Root cause:__ The origin of such metrics is the scheduler. 

    __Resolution:__

    * Run: `kubectl get pods -n runai | grep scheduler`. Verify that the pod is running.
    * Review the scheduler logs and look for errors. If such errors exist, contact Run:ai customer support. 

??? "All metrics are showing ""No Data"""
    __Symptom:__ all data on all dashboards is showing the text "No Data".

    __Root cause:__ Internal issue with metrics infrastructure.

    __Resolution:__ Please contact Run:ai customer support.

## Log in and Authentication Issues

??? "After a successful login, you are redirected to the same login page"
    For a self-hosted installation, check Linux clock synchronization as described above. Use the [Run:ai pre-install script](../cluster-setup/cluster-prerequisites.md#pre-install-script) to test this automatically. 

??? "Single-sign-on issues"
    For single-sign-on issues, see the troubleshooting section in the [single-sign-on](../authentication/sso.md#troubleshooting) configuration document. 

## Issues when Submitting Jobs from User Interface

??? "New Job button is greyed out"
    __Symptom:__ The `New Job` button on the top right of the Job list is grayed out.
    
    __Root Cause:__ This can happen due to multiple configuration issues. 
    
    * Open Chrome developer tools and refresh the screen.
    * Under `Network` locate a network call error. Search for the HTTP error code.

    __Resolution for 401 HTTP Error__

    * The Cluster certificate provided as part of the installation is valid and trusted (not self-signed).
    * [Researcher Authentication](../authentication/researcher-authentication.md) has not been properly configured. Try running `runai login` from the Command-line interface. Alternatively, run: `kubectl get pods -n kube-system`, identify the api-server pod and review its logs. 

    __Resolution for 403 HTTP Error__

    Run `kubectl get pods -n runai` identify the `agent` pod, see that it's running, and review its logs.

??? "New Job button is not showing"
    __Symptom:__ The `New Job` button on the top right of the Job list does not show.

    __Root Causes:__ (multiple)

    * You do not have `Researcher` or `Research Manager` permissions.
    * Cluster version is 2.3 or lower.
    * Under `Settings | General`, verify that `Unified UI` is on.


??? "Submit form is distorted"
    __Symptom:__ Submit form is showing vertical lines.

    __Root Cause:__ The control plane does not know the cluster URL.

    Using the Run:ai user interface, go to the Clusters list. See that there is no cluster URL next to your cluster.

    __Resolution:__ Cluster must be re-installed. 


??? "Submit form is not showing after pressing Create Job button"
    __Symptom:__ (SaaS only) Submit form now showing and under Chrome developer tools you see that all network calls with `/workload` return an error. 

    __Root Cause:__ Multiple network-related issues.

    __Resolution:__ 

    * Incorrect cluster IP
    * Cluster certificate has not been created. 
        * Run: `kubectl get certificate -n runai`. Verify that all 3 entries are of status `Ready`.
        * Run: `kubectl get pods -n cert-manager and verify that all pods are Running.


??? "Submit form does not show the list of Projects"
    __Symptom:__ When connected with Single-sign-on, in the Submit form, the list of Projects is empty.

    __Root Cause:__  SSO is on and researcher authentication is not properly configured as such.

    __Resolution:__ Verify API Server settings as described in [Researcher Authentication configuration](../authentication/researcher-authentication.md).



## Networking Issues

??? "'admission controller' connectivity issue"
    __Symptoms:__

    * Pods are failing with 'admission controller' connectivity errors.
    * The command-line `runai submit` fails with an 'admission controller' connectivity error.
    * Agent or cluster sync pods are crashing in self-hosted installation.

    __Root cause:__ Connectivity issues between different nodes in the cluster

    __Resolution:__

    * Run the [preinstall script](../cluster-setup/cluster-prerequisites.md#pre-install-script) and search for networking errors.
    * Run `kubectl get nodes`. Check that all nodes are ready and connected
    * Run `kubectl get pods -o wide -A` to see which pods are Pending or in Error and which nodes they belong to. 
    * See if pods from different nodes have trouble communicating with each other.
    * Advanced: Run `kubectl exec <pod-name> -it /bin/sh` from a pod in one node and ping a pod from another. 


??? "Projects are not syncing"
    __Symptom:__ Create a Project on the Run:ai user interface, then run: `runai list projects`. The new Project does __not__ appear.

    __Root cause:__ The Run:ai _agent_ is not syncing properly. This may be due to firewall issues. 
    
    __Resolution__
    
    * Run: `runai pods -n runai | grep agent`. See that the agent is in _Running_ state. Select the agent's full name and run: `kubectl logs -n runai runai-agent-<id>`
    * Verify that there are no errors. If there are connectivity-related errors you may need to check your firewall for outbound connections. See the required permitted URL list in [Network requirements](../cluster-setup/cluster-prerequisites.md#network-requirements). 
    * If you need to set up an internet proxy or certificate, please contact Run:ai customer support. 


## Job-related Issues

??? "Jobs fail with ContainerCannotRun status "
    __Symptom:__ When running `runai list jobs`, your Job has a status of `ContainerCannotRun`.

    __Root Cause:__ The issue may be caused due to an unattended upgrade of the NVIDIA driver.

    To verify, run `runai describe job <job-name>`, and search for an error `driver/library version mismatch`.

    __Resolution:__ Reboot the node on which the Job attempted to run.

    Going forward, we recommend blacklisting NVIDIA driver from unattended-upgrades.  
    You can do that by editing `/etc/apt/apt.conf.d/50unattended-upgrades`, and adding `nvidia-driver-` to the `Unattended-Upgrade::Package-Blacklist` section.  
    It should look something like that:

    ``` CONF
    Unattended-Upgrade::Package-Blacklist {
        // The following matches all packages starting with linux-
        //  "linux-";
        "nvidia-driver-";
    ```

## Kubernetes-specific Issues

??? "Cluster Installation failed on Rancher-based Kubernetes (RKE)"
    __Symptom:__ Cluster is not installed. When running `kubectl get pods -n runai` you see that pod `init-ca` has not started.

    __Resolution:__

    During initialization, Run:ai creates a Certificate Signing Request (CSR) which needs to be approved by the cluster's Certificate Authority (CA). In RKE, this is not enabled by default, and the paths to your Certificate Authority's keypair must be referenced manually by adding the following parameters inside your cluster.yml file, under kube-controller:

    ``` YAML
    services:
    kube-controller:
        extra_args:
        cluster-signing-cert-file: /etc/kubernetes/ssl/kube-ca.pem
        cluster-signing-key-file: /etc/kubernetes/ssl/kube-ca-key.pem
    ```

    For further information see [here](https://github.com/rancher/rancher/issues/14674){target=_blank}.