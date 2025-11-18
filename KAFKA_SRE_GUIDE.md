# Kafka SRE Operations & Debugging Guide

A comprehensive guide for Site Reliability Engineers working with Apache Kafka - covering daily operations, debugging, monitoring, and incident response.

## 📋 **Table of Contents**

1. [Daily Health Checks](#daily-health)
2. [Kafka Cluster Operations](#cluster-ops)
3. [Topic Management](#topics)
4. [Consumer Group Management](#consumer-groups)
5. [Producer Monitoring](#producers)
6. [Lag Monitoring & Analysis](#lag-monitoring)
7. [Performance Troubleshooting](#performance)
8. [Incident Response Playbook](#incidents)
9. [Data Management](#data-management)
10. [Security & ACLs](#security)
11. [Monitoring & Alerting](#monitoring)
12. [Backup & Recovery](#backup)
13. [Kubernetes-Specific Operations](#kubernetes)
14. [Common Issues & Solutions](#common-issues)
15. [Useful Scripts](#scripts)

---

## <a name="daily-health"></a>🏥 **1. Daily Health Checks**

### **Quick Health Check Script**

```bash
#!/bin/bash
# Daily Kafka Health Check

NAMESPACE="kafka"
BOOTSTRAP_SERVER="kafka-service:9092"

echo "=== Kafka Daily Health Check ==="
echo "Date: $(date)"
echo ""

# 1. Check Kafka pods
echo "1. Checking Kafka Pods..."
kubectl get pods -n $NAMESPACE -l app=kafka

# 2. Check broker status
echo -e "\n2. Checking Broker Status..."
kubectl exec -n $NAMESPACE kafka-0 -- kafka-broker-api-versions.sh \
  --bootstrap-server $BOOTSTRAP_SERVER 2>/dev/null | grep -c "ApiVersion" && echo "✓ Brokers responding"

# 3. Check under-replicated partitions
echo -e "\n3. Checking Under-Replicated Partitions..."
kubectl exec -n $NAMESPACE kafka-0 -- kafka-topics.sh \
  --bootstrap-server $BOOTSTRAP_SERVER \
  --describe \
  --under-replicated-partitions

# 4. Check consumer lag
echo -e "\n4. Checking Consumer Lag..."
kubectl exec -n $NAMESPACE kafka-0 -- kafka-consumer-groups.sh \
  --bootstrap-server $BOOTSTRAP_SERVER \
  --all-groups \
  --describe | grep -i lag

# 5. Check disk usage
echo -e "\n5. Checking Disk Usage..."
kubectl exec -n $NAMESPACE kafka-0 -- df -h /var/lib/kafka/data

echo -e "\n=== Health Check Complete ==="
```

### **Core Health Checks**

```bash
# Check if Kafka brokers are running
kubectl get pods -n kafka -l app=kafka

# Check broker logs for errors
kubectl logs kafka-0 -n kafka | grep -i "error\|exception\|warn" | tail -50

# Test broker connectivity
kubectl exec -n kafka kafka-0 -- kafka-broker-api-versions.sh \
  --bootstrap-server localhost:9092

# Check ZooKeeper health (if using ZooKeeper mode)
kubectl exec -n kafka zookeeper-0 -- zkServer.sh status

# Check cluster metadata
kubectl exec -n kafka kafka-0 -- kafka-metadata.sh \
  --snapshot /tmp/metadata-snapshot \
  --print

# Verify broker configuration
kubectl exec -n kafka kafka-0 -- kafka-configs.sh \
  --bootstrap-server localhost:9092 \
  --entity-type brokers \
  --entity-default \
  --describe
```

### **Daily Metrics to Monitor**

```bash
# Number of active controllers (should be exactly 1)
kubectl exec -n kafka kafka-0 -- kafka-metadata.sh \
  --bootstrap-server localhost:9092 \
  --describe --entity-type broker-loggers

# Check partition distribution
kubectl exec -n kafka kafka-0 -- kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --describe | grep "Leader:"

# Check ISR (In-Sync Replicas)
kubectl exec -n kafka kafka-0 -- kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --describe | grep "Isr:"

# Check total topics and partitions
kubectl exec -n kafka kafka-0 -- kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --list | wc -l
```

---

## <a name="cluster-ops"></a>🖥️ **2. Kafka Cluster Operations**

### **Broker Management**

```bash
# List all brokers
kubectl exec -n kafka kafka-0 -- kafka-broker-api-versions.sh \
  --bootstrap-server localhost:9092 | grep -E "^[0-9]+" | awk '{print $1}' | sort -u

# Check broker details
kubectl exec -n kafka kafka-0 -- kafka-metadata.sh \
  --bootstrap-server localhost:9092 \
  --describe --entity-type brokers

# Get broker configuration
kubectl exec -n kafka kafka-0 -- kafka-configs.sh \
  --bootstrap-server localhost:9092 \
  --entity-type brokers \
  --entity-name 0 \
  --describe

# Update broker configuration (dynamic)
kubectl exec -n kafka kafka-0 -- kafka-configs.sh \
  --bootstrap-server localhost:9092 \
  --entity-type brokers \
  --entity-name 0 \
  --alter \
  --add-config log.retention.hours=168

# Remove configuration override
kubectl exec -n kafka kafka-0 -- kafka-configs.sh \
  --bootstrap-server localhost:9092 \
  --entity-type brokers \
  --entity-name 0 \
  --alter \
  --delete-config log.retention.hours
```

### **Cluster Metadata**

```bash
# View cluster ID
kubectl exec -n kafka kafka-0 -- kafka-cluster.sh cluster-id \
  --bootstrap-server localhost:9092

# Describe cluster
kubectl exec -n kafka kafka-0 -- kafka-metadata.sh \
  --bootstrap-server localhost:9092 \
  --describe

# Check controller
kubectl logs kafka-0 -n kafka | grep "Elected as controller"

# View all broker configurations
for i in 0 1 2; do
  echo "=== Broker $i ==="
  kubectl exec -n kafka kafka-$i -- cat /opt/kafka/config/server.properties | grep -v "^#" | grep -v "^$"
done
```

### **Rolling Restart**

```bash
# Graceful rolling restart of Kafka cluster
for i in 0 1 2; do
  echo "Restarting kafka-$i..."
  kubectl delete pod kafka-$i -n kafka
  
  # Wait for pod to be ready
  kubectl wait --for=condition=ready pod/kafka-$i -n kafka --timeout=300s
  
  # Wait for ISR to stabilize
  sleep 30
  
  # Check under-replicated partitions
  kubectl exec -n kafka kafka-0 -- kafka-topics.sh \
    --bootstrap-server localhost:9092 \
    --describe \
    --under-replicated-partitions
  
  echo "kafka-$i restarted successfully"
done
```

### **Scaling Cluster**

```bash
# Scale up Kafka StatefulSet
kubectl scale statefulset kafka -n kafka --replicas=5

# Wait for new brokers to be ready
kubectl wait --for=condition=ready pod/kafka-3 -n kafka --timeout=300s
kubectl wait --for=condition=ready pod/kafka-4 -n kafka --timeout=300s

# Verify new brokers joined cluster
kubectl exec -n kafka kafka-0 -- kafka-broker-api-versions.sh \
  --bootstrap-server localhost:9092 | grep -E "^[0-9]+"

# Reassign partitions to new brokers (create reassignment plan)
kubectl exec -n kafka kafka-0 -- kafka-reassign-partitions.sh \
  --bootstrap-server localhost:9092 \
  --generate \
  --topics-to-move-json-file /tmp/topics-to-move.json \
  --broker-list "0,1,2,3,4"
```

---

## <a name="topics"></a>📂 **3. Topic Management**

### **List and Describe Topics**

```bash
# List all topics
kubectl exec -n kafka kafka-0 -- kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --list

# List topics with details
kubectl exec -n kafka kafka-0 -- kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --list \
  --exclude-internal

# Describe specific topic
kubectl exec -n kafka kafka-0 -- kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --describe \
  --topic my-topic

# Describe all topics
kubectl exec -n kafka kafka-0 -- kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --describe

# Get topic configuration
kubectl exec -n kafka kafka-0 -- kafka-configs.sh \
  --bootstrap-server localhost:9092 \
  --entity-type topics \
  --entity-name my-topic \
  --describe

# List topics with partition count
kubectl exec -n kafka kafka-0 -- kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --describe | grep -E "Topic:|PartitionCount:"
```

### **Create Topics**

```bash
# Create simple topic
kubectl exec -n kafka kafka-0 -- kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --create \
  --topic my-new-topic \
  --partitions 3 \
  --replication-factor 2

# Create topic with specific configuration
kubectl exec -n kafka kafka-0 -- kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --create \
  --topic my-topic \
  --partitions 6 \
  --replication-factor 3 \
  --config retention.ms=86400000 \
  --config segment.ms=3600000 \
  --config compression.type=snappy

# Create compacted topic (for changelog/state)
kubectl exec -n kafka kafka-0 -- kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --create \
  --topic user-profiles \
  --partitions 12 \
  --replication-factor 3 \
  --config cleanup.policy=compact \
  --config min.compaction.lag.ms=0 \
  --config delete.retention.ms=86400000
```

### **Modify Topics**

```bash
# Increase partitions (cannot decrease!)
kubectl exec -n kafka kafka-0 -- kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --alter \
  --topic my-topic \
  --partitions 12

# Update topic configuration
kubectl exec -n kafka kafka-0 -- kafka-configs.sh \
  --bootstrap-server localhost:9092 \
  --entity-type topics \
  --entity-name my-topic \
  --alter \
  --add-config retention.ms=604800000

# Update multiple configurations
kubectl exec -n kafka kafka-0 -- kafka-configs.sh \
  --bootstrap-server localhost:9092 \
  --entity-type topics \
  --entity-name my-topic \
  --alter \
  --add-config retention.ms=604800000,segment.ms=3600000,compression.type=lz4

# Remove configuration (revert to default)
kubectl exec -n kafka kafka-0 -- kafka-configs.sh \
  --bootstrap-server localhost:9092 \
  --entity-type topics \
  --entity-name my-topic \
  --alter \
  --delete-config retention.ms
```

### **Delete Topics**

```bash
# Delete a topic
kubectl exec -n kafka kafka-0 -- kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --delete \
  --topic my-topic

# Verify deletion
kubectl exec -n kafka kafka-0 -- kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --list | grep my-topic

# Force delete topic (if stuck) - requires Kafka restart
# WARNING: Use with caution!
kubectl exec -n kafka kafka-0 -- rm -rf /var/lib/kafka/data/my-topic-*
```

### **Topic Inspection**

```bash
# Check topic size
kubectl exec -n kafka kafka-0 -- kafka-log-dirs.sh \
  --bootstrap-server localhost:9092 \
  --describe \
  --topic-list my-topic

# Get earliest and latest offsets
kubectl exec -n kafka kafka-0 -- kafka-run-class.sh kafka.tools.GetOffsetShell \
  --broker-list localhost:9092 \
  --topic my-topic \
  --time -2  # -2 for earliest

kubectl exec -n kafka kafka-0 -- kafka-run-class.sh kafka.tools.GetOffsetShell \
  --broker-list localhost:9092 \
  --topic my-topic \
  --time -1  # -1 for latest

# Count messages in topic
kubectl exec -n kafka kafka-0 -- kafka-run-class.sh kafka.tools.GetOffsetShell \
  --broker-list localhost:9092 \
  --topic my-topic | awk -F ":" '{sum += $3} END {print sum}'

# Check partition distribution
kubectl exec -n kafka kafka-0 -- kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --describe \
  --topic my-topic | grep "Leader:"
```

---

## <a name="consumer-groups"></a>👥 **4. Consumer Group Management**

### **List and Describe Consumer Groups**

```bash
# List all consumer groups
kubectl exec -n kafka kafka-0 -- kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --list

# Describe specific consumer group
kubectl exec -n kafka kafka-0 -- kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --group my-consumer-group \
  --describe

# Describe all consumer groups
kubectl exec -n kafka kafka-0 -- kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --all-groups \
  --describe

# Get consumer group state
kubectl exec -n kafka kafka-0 -- kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --group my-consumer-group \
  --state

# Get consumer group members
kubectl exec -n kafka kafka-0 -- kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --group my-consumer-group \
  --members

# Get detailed member information
kubectl exec -n kafka kafka-0 -- kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --group my-consumer-group \
  --members \
  --verbose
```

### **Consumer Group Lag Analysis**

```bash
# Check lag for specific consumer group
kubectl exec -n kafka kafka-0 -- kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --group my-consumer-group \
  --describe | grep LAG

# Check lag for all groups
kubectl exec -n kafka kafka-0 -- kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --all-groups \
  --describe | grep -v "PARTITION\|^$" | awk '{if($6>0)print}'

# Get total lag per consumer group
kubectl exec -n kafka kafka-0 -- kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --group my-consumer-group \
  --describe | awk 'NR>1 {sum += $5} END {print "Total Lag:", sum}'

# Monitor lag in real-time
watch -n 5 'kubectl exec -n kafka kafka-0 -- kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --group my-consumer-group \
  --describe'

# Find consumer groups with high lag (>10000)
kubectl exec -n kafka kafka-0 -- kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --all-groups \
  --describe | awk '$6 > 10000 {print $1, $2, $3, $6}'
```

### **Reset Consumer Group Offsets**

```bash
# Reset to earliest offset (reprocess all messages)
kubectl exec -n kafka kafka-0 -- kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --group my-consumer-group \
  --topic my-topic \
  --reset-offsets \
  --to-earliest \
  --execute

# Reset to latest offset (skip all current messages)
kubectl exec -n kafka kafka-0 -- kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --group my-consumer-group \
  --topic my-topic \
  --reset-offsets \
  --to-latest \
  --execute

# Reset to specific offset
kubectl exec -n kafka kafka-0 -- kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --group my-consumer-group \
  --topic my-topic:0 \
  --reset-offsets \
  --to-offset 1000 \
  --execute

# Reset to specific datetime
kubectl exec -n kafka kafka-0 -- kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --group my-consumer-group \
  --topic my-topic \
  --reset-offsets \
  --to-datetime 2025-11-18T10:00:00.000 \
  --execute

# Shift offset by N (negative to replay, positive to skip)
kubectl exec -n kafka kafka-0 -- kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --group my-consumer-group \
  --topic my-topic \
  --reset-offsets \
  --shift-by -1000 \
  --execute

# Reset by duration (go back 2 hours)
kubectl exec -n kafka kafka-0 -- kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --group my-consumer-group \
  --topic my-topic \
  --reset-offsets \
  --by-duration PT2H \
  --execute

# Dry-run before executing (recommended!)
kubectl exec -n kafka kafka-0 -- kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --group my-consumer-group \
  --topic my-topic \
  --reset-offsets \
  --to-earliest \
  --dry-run
```

### **Delete Consumer Group**

```bash
# Delete consumer group (must be inactive)
kubectl exec -n kafka kafka-0 -- kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --group my-consumer-group \
  --delete

# Force delete all groups matching pattern (use with caution!)
kubectl exec -n kafka kafka-0 -- kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --list | grep "test-" | while read group; do
    kubectl exec -n kafka kafka-0 -- kafka-consumer-groups.sh \
      --bootstrap-server localhost:9092 \
      --group $group \
      --delete
done
```

### **Export/Import Offsets**

```bash
# Export current offsets to file
kubectl exec -n kafka kafka-0 -- kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --group my-consumer-group \
  --describe > consumer-offsets-backup.txt

# Create offset reset file
cat > offsets-to-reset.csv << EOF
my-topic,0,1000
my-topic,1,1000
my-topic,2,1000
EOF

# Import offsets from CSV
kubectl cp offsets-to-reset.csv kafka/kafka-0:/tmp/offsets.csv
kubectl exec -n kafka kafka-0 -- kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --group my-consumer-group \
  --reset-offsets \
  --from-file /tmp/offsets.csv \
  --execute
```

---

## <a name="producers"></a>📤 **5. Producer Monitoring**

### **Producer Metrics**

```bash
# Check producer logs in application
kubectl logs <producer-app-pod> -n <namespace> | grep -i "producer\|metadata"

# Monitor producer performance
kubectl exec <producer-app-pod> -n <namespace> -- curl http://localhost:8080/actuator/metrics/kafka.producer

# Check producer configuration
kubectl logs <producer-app-pod> -n <namespace> | grep "ProducerConfig values:"

# Test producer connectivity
kubectl exec -n kafka kafka-0 -- kafka-producer-perf-test.sh \
  --topic test-topic \
  --num-records 1000 \
  --record-size 1024 \
  --throughput 100 \
  --producer-props bootstrap.servers=localhost:9092
```

### **Console Producer (Testing)**

```bash
# Produce test messages
kubectl exec -it -n kafka kafka-0 -- kafka-console-producer.sh \
  --bootstrap-server localhost:9092 \
  --topic my-topic

# Produce with key
kubectl exec -it -n kafka kafka-0 -- kafka-console-producer.sh \
  --bootstrap-server localhost:9092 \
  --topic my-topic \
  --property "parse.key=true" \
  --property "key.separator=:"

# Produce from file
kubectl cp messages.txt kafka/kafka-0:/tmp/messages.txt
kubectl exec -n kafka kafka-0 -- kafka-console-producer.sh \
  --bootstrap-server localhost:9092 \
  --topic my-topic < /tmp/messages.txt

# Produce with specific properties
kubectl exec -it -n kafka kafka-0 -- kafka-console-producer.sh \
  --bootstrap-server localhost:9092 \
  --topic my-topic \
  --producer-property acks=all \
  --producer-property compression.type=snappy
```

### **Producer Performance Testing**

```bash
# Basic throughput test
kubectl exec -n kafka kafka-0 -- kafka-producer-perf-test.sh \
  --topic perf-test \
  --num-records 100000 \
  --record-size 1024 \
  --throughput -1 \
  --producer-props bootstrap.servers=localhost:9092

# Test with specific batch size
kubectl exec -n kafka kafka-0 -- kafka-producer-perf-test.sh \
  --topic perf-test \
  --num-records 100000 \
  --record-size 1024 \
  --throughput -1 \
  --producer-props bootstrap.servers=localhost:9092 \
  --producer-props batch.size=32768 \
  --producer-props linger.ms=10

# Test different compression types
for compression in none gzip snappy lz4 zstd; do
  echo "Testing $compression compression..."
  kubectl exec -n kafka kafka-0 -- kafka-producer-perf-test.sh \
    --topic perf-test \
    --num-records 10000 \
    --record-size 1024 \
    --throughput -1 \
    --producer-props bootstrap.servers=localhost:9092 \
    --producer-props compression.type=$compression
done
```

---

## <a name="lag-monitoring"></a>📊 **6. Lag Monitoring & Analysis**

### **Comprehensive Lag Check Script**

```bash
#!/bin/bash
# Kafka Consumer Lag Monitoring Script

NAMESPACE="kafka"
BOOTSTRAP_SERVER="localhost:9092"
THRESHOLD=10000

echo "=== Kafka Consumer Lag Report ==="
echo "Generated: $(date)"
echo "Threshold: $THRESHOLD messages"
echo ""

# Get all consumer groups
GROUPS=$(kubectl exec -n $NAMESPACE kafka-0 -- kafka-consumer-groups.sh \
  --bootstrap-server $BOOTSTRAP_SERVER \
  --list 2>/dev/null)

for GROUP in $GROUPS; do
  echo "Consumer Group: $GROUP"
  echo "----------------------------------------"
  
  # Get lag details
  LAG_OUTPUT=$(kubectl exec -n $NAMESPACE kafka-0 -- kafka-consumer-groups.sh \
    --bootstrap-server $BOOTSTRAP_SERVER \
    --group $GROUP \
    --describe 2>/dev/null)
  
  # Calculate total lag
  TOTAL_LAG=$(echo "$LAG_OUTPUT" | awk 'NR>1 {sum += $6} END {print sum+0}')
  
  if [ "$TOTAL_LAG" -gt "$THRESHOLD" ]; then
    echo "⚠️  WARNING: High lag detected ($TOTAL_LAG messages)"
  else
    echo "✓ Lag within threshold ($TOTAL_LAG messages)"
  fi
  
  # Show per-partition lag
  echo "$LAG_OUTPUT" | awk 'NR>1 && $6>0 {printf "  Topic: %s, Partition: %s, Lag: %s\n", $2, $3, $6}'
  
  # Check consumer state
  STATE=$(kubectl exec -n $NAMESPACE kafka-0 -- kafka-consumer-groups.sh \
    --bootstrap-server $BOOTSTRAP_SERVER \
    --group $GROUP \
    --state 2>/dev/null | tail -1)
  echo "  State: $STATE"
  echo ""
done

echo "=== Report Complete ==="
```

### **Lag Monitoring Commands**

```bash
# Real-time lag monitoring
watch -n 5 'kubectl exec -n kafka kafka-0 -- kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --group my-consumer-group \
  --describe'

# Export lag to CSV
kubectl exec -n kafka kafka-0 -- kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --all-groups \
  --describe | awk 'NR>1 {print $1","$2","$3","$4","$5","$6}' > lag-report.csv

# Find partitions with highest lag
kubectl exec -n kafka kafka-0 -- kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --all-groups \
  --describe | awk 'NR>1 {print $6, $1, $2, $3}' | sort -rn | head -20

# Check if consumers are active
kubectl exec -n kafka kafka-0 -- kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --group my-consumer-group \
  --describe | grep -v "PARTITION" | awk '{if($8=="-") print "⚠️  Partition "$3" has no active consumer"}'

# Calculate lag growth rate (run twice with delay)
kubectl exec -n kafka kafka-0 -- kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --group my-consumer-group \
  --describe | awk 'NR>1 {sum += $6} END {print sum}' > /tmp/lag1.txt

sleep 60

kubectl exec -n kafka kafka-0 -- kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --group my-consumer-group \
  --describe | awk 'NR>1 {sum += $6} END {print sum}' > /tmp/lag2.txt

LAG1=$(cat /tmp/lag1.txt)
LAG2=$(cat /tmp/lag2.txt)
GROWTH=$((LAG2 - LAG1))
echo "Lag growth in 60 seconds: $GROWTH messages"
```

### **Lag Alerting Logic**

```bash
#!/bin/bash
# Lag Alert Script (integrate with monitoring system)

BOOTSTRAP_SERVER="localhost:9092"
WARN_THRESHOLD=5000
CRITICAL_THRESHOLD=10000
ALERT_WEBHOOK="https://hooks.slack.com/services/YOUR/WEBHOOK/URL"

check_lag() {
  local GROUP=$1
  local LAG=$(kubectl exec -n kafka kafka-0 -- kafka-consumer-groups.sh \
    --bootstrap-server $BOOTSTRAP_SERVER \
    --group $GROUP \
    --describe 2>/dev/null | awk 'NR>1 {sum += $6} END {print sum+0}')
  
  if [ "$LAG" -gt "$CRITICAL_THRESHOLD" ]; then
    send_alert "CRITICAL" "$GROUP" "$LAG"
  elif [ "$LAG" -gt "$WARN_THRESHOLD" ]; then
    send_alert "WARNING" "$GROUP" "$LAG"
  fi
}

send_alert() {
  local SEVERITY=$1
  local GROUP=$2
  local LAG=$3
  
  MESSAGE="[$SEVERITY] Kafka Consumer Lag Alert\nGroup: $GROUP\nLag: $LAG messages"
  
  curl -X POST -H 'Content-type: application/json' \
    --data "{\"text\":\"$MESSAGE\"}" \
    $ALERT_WEBHOOK
}

# Check all consumer groups
GROUPS=$(kubectl exec -n kafka kafka-0 -- kafka-consumer-groups.sh \
  --bootstrap-server $BOOTSTRAP_SERVER \
  --list 2>/dev/null)

for GROUP in $GROUPS; do
  check_lag $GROUP
done
```

---

## <a name="performance"></a>⚡ **7. Performance Troubleshooting**

### **Identify Slow Consumers**

```bash
# Check consumer processing rate
kubectl exec -n kafka kafka-0 -- kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --all-groups \
  --describe | awk 'NR>1 {print $1, $2, ($5-$4)}' | sort -k3 -rn

# Monitor consumer rebalancing
kubectl logs <consumer-pod> -n <namespace> | grep "rebalance\|revok\|assign"

# Check consumer poll intervals
kubectl logs <consumer-pod> -n <namespace> | grep "poll"

# Identify stuck consumers
kubectl exec -n kafka kafka-0 -- kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --all-groups \
  --describe | awk '$6 > 0 && $8 == "-" {print "Stuck consumer:", $1, $2, $3}'
```

### **Broker Performance**

```bash
# Check broker CPU and memory
kubectl top pods -n kafka -l app=kafka

# Check disk I/O
kubectl exec -n kafka kafka-0 -- iostat -x 1 5

# Check network throughput
kubectl exec -n kafka kafka-0 -- iftop -t -s 5

# Monitor broker logs for performance issues
kubectl logs kafka-0 -n kafka | grep -i "slow\|timeout\|delay"

# Check request queue size
kubectl logs kafka-0 -n kafka | grep "RequestQueue"

# Monitor ISR shrink/expand events
kubectl logs kafka-0 -n kafka | grep "shrinking ISR\|expanding ISR"
```

### **Topic Performance Analysis**

```bash
# Find largest topics
kubectl exec -n kafka kafka-0 -- kafka-log-dirs.sh \
  --bootstrap-server localhost:9092 \
  --describe | jq '.brokers[].logDirs[].partitions[] | {topic: .topic, size: .size}' | \
  jq -s 'group_by(.topic) | map({topic: .[0].topic, total_size: map(.size) | add}) | sort_by(.total_size) | reverse'

# Check segment sizes
kubectl exec -n kafka kafka-0 -- du -sh /var/lib/kafka/data/*

# Find topics with most partitions
kubectl exec -n kafka kafka-0 -- kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --describe | grep "PartitionCount" | sort -t: -k2 -rn | head -10

# Check under-replicated partitions
kubectl exec -n kafka kafka-0 -- kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --describe \
  --under-replicated-partitions

# Find topics with offline partitions
kubectl exec -n kafka kafka-0 -- kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --describe \
  --unavailable-partitions
```

### **Performance Testing**

```bash
# Consumer performance test
kubectl exec -n kafka kafka-0 -- kafka-consumer-perf-test.sh \
  --bootstrap-server localhost:9092 \
  --topic perf-test \
  --messages 100000 \
  --threads 1

# End-to-end latency test
kubectl exec -n kafka kafka-0 -- kafka-run-class.sh kafka.tools.EndToEndLatency \
  localhost:9092 \
  perf-test \
  10000 \
  1 \
  1024

# Network throughput between brokers
kubectl exec -n kafka kafka-0 -- iperf3 -c kafka-1.kafka-headless.kafka.svc.cluster.local
```

---

## <a name="incidents"></a>🚨 **8. Incident Response Playbook**

### **Incident: High Consumer Lag**

```bash
# 1. Identify affected consumer groups
kubectl exec -n kafka kafka-0 -- kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --all-groups \
  --describe | awk '$6 > 10000 {print}'

# 2. Check consumer pod status
kubectl get pods -n <app-namespace> -l app=consumer-app

# 3. Check consumer logs for errors
kubectl logs <consumer-pod> -n <app-namespace> --tail=100 | grep -i "error\|exception"

# 4. Verify Kafka broker health
kubectl get pods -n kafka -l app=kafka
kubectl exec -n kafka kafka-0 -- kafka-broker-api-versions.sh --bootstrap-server localhost:9092

# 5. Check if consumers are rebalancing excessively
kubectl logs <consumer-pod> -n <app-namespace> | grep -c "rebalance"

# 6. Check partition distribution
kubectl exec -n kafka kafka-0 -- kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --group <consumer-group> \
  --members --verbose

# 7. Scale up consumers if needed
kubectl scale deployment consumer-app -n <app-namespace> --replicas=6

# 8. Monitor lag reduction
watch -n 10 'kubectl exec -n kafka kafka-0 -- kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --group <consumer-group> \
  --describe | grep LAG'
```

### **Incident: Broker Down**

```bash
# 1. Identify which broker is down
kubectl get pods -n kafka -l app=kafka

# 2. Check broker logs
kubectl logs kafka-1 -n kafka --tail=100

# 3. Check events
kubectl get events -n kafka --sort-by='.lastTimestamp' | grep kafka-1

# 4. Verify under-replicated partitions
kubectl exec -n kafka kafka-0 -- kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --describe \
  --under-replicated-partitions

# 5. Check if broker is in ISR
kubectl exec -n kafka kafka-0 -- kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --describe | grep "Isr: " | grep -v "1,"

# 6. Restart the broker pod
kubectl delete pod kafka-1 -n kafka

# 7. Wait for broker to rejoin cluster
kubectl wait --for=condition=ready pod/kafka-1 -n kafka --timeout=300s

# 8. Verify all partitions are in-sync
kubectl exec -n kafka kafka-0 -- kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --describe \
  --under-replicated-partitions

# 9. Check if leader election happened
kubectl logs kafka-0 -n kafka | grep "Elected as controller"
```

### **Incident: Disk Space Full**

```bash
# 1. Check disk usage on all brokers
for i in 0 1 2; do
  echo "=== kafka-$i ==="
  kubectl exec -n kafka kafka-$i -- df -h /var/lib/kafka/data
done

# 2. Find largest topics
kubectl exec -n kafka kafka-0 -- du -sh /var/lib/kafka/data/* | sort -rh | head -10

# 3. Check retention policies
kubectl exec -n kafka kafka-0 -- kafka-configs.sh \
  --bootstrap-server localhost:9092 \
  --entity-type topics \
  --all \
  --describe | grep retention

# 4. Trigger log cleanup manually
kubectl exec -n kafka kafka-0 -- kafka-run-class.sh kafka.admin.LogCleaner \
  /var/lib/kafka/data

# 5. Reduce retention for large topics (temporary)
kubectl exec -n kafka kafka-0 -- kafka-configs.sh \
  --bootstrap-server localhost:9092 \
  --entity-type topics \
  --entity-name <large-topic> \
  --alter \
  --add-config retention.ms=3600000  # 1 hour

# 6. Wait for cleanup, then revert retention
sleep 300
kubectl exec -n kafka kafka-0 -- kafka-configs.sh \
  --bootstrap-server localhost:9092 \
  --entity-type topics \
  --entity-name <large-topic> \
  --alter \
  --delete-config retention.ms

# 7. Long-term fix: increase PVC size
kubectl patch pvc data-kafka-0 -n kafka -p '{"spec":{"resources":{"requests":{"storage":"200Gi"}}}}'
```

### **Incident: Producer Timeout/Failures**

```bash
# 1. Check producer logs
kubectl logs <producer-pod> -n <app-namespace> | grep -i "timeout\|failed to send"

# 2. Verify broker connectivity
kubectl exec <producer-pod> -n <app-namespace> -- nc -zv kafka-service 9092

# 3. Check broker request queue
kubectl logs kafka-0 -n kafka | grep "request queue"

# 4. Check network latency
kubectl exec <producer-pod> -n <app-namespace> -- ping kafka-service

# 5. Verify topic exists and is healthy
kubectl exec -n kafka kafka-0 -- kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --describe \
  --topic <topic-name>

# 6. Check broker resource usage
kubectl top pods -n kafka

# 7. Check for network policies blocking traffic
kubectl get networkpolicies -n kafka
kubectl describe networkpolicy <policy-name> -n kafka

# 8. Test with console producer
kubectl exec -it -n kafka kafka-0 -- kafka-console-producer.sh \
  --bootstrap-server localhost:9092 \
  --topic <topic-name>
```

### **Incident: Consumer Group Stuck/Not Processing**

```bash
# 1. Check consumer group state
kubectl exec -n kafka kafka-0 -- kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --group <consumer-group> \
  --describe

# 2. Check if consumers are alive
kubectl exec -n kafka kafka-0 -- kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --group <consumer-group> \
  --members

# 3. Check for rebalancing issues
kubectl logs <consumer-pod> -n <app-namespace> | grep rebalance

# 4. Look for poison messages
kubectl logs <consumer-pod> -n <app-namespace> | grep -A 20 "exception\|error"

# 5. Check consumer application health
kubectl exec <consumer-pod> -n <app-namespace> -- curl http://localhost:8080/actuator/health

# 6. Reset offsets to skip problematic message (if identified)
kubectl exec -n kafka kafka-0 -- kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --group <consumer-group> \
  --topic <topic-name>:0 \
  --reset-offsets \
  --to-offset <offset+1> \
  --execute

# 7. Restart consumer pods
kubectl rollout restart deployment consumer-app -n <app-namespace>

# 8. Monitor recovery
watch kubectl exec -n kafka kafka-0 -- kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --group <consumer-group> \
  --describe
```

---

## <a name="data-management"></a>💾 **9. Data Management**

### **Message Inspection**

```bash
# Consume messages from beginning
kubectl exec -it -n kafka kafka-0 -- kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 \
  --topic my-topic \
  --from-beginning

# Consume with key and timestamp
kubectl exec -it -n kafka kafka-0 -- kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 \
  --topic my-topic \
  --from-beginning \
  --property print.key=true \
  --property print.timestamp=true \
  --property key.separator="|"

# Consume specific partition
kubectl exec -it -n kafka kafka-0 -- kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 \
  --topic my-topic \
  --partition 0 \
  --offset earliest

# Consume N messages and exit
kubectl exec -n kafka kafka-0 -- kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 \
  --topic my-topic \
  --from-beginning \
  --max-messages 10

# Consume messages within time range
kubectl exec -n kafka kafka-0 -- kafka-run-class.sh kafka.tools.GetOffsetShell \
  --broker-list localhost:9092 \
  --topic my-topic \
  --time $(date -d "2025-11-18 10:00:00" +%s%3N)

# Search for specific message content
kubectl exec -n kafka kafka-0 -- kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 \
  --topic my-topic \
  --from-beginning | grep "search-string"
```

### **Data Export/Import**

```bash
# Export topic data to file
kubectl exec -n kafka kafka-0 -- kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 \
  --topic my-topic \
  --from-beginning > topic-export.txt

# Copy data from pod to local
kubectl cp kafka/kafka-0:/tmp/topic-export.txt ./topic-export.txt

# Import data back to topic
kubectl cp topic-export.txt kafka/kafka-0:/tmp/import.txt
kubectl exec -n kafka kafka-0 -- kafka-console-producer.sh \
  --bootstrap-server localhost:9092 \
  --topic my-new-topic < /tmp/import.txt

# Mirror topic data
kubectl exec -n kafka kafka-0 -- kafka-mirror-maker.sh \
  --consumer.config /tmp/consumer.properties \
  --producer.config /tmp/producer.properties \
  --whitelist my-topic
```

### **Partition Management**

```bash
# Check partition distribution
kubectl exec -n kafka kafka-0 -- kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --describe \
  --topic my-topic

# Generate partition reassignment plan
cat > topics-to-move.json << EOF
{
  "topics": [{"topic": "my-topic"}],
  "version": 1
}
EOF

kubectl cp topics-to-move.json kafka/kafka-0:/tmp/topics-to-move.json

kubectl exec -n kafka kafka-0 -- kafka-reassign-partitions.sh \
  --bootstrap-server localhost:9092 \
  --topics-to-move-json-file /tmp/topics-to-move.json \
  --broker-list "0,1,2" \
  --generate

# Execute reassignment
cat > reassignment.json << EOF
{
  "version": 1,
  "partitions": [
    {"topic": "my-topic", "partition": 0, "replicas": [0,1,2], "log_dirs": ["any","any","any"]},
    {"topic": "my-topic", "partition": 1, "replicas": [1,2,0], "log_dirs": ["any","any","any"]}
  ]
}
EOF

kubectl cp reassignment.json kafka/kafka-0:/tmp/reassignment.json

kubectl exec -n kafka kafka-0 -- kafka-reassign-partitions.sh \
  --bootstrap-server localhost:9092 \
  --reassignment-json-file /tmp/reassignment.json \
  --execute

# Verify reassignment status
kubectl exec -n kafka kafka-0 -- kafka-reassign-partitions.sh \
  --bootstrap-server localhost:9092 \
  --reassignment-json-file /tmp/reassignment.json \
  --verify

# Rebalance leader distribution
kubectl exec -n kafka kafka-0 -- kafka-leader-election.sh \
  --bootstrap-server localhost:9092 \
  --election-type preferred \
  --all-topic-partitions
```

### **Log Compaction**

```bash
# Enable compaction on topic
kubectl exec -n kafka kafka-0 -- kafka-configs.sh \
  --bootstrap-server localhost:9092 \
  --entity-type topics \
  --entity-name my-compacted-topic \
  --alter \
  --add-config cleanup.policy=compact

# Configure compaction parameters
kubectl exec -n kafka kafka-0 -- kafka-configs.sh \
  --bootstrap-server localhost:9092 \
  --entity-type topics \
  --entity-name my-compacted-topic \
  --alter \
  --add-config min.compaction.lag.ms=0,delete.retention.ms=86400000,segment.ms=3600000

# Trigger compaction manually
kubectl exec -n kafka kafka-0 -- kafka-run-class.sh kafka.admin.LogCleaner

# Check compaction progress
kubectl logs kafka-0 -n kafka | grep "log-cleaner"
```

---

## <a name="security"></a>🔒 **10. Security & ACLs**

### **ACL Management**

```bash
# List all ACLs
kubectl exec -n kafka kafka-0 -- kafka-acls.sh \
  --bootstrap-server localhost:9092 \
  --list

# Add producer ACL
kubectl exec -n kafka kafka-0 -- kafka-acls.sh \
  --bootstrap-server localhost:9092 \
  --add \
  --allow-principal User:producer-user \
  --operation Write \
  --topic my-topic

# Add consumer ACL
kubectl exec -n kafka kafka-0 -- kafka-acls.sh \
  --bootstrap-server localhost:9092 \
  --add \
  --allow-principal User:consumer-user \
  --operation Read \
  --topic my-topic \
  --group my-consumer-group

# Add ACL for all topics
kubectl exec -n kafka kafka-0 -- kafka-acls.sh \
  --bootstrap-server localhost:9092 \
  --add \
  --allow-principal User:admin-user \
  --operation All \
  --topic '*' \
  --cluster

# Remove ACL
kubectl exec -n kafka kafka-0 -- kafka-acls.sh \
  --bootstrap-server localhost:9092 \
  --remove \
  --allow-principal User:producer-user \
  --operation Write \
  --topic my-topic

# List ACLs for specific user
kubectl exec -n kafka kafka-0 -- kafka-acls.sh \
  --bootstrap-server localhost:9092 \
  --list \
  --principal User:producer-user
```

### **Authentication & SSL**

```bash
# Check SSL configuration
kubectl exec -n kafka kafka-0 -- cat /opt/kafka/config/server.properties | grep ssl

# Verify certificate
kubectl exec -n kafka kafka-0 -- openssl x509 -in /etc/kafka/secrets/kafka.cert -text -noout

# Test SSL connection
kubectl exec -n kafka kafka-0 -- openssl s_client -connect kafka-0:9093 -CAfile /etc/kafka/secrets/ca.cert

# Check SASL configuration
kubectl exec -n kafka kafka-0 -- cat /opt/kafka/config/server.properties | grep sasl

# Test authenticated connection
kubectl exec -n kafka kafka-0 -- kafka-broker-api-versions.sh \
  --bootstrap-server localhost:9093 \
  --command-config /tmp/client.properties
```

---

## <a name="monitoring"></a>📈 **11. Monitoring & Alerting**

### **Key Metrics to Monitor**

```yaml
# Prometheus AlertManager Rules Example
groups:
  - name: kafka_alerts
    interval: 30s
    rules:
      # Under-replicated partitions
      - alert: KafkaUnderReplicatedPartitions
        expr: kafka_server_replicamanager_underreplicatedpartitions > 0
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Kafka has under-replicated partitions"
          
      # Offline partitions
      - alert: KafkaOfflinePartitions
        expr: kafka_controller_kafkacontroller_offlinepartitionscount > 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Kafka has offline partitions"
          
      # Consumer lag
      - alert: KafkaConsumerLag
        expr: kafka_consumergroup_lag > 10000
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Consumer group {{ $labels.consumergroup }} has high lag"
          
      # Disk usage
      - alert: KafkaDiskUsageHigh
        expr: (kafka_log_logmanager_size / kafka_log_logmanager_size_limit) > 0.8
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Kafka broker disk usage above 80%"
          
      # ISR shrink rate
      - alert: KafkaISRShrinkRate
        expr: rate(kafka_server_replicamanager_isrshrinkspers[5m]) > 0
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Kafka ISR shrink rate is high"
```

### **JMX Metrics Collection**

```bash
# Enable JMX in Kafka
kubectl exec -n kafka kafka-0 -- env | grep JMX

# Query JMX metrics using jmxterm
kubectl exec -it -n kafka kafka-0 -- java -jar jmxterm.jar -l localhost:9999

# Example JMX queries (inside jmxterm):
# bean kafka.server:type=BrokerTopicMetrics,name=MessagesInPerSec
# get Count
# get OneMinuteRate

# Export metrics to file
kubectl exec -n kafka kafka-0 -- java -jar jmxterm.jar \
  -l localhost:9999 \
  -n \
  -v silent \
  <  jmx-queries.txt > metrics.txt
```

### **Health Check Endpoints**

```bash
# Kafka broker health
curl http://kafka-0.kafka-headless:9092/health

# Check via Spring Boot Actuator (if using Spring Kafka)
kubectl exec <app-pod> -n <namespace> -- curl http://localhost:8080/actuator/health/kafka

# Custom health check script
#!/bin/bash
BOOTSTRAP_SERVER="localhost:9092"

# Check if brokers are responsive
if kubectl exec -n kafka kafka-0 -- kafka-broker-api-versions.sh \
  --bootstrap-server $BOOTSTRAP_SERVER &>/dev/null; then
  echo "✓ Kafka is healthy"
  exit 0
else
  echo "✗ Kafka is unhealthy"
  exit 1
fi
```

---

## <a name="backup"></a>💾 **12. Backup & Recovery**

### **Topic Configuration Backup**

```bash
# Export all topic configurations
kubectl exec -n kafka kafka-0 -- kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --describe > kafka-topics-backup-$(date +%Y%m%d).txt

# Backup specific topic configuration
kubectl exec -n kafka kafka-0 -- kafka-configs.sh \
  --bootstrap-server localhost:9092 \
  --entity-type topics \
  --entity-name my-topic \
  --describe > my-topic-config-$(date +%Y%m%d).txt

# Export consumer group offsets
kubectl exec -n kafka kafka-0 -- kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --group my-consumer-group \
  --describe > consumer-offsets-backup-$(date +%Y%m%d).txt
```

### **Data Backup**

```bash
# Backup topic data using MirrorMaker
cat > mirror-consumer.properties << EOF
bootstrap.servers=kafka-service:9092
group.id=mirror-maker-backup
EOF

cat > mirror-producer.properties << EOF
bootstrap.servers=backup-kafka-service:9092
EOF

kubectl exec -n kafka kafka-0 -- kafka-mirror-maker.sh \
  --consumer.config /tmp/mirror-consumer.properties \
  --producer.config /tmp/mirror-producer.properties \
  --whitelist "my-topic.*"

# Backup using kafka-console-consumer
kubectl exec -n kafka kafka-0 -- kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 \
  --topic my-topic \
  --from-beginning \
  --property print.key=true \
  --property print.timestamp=true > topic-backup-$(date +%Y%m%d).txt
```

### **Disaster Recovery**

```bash
# 1. Backup PVCs (using volume snapshots)
kubectl get pvc -n kafka -o yaml > kafka-pvcs-backup.yaml

# 2. Create volume snapshots
for i in 0 1 2; do
  kubectl create -f - << EOF
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: kafka-$i-snapshot-$(date +%Y%m%d)
  namespace: kafka
spec:
  source:
    persistentVolumeClaimName: data-kafka-$i
EOF
done

# 3. Restore from snapshot (in case of disaster)
kubectl create -f - << EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-kafka-0-restored
  namespace: kafka
spec:
  dataSource:
    name: kafka-0-snapshot-20251118
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Gi
EOF
```

---

## <a name="kubernetes"></a>☸️ **13. Kubernetes-Specific Operations**

### **Kafka on Kubernetes Best Practices**

```bash
# Check Kafka StatefulSet
kubectl get statefulset kafka -n kafka -o yaml

# Verify headless service
kubectl get service kafka-headless -n kafka

# Check pod disruption budget
kubectl get pdb -n kafka

# Verify persistent volume claims
kubectl get pvc -n kafka

# Check storage class
kubectl get storageclass

# Verify pod anti-affinity rules
kubectl get statefulset kafka -n kafka -o jsonpath='{.spec.template.spec.affinity}'
```

### **Kafka Resource Management**

```bash
# Check resource requests and limits
kubectl get pods -n kafka -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[0].resources}{"\n"}{end}'

# Update resource limits
kubectl patch statefulset kafka -n kafka -p '{"spec":{"template":{"spec":{"containers":[{"name":"kafka","resources":{"limits":{"memory":"4Gi","cpu":"2000m"}}}]}}}}'

# Check actual resource usage
kubectl top pods -n kafka

# Set pod priority
kubectl patch statefulset kafka -n kafka -p '{"spec":{"template":{"spec":{"priorityClassName":"high-priority"}}}}'
```

### **Network Policies**

```bash
# Check existing network policies
kubectl get networkpolicy -n kafka

# Create network policy for Kafka
kubectl apply -f - << EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: kafka-network-policy
  namespace: kafka
spec:
  podSelector:
    matchLabels:
      app: kafka
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: production
    ports:
    - protocol: TCP
      port: 9092
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: kafka
    ports:
    - protocol: TCP
      port: 9092
EOF
```

### **Kafka Operator Management**

```bash
# If using Strimzi Kafka Operator
kubectl get kafka -n kafka
kubectl describe kafka my-cluster -n kafka

# Check Kafka cluster status
kubectl get kafka my-cluster -n kafka -o jsonpath='{.status.conditions}'

# Update Kafka cluster
kubectl edit kafka my-cluster -n kafka

# Check operator logs
kubectl logs -n kafka -l name=strimzi-cluster-operator -f
```

---

## <a name="common-issues"></a>🔧 **14. Common Issues & Solutions**

### **Issue: Consumer Rebalancing Too Frequently**

```bash
# Problem: Consumers keep rebalancing
# Symptoms: High lag, slow processing, frequent "Revoking" messages

# Diagnosis:
kubectl logs <consumer-pod> -n <namespace> | grep -c "Revoking\|partitions assigned"

# Solutions:
# 1. Increase session timeout
#    spring.kafka.consumer.properties.session.timeout.ms=30000
#    spring.kafka.consumer.properties.max.poll.interval.ms=300000

# 2. Check for slow message processing
kubectl logs <consumer-pod> -n <namespace> | grep "processing time"

# 3. Verify consumer group is not too large
kubectl exec -n kafka kafka-0 -- kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --group my-consumer-group \
  --members
```

### **Issue: Out of Memory Errors**

```bash
# Problem: Kafka brokers running out of memory
# Symptoms: OOMKilled, slow performance, unresponsive brokers

# Diagnosis:
kubectl describe pod kafka-0 -n kafka | grep -i "oom"
kubectl logs kafka-0 -n kafka --previous | grep "OutOfMemoryError"

# Solutions:
# 1. Increase memory limits
kubectl patch statefulset kafka -n kafka -p '{"spec":{"template":{"spec":{"containers":[{"name":"kafka","resources":{"limits":{"memory":"8Gi"}}}]}}}}'

# 2. Tune JVM heap size (set to 50-75% of container memory)
#    KAFKA_HEAP_OPTS: "-Xms4g -Xmx4g"

# 3. Check for memory leaks
kubectl exec -n kafka kafka-0 -- jmap -heap 1
```

### **Issue: Message Loss**

```bash
# Problem: Messages are being lost
# Symptoms: Message count doesn't match, data inconsistency

# Diagnosis:
# 1. Check producer acknowledgment settings
kubectl logs <producer-pod> -n <namespace> | grep "acks="

# 2. Verify min.insync.replicas
kubectl exec -n kafka kafka-0 -- kafka-configs.sh \
  --bootstrap-server localhost:9092 \
  --entity-type topics \
  --entity-name my-topic \
  --describe | grep min.insync.replicas

# 3. Check for broker failures during write
kubectl get events -n kafka --sort-by='.lastTimestamp' | grep kafka

# Solutions:
# 1. Set producer acks=all
#    spring.kafka.producer.acks=all

# 2. Set min.insync.replicas
kubectl exec -n kafka kafka-0 -- kafka-configs.sh \
  --bootstrap-server localhost:9092 \
  --entity-type topics \
  --entity-name my-topic \
  --alter \
  --add-config min.insync.replicas=2

# 3. Enable idempotence
#    spring.kafka.producer.properties.enable.idempotence=true
```

### **Issue: Slow Consumer Processing**

```bash
# Problem: Consumers are slow and lag is growing
# Symptoms: High lag, slow throughput

# Diagnosis:
# 1. Check consumer poll time
kubectl logs <consumer-pod> -n <namespace> | grep "poll"

# 2. Profile message processing time
kubectl logs <consumer-pod> -n <namespace> | grep "processing took"

# 3. Check for database bottlenecks
kubectl logs <consumer-pod> -n <namespace> | grep "SQLException\|timeout"

# Solutions:
# 1. Increase consumer instances
kubectl scale deployment consumer-app -n <namespace> --replicas=6

# 2. Increase partition count
kubectl exec -n kafka kafka-0 -- kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --alter \
  --topic my-topic \
  --partitions 12

# 3. Optimize message processing code
# 4. Batch processing if possible
#    spring.kafka.listener.concurrency=3
```

---

## <a name="scripts"></a>📜 **15. Useful Scripts**

### **Complete Health Check Script**

```bash
#!/bin/bash
# Comprehensive Kafka Health Check

NAMESPACE="kafka"
BOOTSTRAP_SERVER="localhost:9092"
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m'

echo "╔════════════════════════════════════════════╗"
echo "║   Kafka Cluster Health Check              ║"
echo "║   Date: $(date '+%Y-%m-%d %H:%M:%S')      ║"
echo "╚════════════════════════════════════════════╝"
echo ""

# 1. Pod Status
echo "━━━ Pod Status ━━━"
POD_STATUS=$(kubectl get pods -n $NAMESPACE -l app=kafka -o jsonpath='{.items[*].status.phase}')
if [[ "$POD_STATUS" == *"Running"* ]]; then
  echo -e "${GREEN}✓${NC} All Kafka pods are running"
else
  echo -e "${RED}✗${NC} Some Kafka pods are not running"
  kubectl get pods -n $NAMESPACE -l app=kafka
fi
echo ""

# 2. Broker Connectivity
echo "━━━ Broker Connectivity ━━━"
if kubectl exec -n $NAMESPACE kafka-0 -- kafka-broker-api-versions.sh \
  --bootstrap-server $BOOTSTRAP_SERVER &>/dev/null; then
  echo -e "${GREEN}✓${NC} Brokers are responsive"
else
  echo -e "${RED}✗${NC} Brokers are not responsive"
fi
echo ""

# 3. Under-Replicated Partitions
echo "━━━ Under-Replicated Partitions ━━━"
URP=$(kubectl exec -n $NAMESPACE kafka-0 -- kafka-topics.sh \
  --bootstrap-server $BOOTSTRAP_SERVER \
  --describe \
  --under-replicated-partitions 2>/dev/null | grep -v "^$" | tail -n +2 | wc -l)

if [ "$URP" -eq 0 ]; then
  echo -e "${GREEN}✓${NC} No under-replicated partitions"
else
  echo -e "${YELLOW}⚠${NC} Found $URP under-replicated partition(s)"
fi
echo ""

# 4. Offline Partitions
echo "━━━ Offline Partitions ━━━"
OFFLINE=$(kubectl exec -n $NAMESPACE kafka-0 -- kafka-topics.sh \
  --bootstrap-server $BOOTSTRAP_SERVER \
  --describe \
  --unavailable-partitions 2>/dev/null | grep -v "^$" | tail -n +2 | wc -l)

if [ "$OFFLINE" -eq 0 ]; then
  echo -e "${GREEN}✓${NC} No offline partitions"
else
  echo -e "${RED}✗${NC} Found $OFFLINE offline partition(s)"
fi
echo ""

# 5. Consumer Lag
echo "━━━ Consumer Lag Summary ━━━"
GROUPS=$(kubectl exec -n $NAMESPACE kafka-0 -- kafka-consumer-groups.sh \
  --bootstrap-server $BOOTSTRAP_SERVER \
  --list 2>/dev/null)

HIGH_LAG_COUNT=0
for GROUP in $GROUPS; do
  TOTAL_LAG=$(kubectl exec -n $NAMESPACE kafka-0 -- kafka-consumer-groups.sh \
    --bootstrap-server $BOOTSTRAP_SERVER \
    --group $GROUP \
    --describe 2>/dev/null | awk 'NR>1 {sum += $6} END {print sum+0}')
  
  if [ "$TOTAL_LAG" -gt 10000 ]; then
    echo -e "${YELLOW}⚠${NC} Group: $GROUP - Lag: $TOTAL_LAG"
    HIGH_LAG_COUNT=$((HIGH_LAG_COUNT + 1))
  fi
done

if [ "$HIGH_LAG_COUNT" -eq 0 ]; then
  echo -e "${GREEN}✓${NC} All consumer groups have acceptable lag"
fi
echo ""

# 6. Disk Usage
echo "━━━ Disk Usage ━━━"
for i in 0 1 2; do
  USAGE=$(kubectl exec -n $NAMESPACE kafka-$i -- df -h /var/lib/kafka/data 2>/dev/null | tail -1 | awk '{print $5}' | tr -d '%')
  if [ "$USAGE" -gt 80 ]; then
    echo -e "${RED}✗${NC} kafka-$i: ${USAGE}% (Critical)"
  elif [ "$USAGE" -gt 70 ]; then
    echo -e "${YELLOW}⚠${NC} kafka-$i: ${USAGE}% (Warning)"
  else
    echo -e "${GREEN}✓${NC} kafka-$i: ${USAGE}%"
  fi
done
echo ""

# 7. Resource Usage
echo "━━━ Resource Usage ━━━"
kubectl top pods -n $NAMESPACE -l app=kafka 2>/dev/null || echo "Metrics server not available"
echo ""

echo "╔════════════════════════════════════════════╗"
echo "║   Health Check Complete                    ║"
echo "╚════════════════════════════════════════════╝"
```

### **Lag Monitoring Script with Alerts**

```bash
#!/bin/bash
# Kafka Lag Monitor with Slack Alerts

NAMESPACE="kafka"
BOOTSTRAP_SERVER="localhost:9092"
LAG_THRESHOLD=5000
SLACK_WEBHOOK="https://hooks.slack.com/services/YOUR/WEBHOOK/URL"

send_slack_alert() {
  local MESSAGE=$1
  curl -X POST -H 'Content-type: application/json' \
    --data "{\"text\":\"$MESSAGE\"}" \
    "$SLACK_WEBHOOK" 2>/dev/null
}

GROUPS=$(kubectl exec -n $NAMESPACE kafka-0 -- kafka-consumer-groups.sh \
  --bootstrap-server $BOOTSTRAP_SERVER \
  --list 2>/dev/null)

for GROUP in $GROUPS; do
  LAG_DATA=$(kubectl exec -n $NAMESPACE kafka-0 -- kafka-consumer-groups.sh \
    --bootstrap-server $BOOTSTRAP_SERVER \
    --group $GROUP \
    --describe 2>/dev/null)
  
  TOTAL_LAG=$(echo "$LAG_DATA" | awk 'NR>1 {sum += $6} END {print sum+0}')
  
  if [ "$TOTAL_LAG" -gt "$LAG_THRESHOLD" ]; then
    ALERT_MSG="🚨 Kafka Lag Alert\nGroup: $GROUP\nTotal Lag: $TOTAL_LAG messages\nThreshold: $LAG_THRESHOLD"
    echo -e "$ALERT_MSG"
    send_slack_alert "$ALERT_MSG"
    
    # Log top lagging partitions
    echo "$LAG_DATA" | awk -v threshold=$LAG_THRESHOLD 'NR>1 && $6>threshold {print "  Topic:", $2, "Partition:", $3, "Lag:", $6}'
  fi
done
```

### **Topic Cleanup Script**

```bash
#!/bin/bash
# Clean up old/unused topics

NAMESPACE="kafka"
BOOTSTRAP_SERVER="localhost:9092"
DAYS_THRESHOLD=30

echo "Finding topics with no activity for $DAYS_THRESHOLD days..."

TOPICS=$(kubectl exec -n $NAMESPACE kafka-0 -- kafka-topics.sh \
  --bootstrap-server $BOOTSTRAP_SERVER \
  --list 2>/dev/null | grep -v "^__")

for TOPIC in $TOPICS; do
  # Get last modified time of topic data
  LAST_MODIFIED=$(kubectl exec -n $NAMESPACE kafka-0 -- \
    find /var/lib/kafka/data/$TOPIC-* -type f -printf '%T@ %p\n' 2>/dev/null | \
    sort -n | tail -1 | awk '{print $1}')
  
  if [ -n "$LAST_MODIFIED" ]; then
    CURRENT_TIME=$(date +%s)
    AGE_DAYS=$(( ($CURRENT_TIME - ${LAST_MODIFIED%.*}) / 86400 ))
    
    if [ "$AGE_DAYS" -gt "$DAYS_THRESHOLD" ]; then
      echo "Topic: $TOPIC - Last activity: $AGE_DAYS days ago"
      read -p "Delete this topic? (y/N): " confirm
      if [[ $confirm == [yY] ]]; then
        kubectl exec -n $NAMESPACE kafka-0 -- kafka-topics.sh \
          --bootstrap-server $BOOTSTRAP_SERVER \
          --delete \
          --topic $TOPIC
        echo "Deleted: $TOPIC"
      fi
    fi
  fi
done
```

---

## 🎯 **Quick Reference Commands**

```bash
# Daily Health Check
kubectl get pods -n kafka && kubectl exec -n kafka kafka-0 -- kafka-topics.sh --bootstrap-server localhost:9092 --describe --under-replicated-partitions

# Check All Consumer Lag
kubectl exec -n kafka kafka-0 -- kafka-consumer-groups.sh --bootstrap-server localhost:9092 --all-groups --describe | grep -v "PARTITION" | awk '$6>0{print}'

# Find Largest Topics
kubectl exec -n kafka kafka-0 -- du -sh /var/lib/kafka/data/* | sort -rh | head -10

# Quick Broker Test
kubectl exec -n kafka kafka-0 -- kafka-broker-api-versions.sh --bootstrap-server localhost:9092

# Emergency: Restart All Kafka Pods
for i in 0 1 2; do kubectl delete pod kafka-$i -n kafka; sleep 60; done

# Check Disk Usage
for i in 0 1 2; do echo "kafka-$i:"; kubectl exec -n kafka kafka-$i -- df -h /var/lib/kafka/data; done
```

---

**Created for Site Reliability Engineers managing Kafka clusters**  
*Last Updated: November 18, 2025*

**Remember:** Always test commands in a non-production environment first!

