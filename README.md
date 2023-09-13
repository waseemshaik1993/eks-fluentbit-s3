### eks-fluentbit
Sending EKS logs to CloudWatch and S3 with Fluent 


Need to get logging on your Kubernetes control plane? Need to monitor some of your application logs? Need your developers to debug some pod or container logs without giving admin access? This is for you…


# EKS 101
EKS is a managed service provided by AWS that simplifies the deployment and management of Kubernetes clusters. Kubernetes is an open-source container orchestration platform.

# Fluent Bit 101
Fluent Bit is a super fast, lightweight, and highly scalable logging and metrics processor and forwarder. It is the preferred choice for cloud and containerized environments

# Pre-requisites
Basic Linux and Kubernetes administration experience
Working EKS cluster
Connected AWS Account
IAM role with write permissions to CloudWatch and S3
S3 bucket
If you need help creating an IAM role, please look at my other article here: https://medium.com/aws-tip/creating-iam-roles-in-terraform-with-custom-iam-policies-67aae9059238

# Objectives
A logging namespace creation.
Fluent-bit service deployed into cluster and running.
Kubernetes logs are being stored in CloudWatch.
Kubernetes logs are being stored in S3.


1. Folder set up
In your repository, create a folder called fluentbit. Create the following files below:

namespace.yaml
cluster-role.yaml
serviceaccount.yaml
cluster-info.yaml
configmap.yaml
daemonset.yaml
2. Create the namespace
We first need to create a namespace. Copy the code below into namespace.yaml. Feel free to change the name to whatever you prefer. In this tutorial, we will be calling this logging.

#Namespace creation
apiVersion: v1
kind: Namespace
metadata:
  name: logging
  labels:
    name: logging
CD into our new folder Fluentbit and run the following command to create it:

kubectl apply -f fluenbit/namespace.yaml
Check that the namespace has been created by running the below command. We should see it populated in our terminal.
kubectl get ns 
3. Create the cluster-role and cluster-role-binding.
You will need to create an IAM role with permissions to Cloudwatch and S3, which will be used for our Fluent Bit service to talk to and send logs with.

Copy the below into cluster-role.yaml.

Feel free to change the role names as you wish.

# Creation of the kubernetes cluster role assigned to our Service
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: fluent-bit-role
rules:
  - nonResourceURLs:
      - /metrics
    verbs:
      - get
  - apiGroups: [""]
    resources:
      - namespaces
      - pods
      - pods/logs
      - nodes
      - nodes/proxy
    verbs: ["get", "list", "watch"]
---
# The cluster role binding that attaches the service account to our cluster role above
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: fluent-bit-role-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: fluent-bit-role
subjects:
  - kind: ServiceAccount
    name: fluent-bit
    namespace: logging
We can see that the cluster permissions we are assigning to our fluent-bit-role allow access to all pods, namespaces, and nodes.

Create these roles using the below command:

kubectl apply -f fluenbit/cluster-role.yaml
Check they have been created using the command below:

kubectl get clusterroles -n logging

kubectl get clusterrolebindings -n logging


4. Create service account
We now need to create the service account, which will use the IAM role to talk to CloudWatch and S3. This service account is used to run the Fluent Bit daemon set.

Copy the code below into your service-account.yaml file. (Remember to replace the IAM role with yours.)

# Creation of the kubernetes service account
apiVersion: v1
kind: ServiceAccount
metadata:
  name: fluent-bit
  namespace: logging
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::3938209364:role/fluent-bit
Create this using the below command:

kubectl apply -f fluenbit/service-account.yaml
Check it has been created using the below command:

kubectl get sa -n logging


5. Set up Fluent bit info
We need to create a config map. A Config map is an API object that lets you store configuration for other objects to use.

Copy the below into cluster-info.yaml.

Remember to change your cluster name and region!

# Creation of configmap 1
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluent-bit-cluster-info
  labels:
    app: fluent-bit
data:
  ClusterName: aarons-cluster
  RegionName: us-east-1
  FluentBitHttpPort: "2020"
  FluentBitReadFromHead: "OFF"
  logs.region: us-east-1
  cluster.name: aarons-cluster
  http.server: "ON"
  http.port: "2020"
  read.head: "OFF"
  read.tail: "ON"
Create the configmap using the below command:

kubectl apply -f fluenbit/cluster-info.yaml
Check it has been created using the below command:

kubectl describe configmap fluent-bit-cluster-info
6. Create your Fluent bit ConfigMap
We need to create another configmap. This will allow you to customise what actions your Fluentbit service will take. You can specify a number of supported Fluentbit plugins by updating the YAML.

Copy the below code into your configmap.yaml.

# Fluentbit config map
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluent-bit-config
  namespace: logging
  labels:
    k8s-app: fluent-bit
data:
  fluent-bit.conf: |
    [SERVICE]
        Flush                     5
        Grace                     30
        Log_Level                 info
        Daemon                    off
        Parsers_File              parsers.conf
        HTTP_Server               ${HTTP_SERVER}
        HTTP_Listen               0.0.0.0
        HTTP_Port                 ${HTTP_PORT}
        storage.path              /var/fluent-bit/state/flb-storage/
        storage.sync              normal
        storage.checksum          off
        storage.backlog.mem_limit 5M
        
    @INCLUDE application-log.conf
    @INCLUDE dataplane-log.conf
    @INCLUDE host-log.conf
  
  application-log.conf: |
    [INPUT]
        Name                tail
        Tag                 application.*
        Exclude_Path        /var/log/containers/cloudwatch-agent*, /var/log/containers/fluent-bit*, /var/log/containers/aws-node*, /var/log/containers/kube-proxy*, /var/log/containers/cert-manager*, /var/log/containers/efs*, /var/log/containers/csi*, /var/log/containers/secrets*, /var/log/containers/istio*, /var/log/containers/external*, /var/log/containers/coredns*, /var/log/containers/cluster*, /var/log/containers/helm*, /var/log/containers/kustomize*, /var/log/containers/notification*
        Path                /var/log/containers/*.log
        multiline.parser    docker, cri
        DB                  /var/fluent-bit/state/flb_container.db
        Mem_Buf_Limit       50MB
        Skip_Long_Lines     On
        Refresh_Interval    10
        Rotate_Wait         30
        storage.type        filesystem
        Read_from_Head      ${READ_FROM_HEAD}

    [INPUT]
        Name                tail
        Tag                 application.*
        Path                /var/log/containers/fluent-bit*
        multiline.parser    docker, cri
        DB                  /var/fluent-bit/state/flb_log.db
        Mem_Buf_Limit       5MB
        Skip_Long_Lines     On
        Refresh_Interval    10
        Read_from_Head      ${READ_FROM_HEAD}

    [INPUT]
        Name                tail
        Tag                 application.*
        Path                /var/log/containers/cloudwatch-agent*
        multiline.parser    docker, cri
        DB                  /var/fluent-bit/state/flb_cwagent.db
        Mem_Buf_Limit       5MB
        Skip_Long_Lines     On
        Refresh_Interval    10
        Read_from_Head      ${READ_FROM_HEAD}

    [FILTER]
        Name                kubernetes
        Match               application.*
        Kube_URL            https://kubernetes.default.svc:443
        Kube_Tag_Prefix     application.var.log.containers.
        Merge_Log           On
        Merge_Log_Key       log_processed
        K8S-Logging.Parser  On
        K8S-Logging.Exclude Off
        Labels              Off
        Annotations         Off
        Use_Kubelet         On
        Kubelet_Port        10250
        Buffer_Size         0

    [OUTPUT]
        Name                cloudwatch_logs
        Match               host.*
        region              ${AWS_REGION}
        log_group_name      /aws/containerinsights/${CLUSTER_NAME}/application
        log_stream_prefix   ${HOST_NAME}-
        auto_create_group   true
        extra_user_agent    container-insights

    [OUTPUT]
        Name                  s3
        Match                 application.*
        bucket                aarons-logs-bucket
        region                ${AWS_REGION}
        total_file_size       250M
        compression           gzip
        s3_key_format         /fluent-bit-logs/$TAG/%Y/%m/%d/%H_%M_%S.gz
        static_file_path      On     

  dataplane-log.conf: |
    [INPUT]
        Name                systemd
        Tag                 dataplane.systemd.*
        Systemd_Filter      _SYSTEMD_UNIT=docker.service
        Systemd_Filter      _SYSTEMD_UNIT=kubelet.service
        DB                  /var/fluent-bit/state/systemd.db
        Path                /var/log/journal
        Read_From_Tail      ${READ_FROM_TAIL}

    [INPUT]
        Name                tail
        Tag                 dataplane.tail.*
        Path                /var/log/containers/aws-node*, /var/log/containers/kube-proxy*
        multiline.parser    docker, cri
        DB                  /var/fluent-bit/state/flb_dataplane_tail.db
        Mem_Buf_Limit       50MB
        Skip_Long_Lines     On
        Refresh_Interval    10
        Rotate_Wait         30
        storage.type        filesystem
        Read_from_Head      ${READ_FROM_HEAD}

    [FILTER]
        Name                modify
        Match               dataplane.systemd.*
        Rename              _HOSTNAME                   hostname
        Rename              _SYSTEMD_UNIT               systemd_unit
        Rename              MESSAGE                     message
        Remove_regex        ^((?!hostname|systemd_unit|message).)*$

    [FILTER]
        Name                aws
        Match               dataplane.*
        imds_version        v1

    [OUTPUT]
        Name                cloudwatch_logs
        Match               dataplane.*
        region              ${AWS_REGION}
        log_group_name      /aws/containerinsights/${CLUSTER_NAME}/dataplane
        log_stream_prefix   ${HOST_NAME}-
        auto_create_group   true
        extra_user_agent    container-insights

  host-log.conf: |
    [INPUT]
        Name                tail
        Tag                 host.dmesg
        Path                /var/log/dmesg
        Key                 message
        DB                  /var/fluent-bit/state/flb_dmesg.db
        Mem_Buf_Limit       5MB
        Skip_Long_Lines     On
        Refresh_Interval    10
        Read_from_Head      ${READ_FROM_HEAD}

    [INPUT]
        Name                tail
        Tag                 host.messages
        Path                /var/log/messages
        Parser              syslog
        DB                  /var/fluent-bit/state/flb_messages.db
        Mem_Buf_Limit       5MB
        Skip_Long_Lines     On
        Refresh_Interval    10
        Read_from_Head      ${READ_FROM_HEAD}

    [INPUT]
        Name                tail
        Tag                 host.secure
        Path                /var/log/secure
        Parser              syslog
        DB                  /var/fluent-bit/state/flb_secure.db
        Mem_Buf_Limit       5MB
        Skip_Long_Lines     On
        Refresh_Interval    10
        Read_from_Head      ${READ_FROM_HEAD}

    [FILTER]
        Name                aws
        Match               host.*
        imds_version        v1

    [OUTPUT]
        Name                cloudwatch_logs
        Match               host.*
        region              ${AWS_REGION}
        log_group_name      /aws/containerinsights/${CLUSTER_NAME}/host
        log_stream_prefix   ${HOST_NAME}.
        auto_create_group   true
        extra_user_agent    container-insights

  parsers.conf: |
    [PARSER]
        Name                syslog
        Format              regex
        Regex               ^(?<time>[^ ]* {1,2}[^ ]* [^ ]*) (?<host>[^ ]*) (?<ident>[a-zA-Z0-9_\/\.\-]*)(?:\[(?<pid>[0-9]+)\])?(?:[^\:]*\:)? *(?<message>.*)$
        Time_Key            time
        Time_Format         %b %d %H:%M:%S

    [PARSER]
        Name                container_firstline
        Format              regex
        Regex               (?<log>(?<="log":")\S(?!\.).*?)(?<!\\)".*(?<stream>(?<="stream":").*?)".*(?<time>\d{4}-\d{1,2}-\d{1,2}T\d{2}:\d{2}:\d{2}\.\w*).*(?=})
        Time_Key            time
        Time_Format         %Y-%m-%dT%H:%M:%S.%LZ

    [PARSER]
        Name                cwagent_firstline
        Format              regex
        Regex               (?<log>(?<="log":")\d{4}[\/-]\d{1,2}[\/-]\d{1,2}[ T]\d{2}:\d{2}:\d{2}(?!\.).*?)(?<!\\)".*(?<stream>(?<="stream":").*?)".*(?<time>\d{4}-\d{1,2}-\d{1,2}T\d{2}:\d{2}:\d{2}\.\w*).*(?=})
        Time_Key            time
        Time_Format         %Y-%m-%dT%H:%M:%S.%LZ
The config map is broken down into:

Application
Host
Data plane
(Make sure you look at the outputs from lines 81 to 98.)

From this configmap you can change the following:

We will be defining how we store the logs in CloudWatch and which S3 bucket we send them to.
You can amend the Input and Filter sections to capture just what you need.
You can also change the formatting of the logs.
Create the configmap using the below command:

kubectl apply -f fluenbit/configmap.yaml
Check it has been created using the below command:

kubectl describe configmap fluent-bit-config
7. Create the Daemon set
We are almost there! We need to create the daemon set for Fluentbit. This is the main deployment file, which uses the official Docker image to run the service.

Copy the below code into daemonset.yaml.

# Fluent Bit Deployment
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluent-bit
  namespace: logging
  labels:
    k8s-app: fluent-bit
    version: v1
    kubernetes.io/cluster-service: "true"
spec:
  selector:
    matchLabels:
      k8s-app: fluent-bit
  template:
    metadata:
      labels:
        k8s-app: fluent-bit
        version: v1
        kubernetes.io/cluster-service: "true"
    spec:
      containers:
      - name: fluent-bit
        image: public.ecr.aws/aws-observability/aws-for-fluent-bit:stable
        imagePullPolicy: Always
        env:
            - name: AWS_REGION
              valueFrom:
                configMapKeyRef:
                  name: fluent-bit-cluster-info
                  key: logs.region
            - name: CLUSTER_NAME
              valueFrom:
                configMapKeyRef:
                  name: fluent-bit-cluster-info
                  key: cluster.name
            - name: HTTP_SERVER
              valueFrom:
                configMapKeyRef:
                  name: fluent-bit-cluster-info
                  key: http.server
            - name: HTTP_PORT
              valueFrom:
                configMapKeyRef:
                  name: fluent-bit-cluster-info
                  key: http.port
            - name: READ_FROM_HEAD
              valueFrom:
                configMapKeyRef:
                  name: fluent-bit-cluster-info
                  key: read.head
            - name: READ_FROM_TAIL
              valueFrom:
                configMapKeyRef:
                  name: fluent-bit-cluster-info
                  key: read.tail
            - name: HOST_NAME
              valueFrom:
                fieldRef:
                  fieldPath: spec.nodeName
            - name: HOSTNAME
              valueFrom:
                fieldRef:
                  apiVersion: v1
                  fieldPath: metadata.name
            - name: CI_VERSION
              value: "k8s/1.3.16"
        resources:
            limits:
              memory: 200Mi
            requests:
              cpu: 500m
              memory: 100Mi
        volumeMounts:
        # Please don't change below read-only permissions
        - name: fluentbitstate
          mountPath: /var/fluent-bit/state
        - name: varlog
          mountPath: /var/log
          readOnly: true
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
        - name: fluent-bit-config
          mountPath: /fluent-bit/etc/
        - name: runlogjournal
          mountPath: /run/log/journal
          readOnly: true
        - name: dmesg
          mountPath: /var/log/dmesg
          readOnly: true
      terminationGracePeriodSeconds: 10
      hostNetwork: true
      dnsPolicy: ClusterFirstWithHostNet
      volumes:
      - name: fluentbitstate
        hostPath:
          path: /var/fluent-bit/state
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
      - name: fluent-bit-config
        configMap:
          name: fluent-bit-config
      - name: runlogjournal
        hostPath:
          path: /run/log/journal
      - name: dmesg
        hostPath:
          path: /var/log/dmesg
      serviceAccountName: fluent-bit
      tolerations:
      - key: node-role.kubernetes.io/master
        operator: Exists
        effect: NoSchedule
      - operator: "Exists"
        effect: "NoExecute"
      - operator: "Exists"
        effect: "NoSchedule"
You don’t really need to make any amendments to this code.

Create the deployment using the below command:

kubectl apply -f fluenbit/daemonset.yaml
Check it has been created using the below:

kubectl get pods -n logging
Your fluent bit services should now be running..

8. Check CloudWatch and S3
Now that you have done all the hard work, you can go check CloudWatch and S3. I would give it 15–30 minutes for the services to start working and sending logs through.

Cloudwatch check:

Open the CloudWatch console at https://console.aws.amazon.com/cloudwatch/.
In the navigation pane, choose Log groups.
Make sure that you’re in the region where you deployed Fluent Bit.
Check the list of log groups in the region. You should see the following:
/aws/containerinsights/Cluster_Name/application
/aws/containerinsights/Cluster_Name/host
/aws/containerinsights/Cluster_Name/dataplane
S3 Check:

Open the S3 console at https://console.aws.amazon.com/s3.
In the navigation pane, choose Buckets.
Make sure that you’re in the region where you deployed Fluent Bit.
Open your bucket, and you should see some logs starting with fluent-bit.
9. Conclusion
You should now have logging set up for your Kubernetes cluster! Forgive the long-winded tutorial, but I feel it’s important to break down these big chunks for better understanding.

Remember to set retention on your CloudWatch logs! To save money
Remember to add lifecycle archival to your S3 objects! To save money
Ensure your configmaps are correct with the right names and regions.
Remember to check the service and pod logs to find any errors.
Good luck!
