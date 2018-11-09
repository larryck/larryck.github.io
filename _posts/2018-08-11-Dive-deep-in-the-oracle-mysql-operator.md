---
title: Dive deep in the oracle mysql operator
description: 
categories:
- k8s
tags:
- statefulset
---



Statefulset app support is one of the key features of the containerized hyper-converged system. Recently I had spent some time on highly available MySQL solutions on containers, and read the article [Running MySQL on Kubernetes using an operator](https://banzaicloud.com/blog/mysql-on-kubernetes/) wrote by Gabor Kozma. The article compares three open-sourced mysql operators and opts for the Oracle MySQL operator. It also mentions some features that oracle operator supports, so I decided to take a closer look at it. 

# CustomResourceDefinition
The oracle operator uses three CustomResourceDefinitions (MySQLCluster, MySQLBackup, MySQLRestore) to manage mysql clusters. The operator has three controllers related to the CustomResourceDefinitions, namely the clusterController, the backupController, and the restoreController. 

## clusterController
The clusterController handles cluster create, update, and delete. When it receives a cluster create request, it finds the correct cluster object, validate its version, and create services and statefulsets for the cluster. The clusterController keeps comparing the actual states of clusters to their desired states and converging the two. 

User can config mysql cluster during creating with ClusterSpec. Below is how ClusterSpec defines:

	// ClusterSpec defines the attributes a user can specify when creating a cluster
	type ClusterSpec struct {
	    // Version defines the MySQL Docker image version.
	    Version string `json:"version"`
	    // Repository defines the image repository from which to pull the MySQL server image.
	    Repository string `json:"repository"`
	    // ImagePullSecret defines the name of the secret that contains the
	    // required credentials for pulling from the specified Repository.
	    ImagePullSecrets []corev1.LocalObjectReference `json:"imagePullSecret"`
	    // Members defines the number of MySQL instances in a cluster
	    Members int32 `json:"members,omitempty"`
	    // BaseServerID defines the base number used to create unique server_id
	    // for MySQL instances in the cluster. Valid range 1 to 4294967286.
	    // If omitted in the manifest file (or set to 0) defaultBaseServerID
	    // value will be used.
	    BaseServerID uint32 `json:"baseServerId,omitempty"`
	    // MultiMaster defines the mode of the MySQL cluster. If set to true,
	    // all instances will be R/W. If false (the default), only a single instance
	    // will be R/W and the rest will be R/O.
	    MultiMaster bool `json:"multiMaster,omitempty"`
	    // NodeSelector is a selector which must be true for the pod to fit on a node.
	    // Selector which must match a node's labels for the pod to be scheduled on that node.
	    // More info: https://kubernetes.io/docs/concepts/configuration/assign-pod-node/
	    // +optional
	    NodeSelector map[string]string `json:"nodeSelector,omitempty"`
	    // If specified, affinity will define the pod's scheduling constraints
	    // +optional
	    Affinity *corev1.Affinity `json:"affinity,omitempty"`
	    // VolumeClaimTemplate allows a user to specify how volumes inside a MySQL cluster
	    // +optional
	    VolumeClaimTemplate *corev1.PersistentVolumeClaim `json:"volumeClaimTemplate,omitempty"`
	    // BackupVolumeClaimTemplate allows a user to specify a volume to temporarily store the
	    // data for a backup prior to it being shipped to object storage.
	    // +optional
	    BackupVolumeClaimTemplate *corev1.PersistentVolumeClaim `json:"backupVolumeClaimTemplate,omitempty"`
	    // If defined, we use this secret for configuring the MYSQL_ROOT_PASSWORD
	    // If it is not set we generate a secret dynamically
	    // +optional
	    RootPasswordSecret *corev1.LocalObjectReference `json:"rootPasswordSecret,omitempty"`
	    // Config allows a user to specify a custom configuration file for MySQL.
	    // +optional
	    Config *corev1.LocalObjectReference `json:"config,omitempty"`
	    // SSLSecret allows a user to specify custom CA certificate, server certificate
	    // and server key for group replication SSL.
	    // +optional
	    SSLSecret *corev1.LocalObjectReference `json:"sslSecret,omitempty"`
	    // SecurityContext holds Pod-level security attributes and common Container settings.
	    SecurityContext *corev1.PodSecurityContext `json:"securityContext,omitempty"`
	    // Tolerations allows specifying a list of tolerations for controlling which
	    // set of Nodes a Pod can be scheduled on
	    Tolerations *[]corev1.Toleration `json:"tolerations,omitempty"`
	    // Resources holds ResourceRequirements for the MySQL Agent & Server Containers
	    Resources *Resources `json:"resources,omitempty"`
	}


Service for a mysql cluster is a headless service which exports port 3306 and labeled with cluster name. Statefulset for a mysql cluster containers two containers, namely mysqlserverContainer and mysqlAgentContainer.

Here I get the start commands for MysqlServerContainer

	args := []string{
		"--server_id=$(expr $base + $index)",
		"--datadir=/var/lib/mysql",
		"--user=mysql",
		"--gtid_mode=ON",
		"--log-bin",
		"--binlog_checksum=NONE",
		"--enforce_gtid_consistency=ON",
		"--log-slave-updates=ON",
		"--binlog-format=ROW",
		"--master-info-repository=TABLE",
		"--relay-log-info-repository=TABLE",
		"--transaction-write-set-extraction=XXHASH64",
		fmt.Sprintf("--relay-log=%s-${index}-relay-bin", cluster.Name),
		fmt.Sprintf("--report-host=\"%[1]s-${index}.%[1]s\"", cluster.Name),
		"--log-error-verbosity=3",
	}

	if cluster.RequiresCustomSSLSetup() {
		args = append(args,
			"--ssl-ca=/etc/ssl/mysql/ca.crt",
			"--ssl-cert=/etc/ssl/mysql/tls.crt",
			"--ssl-key=/etc/ssl/mysql/tls.key")
	}

	entryPointArgs := strings.Join(args, " ")

	cmd := fmt.Sprintf(`
         # Set baseServerID
         base=%d

         # Finds the replica index from the hostname, and uses this to define
         # a unique server id for this instance.
         index=$(cat /etc/hostname | grep -o '[^-]*$')
         /entrypoint.sh %s`, baseServerID, entryPointArgs)

It also exports container port 3306 and sets password/log_console as environment variables. 

mysqlAgentContainer has environment variables like MYSQL_CLUSTER_NAME/REPLICATION_GROUP_SEEDS/MYSQL_CLUSTER_MULTI_MASTER. Mysqlagent initializes the mysql cluster after all the mysql instances start. The No. 0 instance handles the process of bootstrapping, the mysqlsh python command create_cluster is used for that purpose. Mysql cluster controller watches the status of each mysql instance, when mysql container restarts, it tries to rejoin the mysql instance into cluster.

mysqlAgentContainer also watches backup and restore objects and handles mysql backup and restore function when needed. Backup process prioritizes secondary instances of the mysql cluster, the primary instance will be chosen at last if no secondary instance is found. Backup and restore process both use the tool mysqldump.


## Metrics
The oracle mysql operator registers several metrics to track mysql operating events, including
clusterCreateCount/instanceAddCount/instanceRejoinCount. 
