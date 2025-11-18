# Comprehensive Kubernetes kubectl Commands for Spring Boot + Kafka + Redis + MySQL

## 📋 **Table of Contents**
1. [Pod Management & Inspection](#pods)
2. [Logs & Debugging](#logs)
3. [Secrets Management](#secrets)
4. [ConfigMaps](#configmaps)
5. [Deployments & StatefulSets](#deployments)
6. [Services & Networking](#networking)
7. [Resource Monitoring](#monitoring)
8. [Spring Boot Specific](#spring-boot)
9. [Kafka Debugging](#kafka)
10. [Redis Debugging](#redis)
11. [MySQL Debugging](#mysql)
12. [Troubleshooting Scenarios](#troubleshooting)

---

## <a name="pods"></a>🔍 **1. Pod Management & Inspection**

### **LIST PODS**

```bash
# List all pods in namespace
kubectl get pods -n <namespace>

# List all pods across all namespaces
kubectl get pods --all-namespaces

# List pods with more details (node, IP, etc.)
kubectl get pods -n <namespace> -o wide

# List pods with labels
kubectl get pods -n <namespace> --show-labels

# Filter pods by label selector
kubectl get pods -n <namespace> -l app=bhc
kubectl get pods -n <namespace> -l app=bhc,version=v1

# Watch pods in real-time
kubectl get pods -n <namespace> -w

# List pods sorted by restart count
kubectl get pods -n <namespace> --sort-by='.status.containerStatuses[0].restartCount'

# List pods sorted by age
kubectl get pods -n <namespace> --sort-by=.metadata.creationTimestamp

# List only running pods
kubectl get pods -n <namespace> --field-selector=status.phase=Running

# List failed/error pods
kubectl get pods -n <namespace> --field-selector=status.phase=Failed
kubectl get pods -n <namespace> --field-selector=status.phase=Pending
```

### **DESCRIBE & INSPECT PODS**

```bash
# Get detailed information about a pod
kubectl describe pod <pod-name> -n <namespace>

# Get pod YAML configuration
kubectl get pod <pod-name> -n <namespace> -o yaml

# Get pod JSON configuration
kubectl get pod <pod-name> -n <namespace> -o json

# Get specific field from pod (JSONPath)
kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.status.podIP}'
kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.spec.containers[*].name}'
kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.spec.containers[*].image}'

# Check pod resource requests and limits
kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.spec.containers[*].resources}'

# Get pod events
kubectl get events -n <namespace> --field-selector involvedObject.name=<pod-name>

# Get container IDs
kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.status.containerStatuses[*].containerID}'

# Check pod readiness and liveness probes
kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.spec.containers[*].livenessProbe}'
kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.spec.containers[*].readinessProbe}'

# Get pod's node
kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.spec.nodeName}'
```

### **POD INTERACTION**

```bash
# Execute bash in a pod
kubectl exec -it <pod-name> -n <namespace> -- /bin/bash
kubectl exec -it <pod-name> -n <namespace> -- /bin/sh

# Execute specific container in multi-container pod
kubectl exec -it <pod-name> -c <container-name> -n <namespace> -- /bin/bash

# Execute single command
kubectl exec <pod-name> -n <namespace> -- ls -la /app
kubectl exec <pod-name> -n <namespace> -- env
kubectl exec <pod-name> -n <namespace> -- ps aux
kubectl exec <pod-name> -n <namespace> -- df -h
kubectl exec <pod-name> -n <namespace> -- cat /etc/hosts

# Copy files from pod to local
kubectl cp <namespace>/<pod-name>:/path/to/file ./local-file
kubectl cp <namespace>/<pod-name>:/app/logs/application.log ./app.log

# Copy files from local to pod
kubectl cp ./local-file <namespace>/<pod-name>:/path/to/destination

# Port forward to access pod locally
kubectl port-forward <pod-name> -n <namespace> 8080:8080
kubectl port-forward <pod-name> -n <namespace> 8080:8080 9090:9090

# Port forward with specific address binding
kubectl port-forward --address 0.0.0.0 <pod-name> -n <namespace> 8080:8080

# Delete a pod (will restart if managed by deployment)
kubectl delete pod <pod-name> -n <namespace>

# Force delete a stuck pod
kubectl delete pod <pod-name> -n <namespace> --grace-period=0 --force
```

---

## <a name="logs"></a>📝 **2. Logs & Debugging**

### **BASIC LOGS**

```bash
# Get logs from a pod
kubectl logs <pod-name> -n <namespace>

# Follow logs in real-time (tail -f)
kubectl logs -f <pod-name> -n <namespace>

# Get logs with timestamps
kubectl logs <pod-name> -n <namespace> --timestamps

# Get last N lines
kubectl logs <pod-name> -n <namespace> --tail=100
kubectl logs <pod-name> -n <namespace> --tail=1000

# Get logs since specific time
kubectl logs <pod-name> -n <namespace> --since=1h
kubectl logs <pod-name> -n <namespace> --since=30m
kubectl logs <pod-name> -n <namespace> --since=10s

# Get logs from specific time
kubectl logs <pod-name> -n <namespace> --since-time='2025-11-18T10:00:00Z'
```

### **CONTAINER-SPECIFIC LOGS**

```bash
# Get logs from specific container in multi-container pod
kubectl logs <pod-name> -c <container-name> -n <namespace>

# Get logs from init container
kubectl logs <pod-name> -c <init-container-name> -n <namespace>

# Get all containers logs
kubectl logs <pod-name> -n <namespace> --all-containers=true
```

### **PREVIOUS/CRASHED CONTAINER LOGS**

```bash
# Get logs from previous container instance (if crashed/restarted)
kubectl logs <pod-name> -n <namespace> --previous

# Get previous logs from specific container
kubectl logs <pod-name> -c <container-name> -n <namespace> --previous
```

### **LOGS FROM MULTIPLE PODS**

```bash
# Get logs from all pods with label selector
kubectl logs -l app=bhc -n <namespace>

# Follow logs from all pods with label
kubectl logs -f -l app=bhc -n <namespace>

# Get logs from all pods in deployment
kubectl logs -f deployment/<deployment-name> -n <namespace>
```

### **ADVANCED LOG FILTERING**

```bash
# Stream logs and grep for errors
kubectl logs -f <pod-name> -n <namespace> | grep -i error
kubectl logs -f <pod-name> -n <namespace> | grep -i exception
kubectl logs -f <pod-name> -n <namespace> | grep -i "OutOfMemoryError"

# Get logs and save to file
kubectl logs <pod-name> -n <namespace> > pod-logs.txt

# Get logs with context
kubectl logs <pod-name> -n <namespace> | grep -A 10 -B 10 "error"

# Multiple grep patterns
kubectl logs -f <pod-name> -n <namespace> | grep -E "ERROR|WARN|FATAL"
```

### **SPRING BOOT SPECIFIC LOG QUERIES**

```bash
# Find startup issues
kubectl logs <pod-name> -n <namespace> | grep "Started BhcApplication"
kubectl logs <pod-name> -n <namespace> | grep "APPLICATION FAILED TO START"

# Check database connection
kubectl logs <pod-name> -n <namespace> | grep -i "jdbc"
kubectl logs <pod-name> -n <namespace> | grep -i "hikaripool"
kubectl logs <pod-name> -n <namespace> | grep -i "datasource"

# Check Kafka connection
kubectl logs <pod-name> -n <namespace> | grep -i "kafka"
kubectl logs <pod-name> -n <namespace> | grep -i "consumer"
kubectl logs <pod-name> -n <namespace> | grep -i "producer"

# Check Redis connection
kubectl logs <pod-name> -n <namespace> | grep -i "redis"
kubectl logs <pod-name> -n <namespace> | grep -i "lettuce"

# Find Java exceptions
kubectl logs <pod-name> -n <namespace> | grep -A 20 "Exception"
kubectl logs <pod-name> -n <namespace> | grep -A 20 "java.lang"

# Check health endpoint calls
kubectl logs <pod-name> -n <namespace> | grep "/actuator/health"

# Find OOM errors
kubectl logs <pod-name> -n <namespace> | grep "OutOfMemoryError"
kubectl logs <pod-name> -n <namespace> --previous | grep "OutOfMemoryError"
```

---

## <a name="secrets"></a>🔐 **3. Secrets Management**

### **LIST & VIEW SECRETS**

```bash
# List all secrets
kubectl get secrets -n <namespace>

# List secrets with more info
kubectl get secrets -n <namespace> -o wide

# Describe secret (doesn't show values)
kubectl describe secret <secret-name> -n <namespace>

# Get secret in YAML (base64 encoded)
kubectl get secret <secret-name> -n <namespace> -o yaml

# Get secret in JSON
kubectl get secret <secret-name> -n <namespace> -o json

# List secret keys only
kubectl get secret <secret-name> -n <namespace> -o jsonpath='{.data}'
```

### **DECODE SECRETS**

```bash
# Decode specific secret key
kubectl get secret <secret-name> -n <namespace> -o jsonpath='{.data.<key>}' | base64 -d

# Decode all keys in a secret
kubectl get secret <secret-name> -n <namespace> -o json | jq -r '.data | map_values(@base64d)'

# Decode and format nicely
kubectl get secret <secret-name> -n <namespace> -o go-template='{{range $k,$v := .data}}{{printf "%s: " $k}}{{if not $v}}{{$v}}{{else}}{{$v | base64decode}}{{end}}{{"\n"}}{{end}}'
```

### **COMMON SECRET TYPES**

```bash
# Database password
kubectl get secret mysql-secret -n <namespace> -o jsonpath='{.data.password}' | base64 -d

# Kafka credentials
kubectl get secret kafka-secret -n <namespace> -o jsonpath='{.data.username}' | base64 -d
kubectl get secret kafka-secret -n <namespace> -o jsonpath='{.data.password}' | base64 -d

# Redis password
kubectl get secret redis-secret -n <namespace> -o jsonpath='{.data.redis-password}' | base64 -d

# Docker registry credentials
kubectl get secret docker-registry-secret -n <namespace> -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d | jq

# TLS certificates
kubectl get secret tls-secret -n <namespace> -o jsonpath='{.data.tls\.crt}' | base64 -d
kubectl get secret tls-secret -n <namespace> -o jsonpath='{.data.tls\.key}' | base64 -d
```

### **CREATE SECRETS**

```bash
# Create generic secret from literal values
kubectl create secret generic my-secret -n <namespace> \
  --from-literal=username=admin \
  --from-literal=password=secret123

# Create secret from file
kubectl create secret generic my-secret -n <namespace> \
  --from-file=ssh-privatekey=/path/to/.ssh/id_rsa

# Create docker registry secret
kubectl create secret docker-registry docker-secret -n <namespace> \
  --docker-server=registry.example.com \
  --docker-username=user \
  --docker-password=pass \
  --docker-email=user@example.com

# Create TLS secret
kubectl create secret tls tls-secret -n <namespace> \
  --cert=path/to/tls.crt \
  --key=path/to/tls.key
```

### **UPDATE/DELETE SECRETS**

```bash
# Edit secret (will open in editor)
kubectl edit secret <secret-name> -n <namespace>

# Delete secret
kubectl delete secret <secret-name> -n <namespace>

# Replace secret
kubectl delete secret <secret-name> -n <namespace>
kubectl create secret generic <secret-name> -n <namespace> --from-literal=key=value

# Patch secret (add/update a key)
kubectl patch secret <secret-name> -n <namespace> -p '{"data":{"newkey":"'$(echo -n "newvalue" | base64)'"}}'
```

### **VERIFY SECRET IN POD**

```bash
# Check if secret is mounted in pod
kubectl exec <pod-name> -n <namespace> -- ls -la /path/to/secret

# Read mounted secret file
kubectl exec <pod-name> -n <namespace> -- cat /path/to/secret/key

# Check environment variable from secret
kubectl exec <pod-name> -n <namespace> -- env | grep SECRET_KEY

# Describe pod to see secret references
kubectl describe pod <pod-name> -n <namespace> | grep -A 5 "Environment:"
kubectl describe pod <pod-name> -n <namespace> | grep -A 5 "Mounts:"
```

---

## <a name="configmaps"></a>⚙️ **4. ConfigMaps**

### **LIST & VIEW CONFIGMAPS**

```bash
# List all configmaps
kubectl get configmap -n <namespace>
kubectl get cm -n <namespace>  # short form

# Describe configmap
kubectl describe configmap <configmap-name> -n <namespace>

# Get configmap in YAML
kubectl get configmap <configmap-name> -n <namespace> -o yaml

# Get configmap data
kubectl get configmap <configmap-name> -n <namespace> -o jsonpath='{.data}'

# Get specific key from configmap
kubectl get configmap <configmap-name> -n <namespace> -o jsonpath='{.data.<key>}'
```

### **CREATE CONFIGMAPS**

```bash
# Create from literal values
kubectl create configmap app-config -n <namespace> \
  --from-literal=log.level=DEBUG \
  --from-literal=server.port=8080

# Create from file
kubectl create configmap app-config -n <namespace> \
  --from-file=application.yml

# Create from directory
kubectl create configmap app-config -n <namespace> \
  --from-file=config/

# Create from env file
kubectl create configmap app-config -n <namespace> \
  --from-env-file=app.env
```

### **UPDATE CONFIGMAPS**

```bash
# Edit configmap
kubectl edit configmap <configmap-name> -n <namespace>

# Replace configmap from file
kubectl create configmap <configmap-name> -n <namespace> \
  --from-file=application.yml \
  --dry-run=client -o yaml | kubectl replace -f -

# Patch configmap
kubectl patch configmap <configmap-name> -n <namespace> \
  -p '{"data":{"key":"value"}}'
```

### **VERIFY CONFIGMAP IN POD**

```bash
# Check mounted configmap
kubectl exec <pod-name> -n <namespace> -- ls -la /path/to/config

# Read configmap file
kubectl exec <pod-name> -n <namespace> -- cat /path/to/config/application.yml

# Check environment from configmap
kubectl exec <pod-name> -n <namespace> -- env | grep CONFIG_KEY
```

### **SPRING BOOT SPECIFIC**

```bash
# Check application.yml in pod
kubectl exec <pod-name> -n <namespace> -- cat /config/application.yml

# Verify Spring profiles
kubectl exec <pod-name> -n <namespace> -- env | grep SPRING_PROFILES_ACTIVE
```

---

## <a name="deployments"></a>🚀 **5. Deployments & StatefulSets**

### **DEPLOYMENTS**

```bash
# List deployments
kubectl get deployments -n <namespace>
kubectl get deploy -n <namespace>  # short form

# Describe deployment
kubectl describe deployment <deployment-name> -n <namespace>

# Get deployment YAML
kubectl get deployment <deployment-name> -n <namespace> -o yaml

# Get deployment status
kubectl rollout status deployment/<deployment-name> -n <namespace>

# Get deployment history
kubectl rollout history deployment/<deployment-name> -n <namespace>

# Get specific revision details
kubectl rollout history deployment/<deployment-name> -n <namespace> --revision=2

# Scale deployment
kubectl scale deployment <deployment-name> -n <namespace> --replicas=3

# Autoscale deployment
kubectl autoscale deployment <deployment-name> -n <namespace> --min=2 --max=10 --cpu-percent=80
```

### **ROLLOUTS & UPDATES**

```bash
# Update image
kubectl set image deployment/<deployment-name> -n <namespace> \
  <container-name>=<new-image>:tag

# Restart deployment (rolling restart)
kubectl rollout restart deployment/<deployment-name> -n <namespace>

# Pause rollout
kubectl rollout pause deployment/<deployment-name> -n <namespace>

# Resume rollout
kubectl rollout resume deployment/<deployment-name> -n <namespace>

# Rollback to previous version
kubectl rollout undo deployment/<deployment-name> -n <namespace>

# Rollback to specific revision
kubectl rollout undo deployment/<deployment-name> -n <namespace> --to-revision=2
```

### **REPLICASETS**

```bash
# List replicasets
kubectl get replicasets -n <namespace>
kubectl get rs -n <namespace>  # short form

# Describe replicaset
kubectl describe rs <replicaset-name> -n <namespace>

# Get replicasets for a deployment
kubectl get rs -n <namespace> -l app=bhc
```

### **STATEFULSETS**

```bash
# List statefulsets
kubectl get statefulsets -n <namespace>
kubectl get sts -n <namespace>  # short form

# Describe statefulset
kubectl describe statefulset <statefulset-name> -n <namespace>

# Scale statefulset
kubectl scale statefulset <statefulset-name> -n <namespace> --replicas=3

# Delete statefulset pod (will recreate with same identity)
kubectl delete pod <statefulset-pod-name> -n <namespace>
```

### **DAEMONSETS**

```bash
# List daemonsets
kubectl get daemonsets -n <namespace>
kubectl get ds -n <namespace>  # short form

# Describe daemonset
kubectl describe daemonset <daemonset-name> -n <namespace>
```

### **JOBS & CRONJOBS**

```bash
# List jobs
kubectl get jobs -n <namespace>

# List cronjobs
kubectl get cronjobs -n <namespace>

# Manually trigger cronjob
kubectl create job --from=cronjob/<cronjob-name> <job-name> -n <namespace>

# Get job logs
kubectl logs job/<job-name> -n <namespace>
```

---

## <a name="networking"></a>🌐 **6. Services & Networking**

### **SERVICES**

```bash
# List services
kubectl get services -n <namespace>
kubectl get svc -n <namespace>  # short form

# Describe service
kubectl describe service <service-name> -n <namespace>

# Get service endpoints
kubectl get endpoints <service-name> -n <namespace>
kubectl get ep <service-name> -n <namespace>  # short form

# Get service YAML
kubectl get service <service-name> -n <namespace> -o yaml

# Get service cluster IP
kubectl get service <service-name> -n <namespace> -o jsonpath='{.spec.clusterIP}'

# Get service external IP (for LoadBalancer)
kubectl get service <service-name> -n <namespace> -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
```

### **TEST SERVICE CONNECTIVITY**

```bash
# Port forward service
kubectl port-forward service/<service-name> -n <namespace> 8080:80

# Test service from within cluster (run temporary pod)
kubectl run test-pod --image=busybox -it --rm -n <namespace> -- sh
# Then inside pod:
# wget -qO- http://<service-name>:<port>
# nslookup <service-name>

# Test with curl container
kubectl run curl-test --image=curlimages/curl -it --rm -n <namespace> -- sh
# Then: curl http://<service-name>:<port>

# DNS resolution test
kubectl run -it --rm debug --image=busybox --restart=Never -n <namespace> -- nslookup <service-name>
```

### **INGRESS**

```bash
# List ingress
kubectl get ingress -n <namespace>
kubectl get ing -n <namespace>  # short form

# Describe ingress
kubectl describe ingress <ingress-name> -n <namespace>

# Get ingress YAML
kubectl get ingress <ingress-name> -n <namespace> -o yaml

# Get ingress address
kubectl get ingress <ingress-name> -n <namespace> -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
```

### **NETWORK POLICIES**

```bash
# List network policies
kubectl get networkpolicies -n <namespace>
kubectl get netpol -n <namespace>  # short form

# Describe network policy
kubectl describe networkpolicy <policy-name> -n <namespace>
```

### **DNS DEBUGGING**

```bash
# Check DNS from pod
kubectl exec <pod-name> -n <namespace> -- nslookup kubernetes.default
kubectl exec <pod-name> -n <namespace> -- nslookup <service-name>
kubectl exec <pod-name> -n <namespace> -- cat /etc/resolv.conf

# Test connectivity between pods
kubectl exec <pod-name> -n <namespace> -- ping <other-pod-ip>
kubectl exec <pod-name> -n <namespace> -- telnet <service-name> <port>
kubectl exec <pod-name> -n <namespace> -- nc -zv <service-name> <port>
```

---

## <a name="monitoring"></a>📊 **7. Resource Monitoring & Performance**

### **RESOURCE USAGE**

```bash
# Get pod resource usage
kubectl top pods -n <namespace>

# Get pod resource usage sorted
kubectl top pods -n <namespace> --sort-by=cpu
kubectl top pods -n <namespace> --sort-by=memory

# Get specific pod resource usage
kubectl top pod <pod-name> -n <namespace>

# Get node resource usage
kubectl top nodes

# Get all containers resource usage
kubectl top pods -n <namespace> --containers
```

### **RESOURCE QUOTAS & LIMITS**

```bash
# List resource quotas
kubectl get resourcequotas -n <namespace>
kubectl get quota -n <namespace>  # short form

# Describe quota
kubectl describe resourcequota <quota-name> -n <namespace>

# List limit ranges
kubectl get limitranges -n <namespace>

# Describe limit range
kubectl describe limitrange <limitrange-name> -n <namespace>
```

### **HORIZONTAL POD AUTOSCALER**

```bash
# List HPA
kubectl get hpa -n <namespace>

# Describe HPA
kubectl describe hpa <hpa-name> -n <namespace>

# Get HPA metrics
kubectl get hpa <hpa-name> -n <namespace> --watch
```

### **PERSISTENT VOLUMES**

```bash
# List persistent volumes (cluster-wide)
kubectl get pv

# List persistent volume claims
kubectl get pvc -n <namespace>

# Describe PVC
kubectl describe pvc <pvc-name> -n <namespace>

# Get PVC usage
kubectl get pvc -n <namespace> -o custom-columns=NAME:.metadata.name,CAPACITY:.spec.resources.requests.storage,STATUS:.status.phase
```

### **NODES**

```bash
# List nodes
kubectl get nodes

# Describe node
kubectl describe node <node-name>

# Get node with resource info
kubectl get nodes -o wide

# Check node conditions
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.conditions[?(@.type=="Ready")].status}{"\n"}{end}'

# Cordon node (mark unschedulable)
kubectl cordon <node-name>

# Uncordon node
kubectl uncordon <node-name>

# Drain node (for maintenance)
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data
```

### **EVENTS**

```bash
# Get all events in namespace
kubectl get events -n <namespace>

# Sort events by timestamp
kubectl get events -n <namespace> --sort-by='.lastTimestamp'

# Watch events
kubectl get events -n <namespace> --watch

# Get events for specific resource
kubectl get events -n <namespace> --field-selector involvedObject.name=<pod-name>

# Get warning events only
kubectl get events -n <namespace> --field-selector type=Warning
```

---

## <a name="spring-boot"></a>☕ **8. Spring Boot Specific Debugging**

### **SPRING BOOT ACTUATOR**

```bash
# Port forward to access actuator endpoints
kubectl port-forward <pod-name> -n <namespace> 8080:8080

# Then access locally:
# curl http://localhost:8080/actuator
# curl http://localhost:8080/actuator/health
# curl http://localhost:8080/actuator/info
# curl http://localhost:8080/actuator/metrics
# curl http://localhost:8080/actuator/env
# curl http://localhost:8080/actuator/loggers

# Check health from inside pod
kubectl exec <pod-name> -n <namespace> -- wget -qO- http://localhost:8080/actuator/health
kubectl exec <pod-name> -n <namespace> -- curl http://localhost:8080/actuator/health
```

### **ENVIRONMENT & CONFIGURATION**

```bash
# Check Spring profile
kubectl exec <pod-name> -n <namespace> -- env | grep SPRING_PROFILES_ACTIVE

# Check all Spring environment variables
kubectl exec <pod-name> -n <namespace> -- env | grep SPRING_

# Check Java version
kubectl exec <pod-name> -n <namespace> -- java -version

# Check Java options
kubectl exec <pod-name> -n <namespace> -- env | grep JAVA_OPTS
kubectl exec <pod-name> -n <namespace> -- env | grep JVM_OPTS

# Check application.yml/properties location
kubectl exec <pod-name> -n <namespace> -- ls -la /config/
kubectl exec <pod-name> -n <namespace> -- cat /config/application.yml
```

### **DATABASE CONNECTION POOL (HIKARICP)**

```bash
# Check Hikari connection pool logs
kubectl logs <pod-name> -n <namespace> | grep -i hikari
kubectl logs <pod-name> -n <namespace> | grep "HikariPool"

# Check database connection errors
kubectl logs <pod-name> -n <namespace> | grep -i "connection"
kubectl logs <pod-name> -n <namespace> | grep -i "datasource"
kubectl logs <pod-name> -n <namespace> | grep "CommunicationsException"
kubectl logs <pod-name> -n <namespace> | grep "SQLException"
```

### **MEMORY & GC ANALYSIS**

```bash
# Check for OOM errors
kubectl logs <pod-name> -n <namespace> | grep "OutOfMemoryError"
kubectl logs <pod-name> -n <namespace> --previous | grep "OutOfMemoryError"

# Check GC logs
kubectl logs <pod-name> -n <namespace> | grep -i "GC"

# Get heap dump (if configured)
kubectl exec <pod-name> -n <namespace> -- jmap -dump:format=b,file=/tmp/heap.hprof 1
kubectl cp <namespace>/<pod-name>:/tmp/heap.hprof ./heap.hprof

# Check memory usage in pod
kubectl exec <pod-name> -n <namespace> -- ps aux
kubectl exec <pod-name> -n <namespace> -- free -m
kubectl top pod <pod-name> -n <namespace>
```

### **THREAD DUMPS**

```bash
# Get thread dump
kubectl exec <pod-name> -n <namespace> -- jstack 1

# Multiple thread dumps for deadlock analysis
kubectl exec <pod-name> -n <namespace> -- sh -c 'for i in 1 2 3; do jstack 1; sleep 5; done'
```

### **STARTUP & SHUTDOWN**

```bash
# Check startup time
kubectl logs <pod-name> -n <namespace> | grep "Started BhcApplication in"

# Check for startup failures
kubectl logs <pod-name> -n <namespace> | grep "APPLICATION FAILED TO START"

# Check readiness/liveness probe failures
kubectl describe pod <pod-name> -n <namespace> | grep -A 10 "Liveness\|Readiness"

# Check graceful shutdown
kubectl logs <pod-name> -n <namespace> | grep "Shutdown"
```

---

## <a name="kafka"></a>📨 **9. Kafka Debugging**

### **KAFKA CONNECTION**

```bash
# Check Kafka connection in logs
kubectl logs <pod-name> -n <namespace> | grep -i kafka
kubectl logs <pod-name> -n <namespace> | grep "kafka.bootstrap-servers"
kubectl logs <pod-name> -n <namespace> | grep "ConsumerConfig"
kubectl logs <pod-name> -n <namespace> | grep "ProducerConfig"

# Check Kafka environment variables
kubectl exec <pod-name> -n <namespace> -- env | grep KAFKA
kubectl exec <pod-name> -n <namespace> -- env | grep SPRING_KAFKA
```

### **CONSUMER DEBUGGING**

```bash
# Check consumer group logs
kubectl logs <pod-name> -n <namespace> | grep "consumer group"
kubectl logs <pod-name> -n <namespace> | grep "ConsumerCoordinator"
kubectl logs <pod-name> -n <namespace> | grep "Subscribed to topic"

# Check consumer errors
kubectl logs <pod-name> -n <namespace> | grep "consumer.*error" -i
kubectl logs <pod-name> -n <namespace> | grep "Failed to process"
kubectl logs <pod-name> -n <namespace> | grep "offset"

# Check rebalancing
kubectl logs <pod-name> -n <namespace> | grep "rebalance"
kubectl logs <pod-name> -n <namespace> | grep "Revoking"
kubectl logs <pod-name> -n <namespace> | grep "partitions assigned"
```

### **PRODUCER DEBUGGING**

```bash
# Check producer logs
kubectl logs <pod-name> -n <namespace> | grep "ProducerConfig"
kubectl logs <pod-name> -n <namespace> | grep "producer"

# Check send failures
kubectl logs <pod-name> -n <namespace> | grep "Failed to send"
kubectl logs <pod-name> -n <namespace> | grep "TimeoutException"
```

### **KAFKA POD ACCESS (if Kafka in same cluster)**

```bash
# Get Kafka pods
kubectl get pods -n <kafka-namespace> -l app=kafka

# Access Kafka pod
kubectl exec -it <kafka-pod-name> -n <kafka-namespace> -- bash

# Inside Kafka pod - list topics
kafka-topics.sh --bootstrap-server localhost:9092 --list

# Describe consumer groups
kafka-consumer-groups.sh --bootstrap-server localhost:9092 --describe --group <group-id>

# Check topic details
kafka-topics.sh --bootstrap-server localhost:9092 --describe --topic <topic-name>

# Consume messages (test)
kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic <topic-name> --from-beginning

# Produce test message
kafka-console-producer.sh --bootstrap-server localhost:9092 --topic <topic-name>

# Check consumer lag
kafka-consumer-groups.sh --bootstrap-server localhost:9092 --describe --group <group-id> | grep LAG
```

### **KAFKA CONNECTIVITY TEST FROM APP POD**

```bash
# Test Kafka connectivity
kubectl exec <pod-name> -n <namespace> -- nc -zv <kafka-service> 9092
kubectl exec <pod-name> -n <namespace> -- telnet <kafka-service> 9092

# Check DNS resolution for Kafka
kubectl exec <pod-name> -n <namespace> -- nslookup <kafka-service>
```

---

## <a name="redis"></a>🔴 **10. Redis Debugging**

### **REDIS CONNECTION**

```bash
# Check Redis connection in logs
kubectl logs <pod-name> -n <namespace> | grep -i redis
kubectl logs <pod-name> -n <namespace> | grep "lettuce"
kubectl logs <pod-name> -n <namespace> | grep "RedisConnectionFactory"

# Check Redis environment variables
kubectl exec <pod-name> -n <namespace> -- env | grep REDIS
kubectl exec <pod-name> -n <namespace> -- env | grep SPRING_REDIS
```

### **REDIS POD ACCESS**

```bash
# Get Redis pods
kubectl get pods -n <redis-namespace> -l app=redis

# Access Redis pod
kubectl exec -it <redis-pod-name> -n <redis-namespace> -- bash

# Connect to Redis CLI (without password)
kubectl exec -it <redis-pod-name> -n <redis-namespace> -- redis-cli

# Connect to Redis CLI (with password)
kubectl exec -it <redis-pod-name> -n <redis-namespace> -- redis-cli -a <password>

# Using secret for password
REDIS_PASSWORD=$(kubectl get secret redis-secret -n <namespace> -o jsonpath='{.data.redis-password}' | base64 -d)
kubectl exec -it <redis-pod-name> -n <redis-namespace> -- redis-cli -a $REDIS_PASSWORD
```

### **REDIS COMMANDS (inside redis-cli)**

```bash
# Inside redis-cli:
PING                          # Test connection
INFO                          # Server info
INFO stats                    # Statistics
INFO memory                   # Memory usage
DBSIZE                        # Number of keys
KEYS *                        # List all keys (use with caution in prod)
GET <key>                     # Get value
TTL <key>                     # Time to live
CLIENT LIST                   # Connected clients
MONITOR                       # Real-time command monitoring
SLOWLOG GET 10                # Slow queries
```

### **REDIS CONNECTIVITY TEST FROM APP POD**

```bash
# Test Redis connectivity
kubectl exec <pod-name> -n <namespace> -- nc -zv <redis-service> 6379
kubectl exec <pod-name> -n <namespace> -- telnet <redis-service> 6379

# Check DNS resolution
kubectl exec <pod-name> -n <namespace> -- nslookup <redis-service>

# Test with redis-cli from app pod (if available)
kubectl exec <pod-name> -n <namespace> -- redis-cli -h <redis-service> -p 6379 PING
```

### **REDIS PERFORMANCE**

```bash
# Check Redis memory usage
kubectl exec <redis-pod-name> -n <redis-namespace> -- redis-cli INFO memory

# Check Redis CPU usage
kubectl top pod <redis-pod-name> -n <redis-namespace>

# Monitor Redis commands in real-time
kubectl exec -it <redis-pod-name> -n <redis-namespace> -- redis-cli MONITOR
```

### **REDIS ERRORS IN APPLICATION**

```bash
# Check connection errors
kubectl logs <pod-name> -n <namespace> | grep "redis.*error" -i
kubectl logs <pod-name> -n <namespace> | grep "RedisConnectionFailure"
kubectl logs <pod-name> -n <namespace> | grep "Could not connect to Redis"
kubectl logs <pod-name> -n <namespace> | grep "connection refused" -i
```

---

## <a name="mysql"></a>🐬 **11. MySQL Debugging**

### **MYSQL CONNECTION**

```bash
# Check MySQL connection in logs
kubectl logs <pod-name> -n <namespace> | grep -i mysql
kubectl logs <pod-name> -n <namespace> | grep -i jdbc
kubectl logs <pod-name> -n <namespace> | grep "datasource"

# Check database environment variables
kubectl exec <pod-name> -n <namespace> -- env | grep DB_
kubectl exec <pod-name> -n <namespace> -- env | grep MYSQL
kubectl exec <pod-name> -n <namespace> -- env | grep SPRING_DATASOURCE

# Check connection pool
kubectl logs <pod-name> -n <namespace> | grep "HikariPool"
kubectl logs <pod-name> -n <namespace> | grep "connection.*pool" -i
```

### **MYSQL POD ACCESS**

```bash
# Get MySQL pods
kubectl get pods -n <mysql-namespace> -l app=mysql

# Access MySQL pod
kubectl exec -it <mysql-pod-name> -n <mysql-namespace> -- bash

# Connect to MySQL (from pod)
kubectl exec -it <mysql-pod-name> -n <mysql-namespace> -- mysql -u root -p

# Using secret for password
MYSQL_PASSWORD=$(kubectl get secret mysql-secret -n <namespace> -o jsonpath='{.data.password}' | base64 -d)
kubectl exec -it <mysql-pod-name> -n <mysql-namespace> -- mysql -u root -p$MYSQL_PASSWORD

# One-liner to connect
kubectl exec -it <mysql-pod-name> -n <mysql-namespace> -- mysql -u root -p$(kubectl get secret mysql-secret -n <namespace> -o jsonpath='{.data.password}' | base64 -d)
```

### **MYSQL COMMANDS (inside mysql shell)**

```sql
-- Inside MySQL shell:
SHOW DATABASES;
USE <database_name>;
SHOW TABLES;
SHOW PROCESSLIST;                          -- Active connections
SHOW STATUS;                               -- Server status
SHOW VARIABLES LIKE 'max_connections';     -- Connection limit
SELECT * FROM information_schema.processlist;
SHOW TABLE STATUS;                         -- Table sizes and info

-- Check slow queries
SHOW VARIABLES LIKE 'slow_query_log';
SELECT * FROM mysql.slow_log LIMIT 10;
```

### **MYSQL CONNECTIVITY TEST FROM APP POD**

```bash
# Test MySQL connectivity
kubectl exec <pod-name> -n <namespace> -- nc -zv <mysql-service> 3306
kubectl exec <pod-name> -n <namespace> -- telnet <mysql-service> 3306

# Check DNS resolution
kubectl exec <pod-name> -n <namespace> -- nslookup <mysql-service>

# Test with mysql client from app pod (if available)
kubectl exec <pod-name> -n <namespace> -- mysql -h <mysql-service> -u <user> -p<password> -e "SELECT 1"
```

### **MYSQL ERRORS IN APPLICATION**

```bash
# Check connection errors
kubectl logs <pod-name> -n <namespace> | grep "SQLException"
kubectl logs <pod-name> -n <namespace> | grep "CommunicationsException"
kubectl logs <pod-name> -n <namespace> | grep "Connection refused"
kubectl logs <pod-name> -n <namespace> | grep "Access denied"

# Check connection pool exhaustion
kubectl logs <pod-name> -n <namespace> | grep "Connection is not available"
kubectl logs <pod-name> -n <namespace> | grep "Timeout.*connection" -i

# Check slow queries impact
kubectl logs <pod-name> -n <namespace> | grep "slow" -i
```

### **MYSQL PERFORMANCE**

```bash
# Check MySQL resource usage
kubectl top pod <mysql-pod-name> -n <mysql-namespace>

# Check MySQL logs
kubectl logs <mysql-pod-name> -n <mysql-namespace>
kubectl logs <mysql-pod-name> -n <mysql-namespace> | grep -i error
kubectl logs <mysql-pod-name> -n <mysql-namespace> | grep -i warning

# Copy MySQL logs
kubectl cp <mysql-namespace>/<mysql-pod-name>:/var/log/mysql/error.log ./mysql-error.log
```

### **DATABASE MIGRATIONS**

```bash
# Check Flyway/Liquibase migration logs
kubectl logs <pod-name> -n <namespace> | grep -i flyway
kubectl logs <pod-name> -n <namespace> | grep -i liquibase
kubectl logs <pod-name> -n <namespace> | grep "migration"
kubectl logs <pod-name> -n <namespace> | grep "schema"
```

---

## <a name="troubleshooting"></a>🔧 **12. Common Troubleshooting Scenarios**

### **POD CRASHLOOPBACKOFF**

```bash
# Get pod status
kubectl get pod <pod-name> -n <namespace>

# Check events
kubectl describe pod <pod-name> -n <namespace> | grep -A 20 Events

# Get current logs
kubectl logs <pod-name> -n <namespace>

# Get previous logs (from crashed container)
kubectl logs <pod-name> -n <namespace> --previous

# Check liveness/readiness probes
kubectl describe pod <pod-name> -n <namespace> | grep -A 5 "Liveness\|Readiness"

# Check resource limits
kubectl describe pod <pod-name> -n <namespace> | grep -A 10 "Limits\|Requests"
```

### **POD PENDING / NOT SCHEDULING**

```bash
# Check why pod is pending
kubectl describe pod <pod-name> -n <namespace> | grep -A 20 Events

# Check node resources
kubectl describe nodes | grep -A 5 "Allocated resources"

# Check PVC status
kubectl get pvc -n <namespace>

# Check pod affinity/anti-affinity
kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.spec.affinity}'
```

### **IMAGE PULL ERRORS**

```bash
# Check image pull status
kubectl describe pod <pod-name> -n <namespace> | grep -A 10 "Failed.*pull"

# Verify image exists
kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.spec.containers[*].image}'

# Check imagePullSecrets
kubectl describe pod <pod-name> -n <namespace> | grep "ImagePullSecrets"

# Verify docker registry secret
kubectl get secret <docker-registry-secret> -n <namespace>
```

### **SERVICE NOT ACCESSIBLE**

```bash
# Check service exists and has endpoints
kubectl get service <service-name> -n <namespace>
kubectl get endpoints <service-name> -n <namespace>

# Check if pods are ready
kubectl get pods -l app=<label> -n <namespace>

# Check service selector matches pod labels
kubectl describe service <service-name> -n <namespace> | grep Selector
kubectl get pods -n <namespace> --show-labels

# Test from debug pod
kubectl run debug --image=curlimages/curl -it --rm -n <namespace> -- curl http://<service-name>:<port>
```

### **CONFIGURATION ISSUES**

```bash
# Compare running config with deployment
kubectl get deployment <deployment-name> -n <namespace> -o yaml > deployment.yaml
kubectl get pod <pod-name> -n <namespace> -o yaml > pod.yaml

# Check environment variables
kubectl exec <pod-name> -n <namespace> -- env | sort

# Verify configmap/secret mounting
kubectl describe pod <pod-name> -n <namespace> | grep -A 20 "Mounts:"

# Check for missing environment variables
kubectl logs <pod-name> -n <namespace> | grep "null\|undefined\|not set"
```

### **PERFORMANCE ISSUES**

```bash
# Check resource usage
kubectl top pods -n <namespace>
kubectl top nodes

# Check for resource throttling
kubectl describe pod <pod-name> -n <namespace> | grep -i throttl

# Check HPA status
kubectl get hpa -n <namespace>

# Check pod distribution across nodes
kubectl get pods -n <namespace> -o wide
```

### **DATABASE CONNECTION ISSUES**

```bash
# Full diagnostic
kubectl exec <pod-name> -n <namespace> -- nc -zv <mysql-service> 3306
kubectl logs <pod-name> -n <namespace> | grep -i "connection\|datasource\|hikari"
kubectl exec <pod-name> -n <namespace> -- env | grep -i "db\|mysql\|datasource"
```

### **KAFKA CONNECTION ISSUES**

```bash
# Full diagnostic
kubectl exec <pod-name> -n <namespace> -- nc -zv <kafka-service> 9092
kubectl logs <pod-name> -n <namespace> | grep -i "kafka\|consumer\|producer"
kubectl exec <pod-name> -n <namespace> -- env | grep KAFKA
```

### **MEMORY LEAKS / OOM**

```bash
# Check for OOM kills
kubectl describe pod <pod-name> -n <namespace> | grep -i "oomkilled"

# Check memory limits
kubectl describe pod <pod-name> -n <namespace> | grep -A 5 "Limits:"

# Get heap dump
kubectl exec <pod-name> -n <namespace> -- jmap -dump:format=b,file=/tmp/heap.hprof 1
kubectl cp <namespace>/<pod-name>:/tmp/heap.hprof ./heap.hprof

# Check previous pod logs for OOM
kubectl logs <pod-name> -n <namespace> --previous | grep "OutOfMemoryError"
```

### **QUICK HEALTH CHECK COMMAND**

```bash
# Comprehensive health check
kubectl get pods -n <namespace> && \
kubectl get svc -n <namespace> && \
kubectl get deployments -n <namespace> && \
kubectl top pods -n <namespace> && \
kubectl get events -n <namespace> --sort-by='.lastTimestamp' | tail -20
```

---

## 🎯 **Useful Aliases**

Add these to your `~/.zshrc` or `~/.bashrc` for faster debugging:

```bash
alias k='kubectl'
alias kgp='kubectl get pods'
alias kgs='kubectl get svc'
alias kgd='kubectl get deployments'
alias kdp='kubectl describe pod'
alias kds='kubectl describe service'
alias kl='kubectl logs'
alias klf='kubectl logs -f'
alias kex='kubectl exec -it'
alias kpf='kubectl port-forward'
alias kgpw='kubectl get pods --watch'
alias kge='kubectl get events --sort-by=.metadata.creationTimestamp'
alias ktp='kubectl top pods'
alias ktn='kubectl top nodes'

# With namespace
alias kgpn='kubectl get pods -n'
alias kln='kubectl logs -n'
alias kdn='kubectl describe -n'
```

---

## 📚 **Advanced Tips**

### **JSONPATH QUERIES**

```bash
# Get pod IPs
kubectl get pods -n <namespace> -o jsonpath='{.items[*].status.podIP}'

# Get pod restart counts
kubectl get pods -n <namespace> -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.containerStatuses[0].restartCount}{"\n"}{end}'

# Get container images
kubectl get pods -n <namespace> -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[*].image}{"\n"}{end}'
```

### **WATCH & MONITORING**

```bash
# Watch multiple resources
watch 'kubectl get pods,svc,deploy -n <namespace>'

# Continuous monitoring
while true; do clear; kubectl get pods -n <namespace>; kubectl top pods -n <namespace>; sleep 5; done
```

### **BATCH OPERATIONS**

```bash
# Delete all pods with label
kubectl delete pods -l app=bhc -n <namespace>

# Restart all deployments
kubectl get deployments -n <namespace> -o name | xargs -n1 kubectl rollout restart -n <namespace>

# Get logs from all pods
kubectl logs -l app=bhc -n <namespace> --all-containers=true
```

### **DEBUG CONTAINERS (Kubernetes 1.23+)**

```bash
# Create ephemeral debug container
kubectl debug <pod-name> -n <namespace> -it --image=busybox

# Debug with different image
kubectl debug <pod-name> -n <namespace> -it --image=ubuntu --target=<container-name>
```

---

## 📖 **Quick Reference Commands for BHC Application**

```bash
# Replace <namespace> with your actual namespace (e.g., production, staging, etc.)

# Check application health
kubectl get pods -n <namespace> -l app=bhc
kubectl logs -f <bhc-pod-name> -n <namespace> --tail=100

# Check all dependencies status
kubectl get pods -n <namespace> | grep -E 'bhc|kafka|redis|mysql'

# Quick troubleshooting
kubectl describe pod <bhc-pod-name> -n <namespace>
kubectl logs <bhc-pod-name> -n <namespace> --previous

# Access actuator endpoints
kubectl port-forward <bhc-pod-name> -n <namespace> 8080:8080
# Then: curl http://localhost:8080/actuator/health

# Check database connection
kubectl exec <bhc-pod-name> -n <namespace> -- nc -zv <mysql-service> 3306

# Check Kafka connection
kubectl exec <bhc-pod-name> -n <namespace> -- nc -zv <kafka-service> 9092

# Check Redis connection
kubectl exec <bhc-pod-name> -n <namespace> -- nc -zv <redis-service> 6379
```

---

**Created for BHC (Balance History Collector) Project**
*Last Updated: November 18, 2025*

