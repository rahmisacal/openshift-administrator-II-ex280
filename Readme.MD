# Secret Extract
-	oc extract secret/db-secrets-55cbgc8c6m --to=-

# view parameters of template
-	oc process --parameters mysql-persistent -n openshift

# using template
-	oc new-app --template=cache-service -p APPLICATION_USER=my-user

# Helm
-	helm repo list
-	helm repo add do280-repo http://helm.ocp4.example.com/charts
-	helm search repo --versions
-	helm show values do280-repo/etherpad --version 0.0.6
-	helm install example-app do280-repo/etherpad -f values.yaml --version 0.0.6
-	helm list
-	helm search repo --versions
-	helm upgrade example-app do280-repo/etherpad -f values.yaml --version 0.0.7

# Authentication
-	yum install htpasswd-tools
-	htpasswd -c -B -b ~/DO280/labs/auth-providers/htpasswd new_admin redhat
-	htpasswd -b ~/DO280/labs/auth-providers/htpasswd new_developer developer
-	oc create secret generic localusers --from-file htpasswd=~/DO280/labs/auth-providers/htpasswd -n openshift-config
-	oc adm policy add-cluster-role-to-user cluster-admin new_admin
-	oc get oauth cluster -o yaml > ~/DO280/labs/auth-providers/oauth.yaml

		apiVersion: config.openshift.io/v1
		kind: OAuth
		...output omitted...
		spec:
		  identityProviders:
		  - ldap:
		...output omitted...
			type: LDAP
		  - htpasswd:
			  fileData:
				name: localusers
			mappingMethod: claim
			name: myusers
			type: HTPasswd

-	oc replace -f ~/DO280/labs/auth-providers/oauth.yaml
-	oc extract secret/localusers -n openshift-config --to ~/DO280/labs/auth-providers/ --confirm
-	htpasswd -b ~/DO280/labs/auth-providers/htpasswd manager redhat
-	oc set data secret/localusers --from-file htpasswd=~/DO280/labs/auth-providers/htpasswd -n openshift-config

# RBAC
-	oc describe clusterrolebindings self-provisioners
-	oc adm policy remove-cluster-role-from-group self-provisioner system:authenticated:oauth
-	oc policy add-role-to-user admin leader
-	oc adm groups new dev-group
-	oc adm groups add-users dev-group developer
-	oc policy add-role-to-group edit dev-group

# Project External Traffic With TLS
-	oc expose svc todo-http --hostname todo-http.apps.ocp4.example.com
-	oc create route edge todo-https --service todo-http --hostname todo-https.apps.ocp4.example.com

-	openssl genrsa -out training.key 4096
-	openssl req -new -key training.key -out training.csr -subj "/C=US/ST=North Carolina/L=Raleigh/O=Red Hat/CN=todo-https.apps.ocp4.example.com"
-	openssl x509 -req -in training.csr -passin file:passphrase.txt -CA training-CA.pem -CAkey training-CA.key -CAcreateserial -out training.crt -days 1825 -sha256 -extfile training.ext

-	oc create secret tls todo-certs --cert certs/training.crt --key certs/training.key
-	oc create route passthrough todo-https --service todo-https --port 8443 --hostname todo-https.apps.ocp4.example.com

# NetworkPolicy

        kind: NetworkPolicy
        apiVersion: networking.k8s.io/v1
        metadata:
          name: network-1-policy
          namespace: network-1
        spec:
          podSelector:  
            matchLabels:
              deployment: product-catalog
          ingress:  
          - from:  
            - namespaceSelector:
                matchLabels:
                  network: network-2
            podSelector:
                matchLabels:
                  role: qa
            ports:  
            - port: 8080
              protocol: TCP
	  
	  
        kind: NetworkPolicy
        apiVersion: networking.k8s.io/v1
        metadata:
          name: network-2-policy
          namespace: network-2
        spec:
          podSelector: {}
          ingress:
          - from:
            - namespaceSelector:
                matchLabels:
                  network: network-1
		  

        kind: NetworkPolicy
        apiVersion: networking.k8s.io/v1
        metadata:
          name: default-deny
        spec:
          podSelector: {}
  

        ---
        apiVersion: networking.k8s.io/v1
        kind: NetworkPolicy
        metadata:
          name: allow-from-openshift-ingress
        spec:
          podSelector: {}
          ingress:
          - from:
            - namespaceSelector:
                matchLabels:
                  network.openshift.io/policy-group: ingress
        ---
        apiVersion: networking.k8s.io/v1
        kind: NetworkPolicy
        metadata:
          name: allow-from-openshift-monitoring
        spec:
          podSelector: {}
          ingress:
          - from:
            - namespaceSelector:
                matchLabels:
                  network.openshift.io/policy-group: monitoring
		  
		  
# Internal Traffic with TLS

-	oc annotate service hello service.beta.openshift.io/serving-cert-secret-name=hello-secret

        spec:
        template:
            spec:
            containers:
                - name: hello
                volumeMounts:
                    - name: hello-volume 
                      mountPath: /etc/pki/nginx/ 
            volumes:
                - name: hello-volume 
                secret:
                    defaultMode: 420 
                    secretName: hello-secret 
                    items:
                    - key: tls.crt 
                        path: server.crt 
                    - key: tls.key 
                        path: private/server.key 
				
-	oc create configmap ca-bundle				
-	oc annotate configmap ca-bundle service.beta.openshift.io/inject-cabundle=true

        apiVersion: v1
        kind: Pod
        metadata:
          name: client
        spec:
          containers:
            - name: client
            image: registry.ocp4.example.com:8443/redhattraining/hello-world-nginx
            resources: {}
            volumeMounts:
                - mountPath: /etc/pki/ca-trust/extracted/pem
                name: trusted-ca
          volumes:
            - configMap:
                defaultMode: 420
                name: ca-bundle
                items:
                - key: service-ca.crt
                    path: tls-ca-bundle.pem
            name: trusted-ca

-	oc patch deployment hello --patch-file ~/volume-mount.yaml
-	oc exec no-ca-bundle -- openssl s_client -connect server.network-svccerts.svc:443
-	oc exec client -- curl -s https://server.network-svccerts.svc
-	oc exec client -- openssl s_client -connect server.network-svccerts.svc:443

# LoadBalancer

-	oc expose deployment/virtual-rtsp-1 --type=LoadBalancer --target-port=8554
-	multus
        
        apiVersion: apps/v1
        kind: Deployment
        metadata:
        name: example
        namespace: example
        spec:
        selector:
            matchLabels:
            app: example
            name: example
        template:
            metadata:
            annotations:
                k8s.v1.cni.cncf.io/networks: example
            labels:
                app: example
                name: example
            spec:
        ...output omitted...

-	oc debug node/master01 -- chroot /host ip addr

# ResourcesQouta

-	oc create resourcequota example --hard=count/pods=1
-	oc create resourcequota example --hard=count/deployment=1
-	oc get event --sort-by .metadata.creationTimestamp
-	oc create clusterresourcequota example --project-label-selector=group=dev --hard=requests.cpu=10
-	oc set resources deployment test --requests=cpu=1
-	oc create quota two-cpus --hard=requests.cpu=2
-	oc set resources deployment test --requests=cpu=1
-	oc set resources deployment example --limits=cpu=new-cpu-limit

        apiVersion: v1
        kind: LimitRange
        metadata:
          name: example
          namespace: example
        spec:
          limits:
          - type: Container
            default:
              cpu: 500m
              memory: 512Mi
            defaultRequest:
              cpu: 250m
              memory: 256Mi
            max:
              cpu: "1"
              memory: 1Gi
            min:
              cpu: 125m
              memory: 128Mi
            type: Container

        apiVersion: v1
        kind: ResourceQuota
        metadata:
          name: example
          namespace: example
        spec:
          hard:
            limits.cpu: "8"
            limits.memory: 8Gi
            requests.cpu: "4"
            requests.memory: 4Gi

# Template

-	oc adm create-bootstrap-project-template -o yaml > file
-   oc patch project.config.openshift.io/cluster --type=merge -p '{"spec":{"projectRequestTemplate":{"name": "project-request"}}}'

# SCC

-	oc create serviceaccount service-account-name
-	oc adm policy add-scc-to-user SCC -z service-account
-	oc set serviceaccount deployment/deployment-name service-account-name

-	oc create sa gitlab-sa
-	oc adm policy add-scc-to-user anyuid -z gitlab-sa
-	oc set serviceaccount deployment/gitlab gitlab-sa
-	oc expose service/gitlab --port 80 --hostname gitlab.apps.ocp4.example.com

-	oc adm policy add-role-to-user cluster-role -z service-account
-	oc adm policy add-cluster-role-to-user cluster-role service-account

-	oc create sa configmap-reloader
-	oc adm policy add-role-to-user edit system:serviceaccount:configmap-reloader:configmap-reloader --rolebinding-name=reloader-edit -n appsec-api

# Job&Cronjob

-	oc create job --dry-run=client -o yaml test --image=registry.access.redhat.com/ubi8/ubi:8.6 -- curl https://example.com
-	oc adm create cronjob --dry-run=client -o yaml test --image=registry.access.redhat.com/ubi8/ubi:8.6 --schedule='0 0 * * *' -- curl https://example.com


        apiVersion: v1
        kind: ConfigMap
        metadata:
          name: maintenance
          app: crictl
        data:
          maintenance.sh: |
            #!/bin/bash
            NODES=$(oc get nodes -o=name) 1
            for NODE in ${NODES} 2
            do
            echo ${NODE}
            oc debug ${NODE} -- \ 3
                chroot /host \
                /bin/bash -xc 'crictl images ; crictl rmi --prune' 4
            echo $?
            done

        apiVersion: batch/v1
        kind: CronJob
        metadata:
          name: image-pruner
        spec:
          schedule: 0 * * * *
          jobTemplate:
            spec:
            template:
                spec:
                dnsPolicy: ClusterFirst
                restartPolicy: Never
                containers:
                - name: image-pruner
                    image: quay.io/openshift/origin-cli:4.12
                    resources: {}
                    command:
                    - /opt/scripts/maintenance.sh 1
                    volumeMounts: 2
                    - name: scripts
                    mountPath: /opt
                volumes: 3
                - name: scripts
                    configMap:
                    name: maintenance
                    defaultMode: 0555

# Examples

-	hosts=$(oc get route --all-namespaces -o jsonpath='{.items[*].spec.host}')
-	for host in $hosts ; do curl https://$host -w "%{url_effective} %{http_code}\n" -o /dev/null -s ; done
  
-	oc get pods --all-namespaces -o jsonpath='{range.items[*]}'\
'{"\n"}{.metadata.namespace}{"\t"}{.metadata.name}{"\t"}'\
'{range.status.conditions[*]}{.type}-{.status},{end}{end}{"\n"}'


        apiVersion: batch/v1
        kind: CronJob
        metadata:
          name: backup-cron
          namespace: database
        spec:
          schedule: "1 0 * * *" 1
          jobTemplate: 2
            spec:
            activeDeadlineSeconds: 600
            parallelism: 2
            template:
                metadata:
                name: backup
                spec:
                serviceAccountName: backup_sa
                containers:
                - name: backup
                    image: example/backup_maker:v1.2.3
                restartPolicy: OnFailure
		  
-   1- The example schedule executes the job at one minute past midnight.
-   2- Specify a regular job in the CronJob's jobTemplate field.
		  
        apiVersion: batch/v1
        kind: Job
        metadata:
          name: backup
          namespace: database
        spec:
          activeDeadlineSeconds: 600 1
          parallelism: 2 2
          template: 3
            metadata:
            name: backup
            spec:
            serviceAccountName: backup_sa 4
            containers:
            - name: backup
                image: example/backup_maker:v1.2.3
            restartPolicy: OnFailure	  

-   1- Optionally, provide a duration limit in seconds. The Job will attempt to terminate if it exceeds the deadline.
-   2- Specify the number of pods to run concurrently.
-   3- The spec includes a pod template.
-   4- Specify the name of the service account to associate with the pod. If you do not specify a service account, then the pod will use the default service account in the namespace.

        apiVersion: batch/v1
        kind: Job
        metadata:
          name: audit-sh
          namespace: automation-scripts
        spec:
          template:
            spec:
            serviceAccountName: auditor
            restartPolicy: Never
            containers:
            - name: audit-sh
                image: registry.ocp4.example.com:8443/openshift4/ose-cli:v4.10
                command: ["/bin/sh", "-c"]
                args:
                - "oc get pods --all-namespaces
                    -o jsonpath='{.items[*].spec.containers[*].image}'
                    | sed 's/ /\\\n/g'
                    | sort
                    | uniq"
                                
        apiVersion: batch/v1
        kind: CronJob
        metadata:
          name: audit-cron
          namespace: automation-scripts
        spec:
          schedule: "*/2 * * * *"
          jobTemplate:
            spec:
              template:
                spec:
                serviceAccountName: auditor
                restartPolicy: Never
                containers:
                - name: audit-sh
                    image: registry.ocp4.example.com:8443/openshift4/ose-cli:v4.10
                    command: ["/bin/sh", "-c"]
                    args:
                    - "oc get pods --all-namespaces
                        -o jsonpath='{.items[*].spec.containers[*].image}'
                        | sed 's/ /\\\n/g'
                        | sort
                        | uniq"
                                    
                        
        apiVersion: batch/v1
        kind: CronJob
        metadata:
          name: group-sync
          namespace: auth-ldapsync
        spec:
          schedule: "*/1 * * * *"  1
          jobTemplate:
            spec:
              template:
                spec:
                  restartPolicy: Never
                  containers:
                  - name: hello
                    image: registry.ocp4.example.com:8443/openshift4/ose-cli:v4.10
                    command:
                    - /bin/sh
                    - -c
                    - oc adm groups sync --sync-config=/etc/config/cron-ldap-sync.yml --confirm  2
                    volumeMounts:  3
                    - mountPath: "/etc/config"
                        name: "ldap-sync-volume"
                    - mountPath: "/etc/secrets"
                        name: "ldap-bind-password"
                  volumes:  4
                    - name: "ldap-sync-volume"
                    configMap:
                        name: ldap-config
                    - name: "ldap-bind-password"
                    secret:
                        secretName: ldap-secret
                  serviceAccountName: ldap-group-syncer
                  serviceAccount: ldap-group-syncer
		  
		
-   oc create configmap <CONFIGMAP-NAME> --from-file ca-bundle.crt=<PATH-TO-CERTIFICATE> -n openshift-config		
-   oc patch proxy/cluster --type=merge --patch='{"spec":{"trustedCA":{"name":"<CONFIGMAP-NAME>"}}}'


        apiVersion: batch/v1
        kind: CronJob
        metadata:
          name: image-pruner
        spec:
          schedule: 4 5 2 * *
          jobTemplate:
            spec:
              template:
                spec:
                  dnsPolicy: ClusterFirst
                  restartPolicy: Never
                  containers:
                  - name: image-pruner
                    image: quay.io/openshift/origin-cli:4.12
                    resources: {}
                    command:
                    - /opt/scripts/maintenance.sh 1
                    volumeMounts: 2
                    - name: scripts
                      mountPath: /opt
                  volumes: 3
                  - name: scripts
                      configMap:
                        name: maintenance
                        defaultMode: 0555
                    
                    
        apiVersion: v1
        kind: ConfigMap
        metadata:
        name: maintenance
        app: crictl
        data:
        maintenance.sh: |
            #!/bin/bash
            NODES=$(oc get nodes -o=name) 1
            for NODE in ${NODES} 2
            do
            echo ${NODE}
            oc debug ${NODE} -- \ 3
                chroot /host \
                /bin/bash -xc 'crictl images ; crictl rmi --prune' 4
            echo $?
            done
	
	
-   oc debug node/master01 -- chroot /host crictl images | egrep '^IMAGE|httpd|nginx' 
-   oc debug node/master01 -- chroot /host crictl rmi --prune


-   oc exec -it statefulset/prometheus-k8s -c prometheus -n openshift-monitoring -- curl -fsSL 'http://localhost:9090/api/v1/alerts' | jq . > alerts.json
	
When a stable version of a feature is released, the beta versions are marked as deprecated and are removed after three Kubernetes releases.


-	The service CA certificate is valid for 26 months by default and is automatically rotated after 13 months. After rotation is a 13-month grace period where the original CA certificate is still valid. During this grace period, each pod that is configured to trust the original CA certificate must be restarted in some way. A service restart automatically injects the new CA bundle.
-	oc get event --sort-by .metadata.creationTimestamp
- oc create resourcequota example --hard=count/deployment=1
-	oc create clusterresourcequota example --project-label-selector=group=dev --hard=requests.cpu=10
-	oc get catalogsource -n openshift-marketplace
-	oc get packagemanifests
-	oc describe packagemanifest web-terminal -n openshift-marketplace

-   oc run query-db -it --rm --image registry.ocp4.example.com:8443/rhel8/mysql-80 --restart Never --command -- /bin/bash -c "mysql -uuser1 -pmypasswd --protocol tcp -h mysql -P3306 sampledb -e 'SHOW DATABASES;'"


-   oc create secret tls custom-tls --cert combined-cert.pem --key wildcard-api-key.pem -n openshift-config
-   oc label configmap <CONFIGMAP-NAME> config.openshift.io/inject-trusted-cabundle=true

-   oc create configmap ca-certs
-   oc label configmap ca-certs config.openshift.io/inject-trusted-cabundle=true
-   oc set volume deployment/hello1 -t configmap --name trusted-ca --add --read-only=true --mount-path /etc/pki/ca-trust/extracted/pem --configmap-name ca-certs
-   oc set data secret <SECRET-NAME> --from-file tls.crt=<PATH-TO-NEW-CERTIFICATE> --from-file tls.key=<PATH-TO-KEY> -n openshift-config
-   oc get events --sort-by='.lastTimestamp' -n openshift-kube-apiserver

-   oc adm drain worker01 --ignore-daemonsets --disable-eviction
-   oc get pods -A -o wide --field-selector spec.nodeName=worker03

        apiVersion: apps/v1
        kind: Deployment
        metadata:
        name: example
        spec:
        replicas: 4
        selector:
            matchLabels:
            app: example
        template:
            metadata:
            labels:
                app: example
            spec:
            nodeSelector:
                node-role.kubernetes.io/infra: ""
            containers:
            - image: example:v1.0
                name: example
                ports:
                - containerPort: 8080
                protocol: TCP


-   oc annotate storageclass standard --overwrite "storageclass.kubernetes.io/is-default-class=true"

-   Monitoring

        Prometheus for deploying Prometheus instances. Display all resource instances using the following command:
        [user@host ~]$ oc get prometheuses -A

        ServiceMonitor for managing configuration files that describe how and where to scrape the data from application services. Display all resource instances using the following command:
        [user@host ~]$ oc get servicemonitors -A

        PodMonitor for managing configuration files for scraping data from pods. Display all resource instances using the following command:
        [user@host ~]$ oc get podmonitors -A

        Alertmanager for deploying and managing alert manager instances. Display all resource instances using the following command:
        [user@host ~]$ oc get alertmanagers -A

        PrometheusRule for managing rules. Rules correspond to a query set, a filter, and any other field. Display all resource instances using the following command:
        [user@host ~]$ oc get prometheusrules -A

        oc adm policy add-cluster-role-to-user cluster-monitoring-view USER

-   alert

        ALERTMANAGER="$(oc get route/alertmanager-main \
        -n openshift-monitoring -o jsonpath='{.spec.host}')"

        [user@host ~]$ curl -s -k -H "Authorization: Bearer $(oc sa get-token \
        prometheus-k8s -n openshift-monitoring)" \
        https://${ALERTMANAGER}/api/v1/alerts | jq .



        "global":
        "resolve_timeout": "5m"
        "receivers":
        - "name": "pagerduty-notification" 1
        "pagerduty_configs":
            "service_key": "e679d31da9b34aa4b8bb4f412186546d" 2
        - "name": "default"
        "route":
        "group_by":
        - "job"
        "group_interval": "5m"
        "group_wait": "30s"
        "receiver": "default"
        "repeat_interval": "12h"
        "routes":
        - "match":
            "alertname": "Watchdog"
            "receiver": "default"
        - "match":
            "severity": "critical" 3
            "receiver": "pagerduty-notification"


"global":
  "resolve_timeout": "5m"
  "smtp_smarthost": "utility.lab.example.com:25" 1
  "smtp_from": "alerts@ocp4.example.com" 2
  "smtp_auth_username": "smtp_training" 3
  "smtp_auth_password": "Red_H4T@!" 4
  "smtp_require_tls": false 5
"receivers":
- "name": "email-notification" 6
  "email_configs": 7
    - "to": "ocp-admins@example.com" 8
- "name": "default"
"route":
  "group_by":
  - "job"
  "group_interval": "5m"
  "group_wait": "30s"
  "receiver": "default"
  "repeat_interval": "12h"
  "routes":
  - "match":
      "alertname": "Watchdog"
    "receiver": "default"
  - "match":
      "severity": "critical"
    "receiver": "email-notification" 9

# logging

        [user@host ~]$ oc get prometheusrules -n openshift-logging
        NAME                             AGE
        collector                        42m
        elasticsearch-prometheus-rules   43m

        [user@host ~]$ oc get prometheusrules collector -n openshift-logging -o yaml
        apiVersion: monitoring.coreos.com/v1
        kind: PrometheusRule
        ...output omitted...
        spec:
        groups:
        - name: logging_fluentd.alerts
            rules:
            - alert: FluentdNodeDown
        ...output omitted...
            - alert: FluentdQueueLengthIncreasing
            annotations:
                message: For the last hour, fluentd {{ $labels.instance }} output '{{ $labels.plugin_id
                }}' average buffer queue length has increased continuously.
                summary: Fluentd is unable to keep up with traffic over time for forwarder
                output {{ $labels.plugin_id }}.
            expr: |
                ( 0 * (deriv(fluentd_output_status_emit_records[1m] offset 1h)))  + on(pod,plugin_id)  ( deriv(fluentd_output_status_buffer_queue_length[10m]) > 0 and delta(fluentd_output_status_buffer_queue_length[1h]) > 1 )
            for: 1h
            labels:
                service: fluentd
                severity: Warning
        ...output omitted...

-   Click Observe → Metrics, enter the expression max_over_time(fluentd_output_status_buffer_queue_length[1m]), and then click Run Queries. Review the chart that displays the maximum number of logs per minute in the collector buffer. Notice the increasing queue length.

-   Click Observe → Metrics, enter the expression max_over_time(fluentd_output_status_buffer_queue_length[1m]), and then click Run Queries. Review the chart that displays the maximum number of logs per minute in the collector's buffer.

-   oc autoscale deployment/image-registry --min=5 --max=7 --cpu-percent=75

# Intro

Chapter 1, Declarative Resource Management
Deploy and update applications from resource manifests that are parameterized for different target environments.

Deploy and update applications from resource manifests that are stored as YAML files.

Deploy and update applications from resource manifests that are augmented by Kustomize.

Chapter 2, Deploy Packaged Applications
Deploy and update applications from resource manifests that are packaged for sharing and distribution.

Deploy an application and its dependencies from resource manifests that are stored in an OpenShift template.

Deploy and update applications from resource manifests that are packaged as Helm charts.

Chapter 3, Authentication and Authorization
Configure authentication with the HTPasswd identity provider and assign roles to users and groups.

Configure the HTPasswd identity provider for OpenShift authentication.

Define role-based access controls and apply permissions to users.

Chapter 4, Network Security
Protect network traffic between applications inside and outside the cluster.

Allow and protect network connections to applications inside an OpenShift cluster.

Restrict network traffic between projects and pods.

Configure and use automatic service certificates.

Chapter 5, Expose non-HTTP/SNI Applications
Expose applications to external access without using an ingress controller.

Expose applications to external access by using load balancer services.

Expose applications to external access by using a secondary network.

Chapter 6, Enable Developer Self-Service
Configure clusters for safe self-service by developers from multiple teams, and disallow self-service if operations staff must provision projects.

Configure compute resource quotas and Kubernetes resource count quotas per project and cluster-wide.

Configure default and maximum compute resource requirements for pods per project.

Configure default quotas, limit ranges, role bindings, and other restrictions for new projects, and the allowed users to self-provision new projects.

Chapter 7, Manage Kubernetes Operators
Install and update operators that the Operator Lifecycle Manager and the Cluster Version Operator manage.

Explain the operator pattern and different approaches for installing and updating Kubernetes operators.

Install and update operators by using the web console and the Operator Lifecycle Manager.

Install and update operators by using the Operator Lifecycle Manager APIs.

Chapter 8, Application Security
Run applications that require elevated or special privileges from the host operating system or Kubernetes.

Create service accounts and apply permissions, and manage security context constraints.

Run an application that requires access to the Kubernetes API of the application's cluster.

Automate regular cluster and application management tasks by using Kubernetes cron jobs.

Chapter 9, OpenShift Updates
Update an OpenShift cluster and minimize disruption to deployed applications.

Describe the cluster update process.

Identify applications that use deprecated Kubernetes APIs.

Update OLM-managed operators by using the web console and CLI.