# SpoddyCoder Kube Tools

`scube-tools` is a suite of simple but useful Kubernetes helper tools.

These should all be considered beta, only my setups / systems have been tested so far. PR's welcome!


## Requirements / Assumptions

* `kubectl` installed
* `~/.kube/config` exists
* All commands use the `current-context` defined in `~/.kube/config`
    * If you manage multiple clusters, check you are using the right context!
* You have correct permissions for the namespaces + resources you want to query / modify


## Usage + Optional Installation

The tools can be run directly, so just copy wherever you need and run, eg: 

`./find-pod-and bash my-app`

If you want them available system wide, do something like this...

```
sudo cp find-pod-and /usr/local/bin/scube-find-pod-and
sudo cp launch-dashboard /usr/local/bin/scube-launch-dashboard
```

Then run the commands as you would any other on your system (assuming `/usr/local/bin/` is in your `$PATH`)...

```
scube-find-pod-and tail my-app

scube-launch-dashboard
```


## `find-pod-and`

Because who hasn't had enough of using `kubectl` to query running pods?... 
Then copy-pasting auto-generated deployment id's into a second `kubectl` command... which you often forget the exact syntax for...
Only to find out it has multiple containers... which you need to `kubectl` query...
And then rerun th... FML!

### Usage

`./find-pod-and [command] [searchterm] [namespace]`

Search running pods by `[searchterm]` and list matches, select one from the list and do `[command]`.
If a pod has multiple containers, each is shown. eg: 

`./find-pod-and describe my-app -A`

* `[command]`:
    * `bash`        - /bin/bash into container
    * `sh`          - /bin/sh into container
    * `log`         - show container logs (`logs` also works)
    * `tail`        - tail container logs
    * `describe`    - describe the pod
    * `help`        - show usage info (`--help` also works)

* `[searchterm]`:
    * `somestring`  - search for pods matching somestring (also searches namespaces)
    * `-`           - match-all, return all pods in selected namespace

* `[namespace]` (optional):
    * no value      - search the current namespace in the current context
    * `-A`          - search across all namespaces
    * `mynamespace` - search in mynamespace

### Examples

Shell into a container...

```
scube-find-pod-and bash example

Searching for containers matching 'example' in namespace: default

1) default: example-app-1-deployment-1d23456789-9876 -> nginx

Select container number (or Enter to exit): 1

root@example-app-1-deployment-1d23456789-9876:/# service nginx status
nginx is running.
```

Tail application logs...

```
scube-find-pod-and tail app-1 my-app

Searching for containers matching 'app-1' in namespace: my-app

1) my-app: app-1-mysql-deployment-1298379231-1298 -> mysql
2) my-app: example-app-1-deployment-1d23456789-9876 -> varnish
3) my-app: example-app-1-deployment-1d23456789-9876 -> nginx

Select container number (or Enter to exit): 3

1000.99.99.99 - - [12/Jun/2022:22:31:52 +0000] "GET / HTTP/1.1" 304 0 "-" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/102.0.0.0 Safari/537.36" "999.22.33.44"
2022/06/12 22:31:59 [error] 30#30: *1 open() "/usr/share/nginx/html/howaboutthis" failed (2: No such file or directory), client: 999.22.33.44, server: localhost, request: "GET /howaboutthis HTTP/1.1", host: "test.example.com"
1000.99.99.99 - - [12/Jun/2022:22:31:59 +0000] "GET /howaboutthis HTTP/1.1" 404 555 "-" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/102.0.0.0 Safari/537.36" "999.22.33.44"
```

Get logs from kube-system core components...

```
scube-find-pod-and log kube -A

Searching for containers matching 'kube' across all namespaces

1) kube-system: coredns-123454567-99999 -> coredns
2) kube-system: coredns-786543211-99998 -> coredns
3) kube-system: etcd-k-master -> etcd
4) kube-system: kube-apiserver-k-master -> kube-apiserver
5) kube-system: kube-controller-manager-k-master -> kube-controller-manager
6) kube-system: kube-flannel-ds-12345 -> kube-flannel
7) kube-system: kube-proxy-4321 -> kube-proxy
8) kube-system: kube-scheduler-k-master -> kube-scheduler

Select container number (or Enter to exit): 2

.:53
[INFO] plugin/reload: Running configuration MD5 = f4304ahahahahahahahacf234m439
CoreDNS-55.5.5
linux/amd64, go5.55.5, 555555
```


## `launch-dashboard`

Frugal dashboardery! Why have pods running when no-one's looking? 
This makes accessing the kubernetes dashboard as simple as it can possibly get...

```
scube-launch-dashboard
```

### Scube is a good boi! Does all this for you...

* Get a new token for dashboard login and copy it to clipboard
    * Mac only - will print to STDOUT if `pbcopy` not found
    * Service account for the token generation is configurable
* Redeploy the dashboard pods
    * Scale is configurable
* Start the `kubectl proxy`
* Open the kube dashbaord url with default query `workloads?all-namepsaces=&namespace=_all`
    * Browser app is configurable
    * Kube proxy dash URL is configurable
    * Default query is configurable
* Listen for `CTRL+C` quit and cleans up
    * Stop the `kubectl proxy`
    * Undeploy the dashboard pods
* If the `kubectl proxy` is already running...
    * Just copy new token to the clipboard (or print to STDOUT)
    * Useful if your dashboard browser session times out

### Kubernetes Dashboard Service Acccounts

By default this tool generates a token for the `k8s-dashboard-admin` service account. 
You can generate a token for a different service account by using the configuration options.

Example yaml to create a `k8s-dashboard-admin` service account with a `cluster-admin` rolebinding...

```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: k8s-dashboard-admin
  namespace: kubernetes-dashboard
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: k8s-dashboard-admin-crb
subjects:
- kind: ServiceAccount
  name: k8s-dashboard-admin
  namespace: kubernetes-dashboard
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
```

Warning: ensure you understand the consequences, this gives the `k8s-dashboard-admin` service account full access to your cluster.

### Configuration

Available settings;

* service name for token generation
    * default: `k8s-dashboard-admin`
* browser app
    * default: `/Applications/Google Chrome.app` (mac chrome)
* proxy dashboard url
    * default: `http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/`
* browser query 
    * default: `"workloads?all-namepsaces=&namespace=_all`
* `kubernetes-dashboard` pods deployment scale
    * default: `1`

The tool looks for config files at different locations, they are applied in the following order, each overriding values in the previous...

* Default config: tool source code, eg: `/usr/local/bin/scube-launch-dashboard`
* System config: `/etc/scube-tools/launch-dashboard.conf`
* User config: `~/.scube-tools/launch-dashboard.conf`
* Directory config: `.launch-dashboard.conf` or `.scube-launch-dashboard.conf`

See the tool for the user config variable names and their descriptions. 


## Contributing

Contributions very welcome! Something wrong? Could be better? Additional use-cases? Please post in the issues forum or submit a PR. See the tool source code for outstanding ideas / todo's.