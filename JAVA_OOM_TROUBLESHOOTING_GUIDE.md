# Java Out Of Memory (OOM) Troubleshooting Guide

A comprehensive guide for Senior Java Developers and SREs to diagnose, analyze, and resolve Out Of Memory issues in production systems.

## 📋 **Table of Contents**

### **Part 1: Understanding OOM**
1. [Types of OOM Errors](#types-of-oom)
2. [Memory Areas in JVM](#memory-areas)
3. [Diagnostic Methodology](#methodology)
4. [Essential Tools](#tools)

### **Part 2: Diagnostic Scenarios**
5. [Scenario 1: Heap Space OOM](#scenario-heap)
6. [Scenario 2: Metaspace/PermGen OOM](#scenario-metaspace)
7. [Scenario 3: Direct Memory OOM](#scenario-direct)
8. [Scenario 4: Unable to Create Native Thread](#scenario-thread)
9. [Scenario 5: GC Overhead Limit Exceeded](#scenario-gc-overhead)
10. [Scenario 6: Array Size Too Large](#scenario-array)

### **Part 3: Common Patterns**
11. [Memory Leak Patterns](#leak-patterns)
12. [Anti-Patterns That Cause OOM](#anti-patterns)
13. [Framework-Specific Issues](#framework-issues)

### **Part 4: Solutions & Prevention**
14. [Immediate Response Playbook](#immediate-response)
15. [Long-term Solutions](#long-term-solutions)
16. [Best Practices](#best-practices)
17. [Monitoring & Alerting](#monitoring)

---

## <a name="types-of-oom"></a>**1. Types of OOM Errors**

### **Complete Error Types**

```java
// 1. Java heap space
java.lang.OutOfMemoryError: Java heap space

// 2. GC Overhead limit exceeded
java.lang.OutOfMemoryError: GC overhead limit exceeded

// 3. Metaspace (Java 8+)
java.lang.OutOfMemoryError: Metaspace

// 4. PermGen space (Java 7 and below)
java.lang.OutOfMemoryError: PermGen space

// 5. Unable to create new native thread
java.lang.OutOfMemoryError: unable to create new native thread

// 6. Direct buffer memory
java.lang.OutOfMemoryError: Direct buffer memory

// 7. request size bytes for reason. Out of swap space?
java.lang.OutOfMemoryError: request 32756 bytes for ChunkPool::allocate. Out of swap space?

// 8. Requested array size exceeds VM limit
java.lang.OutOfMemoryError: Requested array size exceeds VM limit

// 9. Kill process or sacrifice child (Linux OOM Killer)
// Check: dmesg | grep -i "out of memory"
```

### **Understanding Each Type**

| Error Type | Memory Area | Common Causes | Severity |
|------------|-------------|---------------|----------|
| Java heap space | Heap | Objects not GC'd, memory leaks, insufficient heap | Critical |
| GC overhead limit | Heap | Too much time in GC, little memory freed | Critical |
| Metaspace | Non-heap | Too many classes loaded, class loader leak | High |
| Unable to create thread | OS | Thread leak, stack size too large | Critical |
| Direct buffer memory | Off-heap | NIO buffers not released | High |
| Out of swap | OS | System-wide memory exhaustion | Critical |

---

## <a name="memory-areas"></a>**2. Memory Areas in JVM**

### **JVM Memory Structure**

```
┌─────────────────────────────────────────────────────────┐
│                    JVM Process Memory                    │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  ┌────────────────────────────────────────────────┐    │
│  │              Heap Memory (-Xmx)                │    │
│  │  ┌──────────────┐  ┌──────────────────────┐   │    │
│  │  │  Young Gen   │  │     Old Gen          │   │    │
│  │  │ ┌──────────┐ │  │                      │   │    │
│  │  │ │  Eden    │ │  │                      │   │    │
│  │  │ ├──────────┤ │  │                      │   │    │
│  │  │ │Survivor 0│ │  │                      │   │    │
│  │  │ │Survivor 1│ │  │                      │   │    │
│  │  │ └──────────┘ │  └──────────────────────┘   │    │
│  │  └──────────────┘                              │    │
│  └────────────────────────────────────────────────┘    │
│                                                          │
│  ┌────────────────────────────────────────────────┐    │
│  │         Metaspace / PermGen (-XX:MaxMetaspaceSize)  │
│  │  • Class metadata                              │    │
│  │  • Method data                                 │    │
│  │  • Constant pool                               │    │
│  └────────────────────────────────────────────────┘    │
│                                                          │
│  ┌────────────────────────────────────────────────┐    │
│  │         Code Cache                              │    │
│  │  • JIT compiled code                           │    │
│  └────────────────────────────────────────────────┘    │
│                                                          │
│  ┌────────────────────────────────────────────────┐    │
│  │         Direct Memory                           │    │
│  │  • NIO direct buffers                          │    │
│  └────────────────────────────────────────────────┘    │
│                                                          │
│  ┌────────────────────────────────────────────────┐    │
│  │         Thread Stacks (-Xss per thread)        │    │
│  │  • Local variables                             │    │
│  │  • Method parameters                           │    │
│  │  • Return addresses                            │    │
│  └────────────────────────────────────────────────┘    │
│                                                          │
│  ┌────────────────────────────────────────────────┐    │
│  │         Native Memory                           │    │
│  │  • JNI code                                    │    │
│  │  • Native libraries                            │    │
│  └────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────┘
```

### **Memory Size Relationships**

```bash
# Total Process Memory ≈
Heap (-Xmx) 
+ Metaspace (-XX:MaxMetaspaceSize)
+ Code Cache (-XX:ReservedCodeCacheSize)
+ Thread Stacks (ThreadCount × -Xss)
+ Direct Memory (-XX:MaxDirectMemorySize)
+ Native Memory
+ JVM Internal Structures
```

---

## <a name="methodology"></a>**3. Diagnostic Methodology**

### **Root Cause Analysis Framework**

```
┌─────────────────────────────────────────────────────────┐
│            OOM Error Occurred                           │
└────────────────┬────────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────────┐
│  STEP 1: Identify OOM Type                             │
│  • Read error message                                   │
│  • Check logs: grep "OutOfMemoryError"                 │
│  • Review monitoring dashboards                         │
└────────────────┬────────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────────┐
│  STEP 2: Collect Diagnostic Data                       │
│  • Heap dump (if available)                            │
│  • Thread dump                                          │
│  • GC logs                                              │
│  • System metrics                                       │
│  • Application logs                                     │
└────────────────┬────────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────────┐
│  STEP 3: Analyze Memory Usage                          │
│  • What objects consume most memory?                   │
│  • Where are they allocated?                           │
│  • Why aren't they being GC'd?                        │
└────────────────┬────────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────────┐
│  STEP 4: Identify Root Cause                           │
│  • Memory leak?                                         │
│  • Insufficient memory?                                 │
│  • Memory spike?                                        │
│  • Configuration issue?                                 │
└────────────────┬────────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────────┐
│  STEP 5: Implement Solution                            │
│  • Immediate: Restart with more memory                 │
│  • Short-term: Fix leak or optimize code               │
│  • Long-term: Architecture changes                     │
└─────────────────────────────────────────────────────────┘
```

### **Decision Tree for OOM Diagnosis**

```
OOM Error?
    │
    ├─ "Java heap space"
    │   │
    │   ├─ Heap dump shows: 
    │   │   ├─ Large collections → Memory leak or insufficient sizing
    │   │   ├─ Many small objects → Check object creation rate
    │   │   └─ Few large objects → Check caching strategy
    │   │
    │   └─ GC logs show:
    │       ├─ Memory steadily increasing → Memory leak
    │       ├─ Memory spikes then drops → Traffic spike (need more memory)
    │       └─ Memory flat near max → Insufficient heap size
    │
    ├─ "Metaspace" / "PermGen"
    │   │
    │   └─ Check:
    │       ├─ Number of classes loaded → Use class loading monitoring
    │       ├─ Classloader leaks → Check for redeploys without cleanup
    │       └─ Too many generated classes → Review dynamic proxies/reflection
    │
    ├─ "unable to create new native thread"
    │   │
    │   └─ Check:
    │       ├─ Thread count → jstack, visualvm
    │       ├─ OS limits → ulimit -u
    │       └─ Thread leaks → Review ExecutorService usage
    │
    ├─ "Direct buffer memory"
    │   │
    │   └─ Check:
    │       ├─ NIO usage → Look for ByteBuffer.allocateDirect()
    │       └─ Netty/network libraries → Review buffer pools
    │
    └─ "GC overhead limit exceeded"
        │
        └─ Check:
            ├─ GC time > 98% → Heap exhausted
            └─ Memory freed < 2% → Severe memory pressure
```

---

## <a name="tools"></a>**4. Essential Tools**

### **JVM Built-in Tools**

```bash
# ============================================
# 1. jps - List Java Processes
# ============================================
jps -lv
# Output: PID MainClass JVM_Args

# ============================================
# 2. jstat - JVM Statistics
# ============================================

# Monitor GC activity
jstat -gc <pid> 1000 10
# Every 1000ms, 10 times

# Monitor GC causes
jstat -gccause <pid> 1000

# Monitor heap usage
jstat -gcutil <pid> 1000

# Output columns:
# S0     - Survivor space 0 utilization (%)
# S1     - Survivor space 1 utilization (%)
# E      - Eden space utilization (%)
# O      - Old space utilization (%)
# M      - Metaspace utilization (%)
# CCS    - Compressed class space utilization (%)
# YGC    - Young generation GC count
# YGCT   - Young generation GC time
# FGC    - Full GC count
# FGCT   - Full GC time
# GCT    - Total GC time

# ============================================
# 3. jmap - Memory Map
# ============================================

# Generate heap dump
jmap -dump:format=b,file=heap.hprof <pid>

# With live objects only (forces Full GC first)
jmap -dump:live,format=b,file=heap-live.hprof <pid>

# Show heap configuration
jmap -heap <pid>

# Show histogram of object counts
jmap -histo <pid> | head -20

# Show histogram of live objects
jmap -histo:live <pid> | head -20

# ============================================
# 4. jstack - Thread Dump
# ============================================

# Generate thread dump
jstack <pid> > thread-dump.txt

# Multiple thread dumps (for deadlock detection)
for i in {1..5}; do
  echo "=== Thread Dump $i at $(date) ===" >> thread-dumps.txt
  jstack <pid> >> thread-dumps.txt
  sleep 5
done

# ============================================
# 5. jcmd - JVM Diagnostic Command
# ============================================

# List available commands
jcmd <pid> help

# Generate heap dump
jcmd <pid> GC.heap_dump heap.hprof

# Show VM flags
jcmd <pid> VM.flags

# Show system properties
jcmd <pid> VM.system_properties

# Show JVM uptime
jcmd <pid> VM.uptime

# Trigger GC
jcmd <pid> GC.run

# Show class statistics
jcmd <pid> GC.class_histogram

# ============================================
# 6. jinfo - Configuration Info
# ============================================

# Show all VM flags
jinfo -flags <pid>

# Show specific flag
jinfo -flag MaxHeapSize <pid>

# Set flag (if modifiable)
jinfo -flag +PrintGCDetails <pid>

# ============================================
# 7. Native Memory Tracking (NMT)
# ============================================

# Enable NMT (requires JVM restart)
java -XX:NativeMemoryTracking=summary -jar app.jar
# or
java -XX:NativeMemoryTracking=detail -jar app.jar

# Get NMT summary
jcmd <pid> VM.native_memory summary

# Get NMT detail
jcmd <pid> VM.native_memory detail

# Baseline for comparison
jcmd <pid> VM.native_memory baseline

# Compare with baseline
jcmd <pid> VM.native_memory summary.diff
```

### **Kubernetes/Docker Commands**

```bash
# ============================================
# Container Memory Analysis
# ============================================

# Check container memory limits
kubectl describe pod <pod-name> -n <namespace> | grep -A 5 "Limits:"

# Check actual memory usage
kubectl top pod <pod-name> -n <namespace>

# Check OOMKilled status
kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.status.containerStatuses[0].lastState.terminated.reason}'

# View container logs before crash
kubectl logs <pod-name> -n <namespace> --previous

# Get heap dump from container
kubectl exec <pod-name> -n <namespace> -- jmap -dump:format=b,file=/tmp/heap.hprof 1
kubectl cp <namespace>/<pod-name>:/tmp/heap.hprof ./heap.hprof

# Get thread dump from container
kubectl exec <pod-name> -n <namespace> -- jstack 1 > thread-dump.txt

# Check JVM flags in container
kubectl exec <pod-name> -n <namespace> -- jinfo -flags 1

# Monitor memory in real-time
kubectl exec <pod-name> -n <namespace> -- sh -c 'while true; do jstat -gcutil 1 1000 1; sleep 2; done'

# Docker equivalent
docker stats <container-id>
docker exec <container-id> jmap -dump:format=b,file=/tmp/heap.hprof 1
```

### **Heap Dump Analysis Tools**

```bash
# ============================================
# 1. Eclipse Memory Analyzer (MAT)
# ============================================

# Download: https://eclipse.dev/mat/downloads.php

# Command line analysis
./mat/ParseHeapDump.sh heap.hprof org.eclipse.mat.api:suspects
./mat/ParseHeapDump.sh heap.hprof org.eclipse.mat.api:overview
./mat/ParseHeapDump.sh heap.hprof org.eclipse.mat.api:top_components

# Key reports in MAT:
# - Leak Suspects: Automatically identifies potential memory leaks
# - Dominator Tree: Shows retained heap by object
# - Top Consumers: Largest objects
# - Duplicate Classes: Classloader issues
# - Thread Overview: Memory per thread

# ============================================
# 2. VisualVM
# ============================================

# Download: https://visualvm.github.io/download.html

# Connect to running JVM
# File → Load → heap.hprof

# Features:
# - OQL (Object Query Language) console
# - Class histogram
# - Instance browser
# - GC roots

# Sample OQL queries:
# Find all strings containing "password"
select s from java.lang.String s where s.toString().contains("password")

# Find large byte arrays
select s from byte[] s where s.@length > 1000000

# ============================================
# 3. jhat (deprecated but still useful)
# ============================================

# Start web server with heap dump
jhat -J-Xmx4g heap.hprof

# Access at http://localhost:7000

# ============================================
# 4. YourKit
# ============================================

# Commercial tool with advanced features
# https://www.yourkit.com/

# ============================================
# 5. JProfiler
# ============================================

# Commercial tool
# https://www.ej-technologies.com/products/jprofiler/overview.html
```

### **GC Log Analysis**

```bash
# ============================================
# Enable GC Logging (Java 8)
# ============================================

java -XX:+PrintGCDetails \
     -XX:+PrintGCDateStamps \
     -XX:+PrintGCTimeStamps \
     -Xloggc:gc.log \
     -XX:+UseGCLogFileRotation \
     -XX:NumberOfGCLogFiles=5 \
     -XX:GCLogFileSize=10M \
     -jar app.jar

# ============================================
# Enable GC Logging (Java 9+)
# ============================================

java -Xlog:gc*:file=gc.log:time,uptime,level,tags \
     -Xlog:gc=debug:file=gc-debug.log \
     -jar app.jar

# ============================================
# GC Log Analysis Tools
# ============================================

# 1. GCViewer
# https://github.com/chewiebug/GCViewer
java -jar gcviewer.jar gc.log

# 2. GCeasy (Online)
# https://gceasy.io/
# Upload gc.log file

# 3. GC Log Analyzer (Command line)
# Parse GC logs
grep "Full GC" gc.log | wc -l  # Count Full GCs
grep "Full GC" gc.log | tail -10  # Recent Full GCs

# Extract pause times
grep -oP "\[Times.*real=\K[0-9.]+" gc.log | sort -n | tail -20

# ============================================
# Analyzing GC Patterns
# ============================================

# Good pattern:
# - Young GC: frequent, fast (<100ms)
# - Full GC: rare (hours/days apart)
# - Memory drops significantly after GC

# Bad pattern (Memory Leak):
# [Young GC] Heap: 500M → 450M
# [Young GC] Heap: 550M → 500M
# [Young GC] Heap: 600M → 550M
# [Full GC]  Heap: 600M → 580M  ← Old gen not cleared
# [Young GC] Heap: 650M → 600M
# [Full GC]  Heap: 700M → 680M  ← Getting worse
# ... eventually OOM

# Good pattern (Normal operation):
# [Young GC] Heap: 500M → 200M
# [Young GC] Heap: 500M → 200M
# [Young GC] Heap: 500M → 200M
# [Full GC]  Heap: 800M → 150M  ← Old gen cleaned
```

---

## <a name="scenario-heap"></a>**5. Scenario 1: Java Heap Space OOM**

### **Error Message**

```
java.lang.OutOfMemoryError: Java heap space
	at java.util.Arrays.copyOf(Arrays.java:3332)
	at java.lang.AbstractStringBuilder.ensureCapacityInternal(AbstractStringBuilder.java:124)
	at java.lang.AbstractStringBuilder.append(AbstractStringBuilder.java:448)
	at java.lang.StringBuilder.append(StringBuilder.java:136)
	at com.example.service.ReportGenerator.generateReport(ReportGenerator.java:45)
```

### **Symptoms**

- Application crashes with OOM
- Before crash: slow performance, high GC activity
- Pod shows "OOMKilled" status in Kubernetes
- Memory usage steadily increasing over time

### **Investigation Steps**

```bash
# 1. Check if heap dump was auto-generated
# (if -XX:+HeapDumpOnOutOfMemoryError was set)
ls -lh /path/to/heapdump/

# 2. If running, generate heap dump immediately
kubectl exec <pod-name> -n <namespace> -- jmap -dump:live,format=b,file=/tmp/heap.hprof 1

# 3. Copy heap dump locally
kubectl cp <namespace>/<pod-name>:/tmp/heap.hprof ./heap-$(date +%Y%m%d-%H%M%S).hprof

# 4. Check heap configuration
kubectl exec <pod-name> -n <namespace> -- jinfo -flag MaxHeapSize 1
kubectl exec <pod-name> -n <namespace> -- jinfo -flag InitialHeapSize 1

# 5. Check current memory usage
kubectl exec <pod-name> -n <namespace> -- jstat -gcutil 1 1000 5

# 6. Review recent GC activity
kubectl logs <pod-name> -n <namespace> | grep "Full GC"
```

### **Heap Dump Analysis with MAT**

```bash
# 1. Open heap dump in Eclipse MAT

# 2. Check "Leak Suspects" report (automatic)
# MAT will show top suspects like:
# "One instance of java.util.HashMap loaded by <classloader> 
#  occupies 1.2 GB (85%) of total heap"

# 3. View Dominator Tree
# Shows objects and their retained heap size
# Look for:
# - Unexpectedly large objects
# - Collections with many elements
# - Cached data that's too large

# 4. Check for duplicate classes
# Histogram → Right-click → "Group by Class Loader"
# If you see same class multiple times = classloader leak

# 5. Find objects preventing GC
# Right-click object → "Path to GC Roots" → "exclude weak references"
# This shows what's keeping the object alive
```

### **Common Root Causes**

**1. Memory Leak - Static Collections Growing**

```java
// ❌ BAD - Memory Leak
public class UserCache {
    // This will grow forever!
    private static final Map<String, User> cache = new HashMap<>();
    
    public void cacheUser(String id, User user) {
        cache.put(id, user);  // Never removed!
    }
}

// Heap dump shows:
// HashMap @ 0x123456 - Size: 2.5 GB (85% of heap)
//   └─ 5,000,000 User objects

// ✅ GOOD - Solution 1: Use bounded cache
public class UserCache {
    private static final Cache<String, User> cache = Caffeine.newBuilder()
        .maximumSize(10_000)
        .expireAfterWrite(Duration.ofHours(1))
        .build();
    
    public void cacheUser(String id, User user) {
        cache.put(id, user);
    }
}

// ✅ GOOD - Solution 2: Use WeakHashMap
public class UserCache {
    private static final Map<String, User> cache = 
        Collections.synchronizedMap(new WeakHashMap<>());
    
    public void cacheUser(String id, User user) {
        cache.put(id, user);  // Will be GC'd when user is no longer referenced
    }
}

// ✅ GOOD - Solution 3: Manual eviction
public class UserCache {
    private static final Map<String, CachedUser> cache = new ConcurrentHashMap<>();
    
    static class CachedUser {
        User user;
        long timestamp;
    }
    
    @Scheduled(fixedDelay = 60000)
    public void evictExpired() {
        long cutoff = System.currentTimeMillis() - TimeUnit.HOURS.toMillis(1);
        cache.entrySet().removeIf(entry -> entry.getValue().timestamp < cutoff);
    }
}
```

**2. Memory Leak - Event Listeners Not Removed**

```java
// ❌ BAD - Listener Leak
public class OrderService {
    @Autowired
    private ApplicationEventPublisher eventPublisher;
    
    public void createOrder(Order order) {
        eventPublisher.publishEvent(new OrderCreatedEvent(order));
        
        // If listeners keep references to 'order', it won't be GC'd
    }
}

@Component
public class OrderListener {
    // This list grows forever!
    private List<Order> processedOrders = new ArrayList<>();
    
    @EventListener
    public void handleOrderCreated(OrderCreatedEvent event) {
        processedOrders.add(event.getOrder());  // Memory leak!
    }
}

// ✅ GOOD - Don't store unnecessary references
@Component
public class OrderListener {
    
    @EventListener
    public void handleOrderCreated(OrderCreatedEvent event) {
        // Process order immediately
        processOrder(event.getOrder());
        // No storage = no leak
    }
}
```

**3. Insufficient Heap Size**

```java
// Symptoms:
// - Memory usage is normal (no leak)
// - OOM happens during peak traffic
// - After GC, memory drops significantly

// Analysis:
// jstat shows: Old gen usage: 60-70% normally
// During peak: Old gen usage: 95-98%
// After Full GC: Old gen usage: 65%
// Conclusion: Need more heap, not a leak

// ✅ Solution: Increase heap size
// deployment.yaml
env:
  - name: JAVA_OPTS
    value: "-Xmx4g -Xms4g"  # Was 2g

// Or in Dockerfile
ENV JAVA_OPTS="-Xmx4g -Xms4g"
```

**4. Large Object Allocation**

```java
// ❌ BAD - Loading entire dataset into memory
@GetMapping("/export")
public ResponseEntity<byte[]> exportAllUsers() {
    List<User> users = userRepository.findAll();  // 10 million users!
    byte[] csv = convertToCsv(users);  // 5 GB in memory
    return ResponseEntity.ok(csv);
}

// ✅ GOOD - Stream processing
@GetMapping("/export")
public void exportAllUsers(HttpServletResponse response) {
    response.setContentType("text/csv");
    
    try (PrintWriter writer = response.getWriter()) {
        // Process in batches
        int pageSize = 1000;
        int page = 0;
        
        Page<User> users;
        do {
            users = userRepository.findAll(PageRequest.of(page++, pageSize));
            
            for (User user : users) {
                writer.println(convertToCsvLine(user));
            }
            
            users = null;  // Help GC
            
        } while (users.hasNext());
    }
}
```

**5. String Concatenation in Loop**

```java
// ❌ BAD - Creates many temporary String objects
public String generateReport(List<Record> records) {
    String report = "";
    for (Record record : records) {
        report += record.toString() + "\n";  // Creates new String each time!
    }
    return report;
}

// With 1 million records, this creates ~1 million String objects
// Each concatenation creates: original + new = temporary string

// ✅ GOOD - Use StringBuilder
public String generateReport(List<Record> records) {
    StringBuilder report = new StringBuilder(records.size() * 100);
    for (Record record : records) {
        report.append(record.toString()).append("\n");
    }
    return report.toString();
}
```

### **Solution Framework**

```bash
# Step 1: Identify if it's a leak or insufficient memory
# Run over time and check if old gen keeps growing

# If old gen keeps growing after Full GC:
#   → Memory Leak (fix code)

# If old gen drops after Full GC but fills up during peak:
#   → Insufficient memory (increase heap)

# Step 2: For memory leaks, analyze heap dump
# Find largest objects → Trace to code → Fix

# Step 3: For insufficient memory
# Calculate required heap:
#   Peak live data × 1.5 = Minimum heap
#   Peak live data × 3 = Comfortable heap

# Example:
# Peak live data: 2 GB
# Min heap: 3 GB
# Recommended: 6 GB

# Step 4: Optimize if needed
# - Reduce object creation
# - Use object pools
# - Implement caching wisely
# - Stream large datasets
```

---

## <a name="scenario-metaspace"></a>**6. Scenario 2: Metaspace/PermGen OOM**

### **Error Message**

```
// Java 8+
java.lang.OutOfMemoryError: Metaspace

// Java 7 and below
java.lang.OutOfMemoryError: PermGen space
```

### **What is Metaspace/PermGen?**

- **PermGen (Java 7-)**: Fixed-size area for class metadata
- **Metaspace (Java 8+)**: Native memory for class metadata (can grow)
- **Contains**: Class definitions, method metadata, static variables, constant pool

### **Symptoms**

- Application fails to load classes
- OOM during application redeployment
- OOM after many dynamic class generations
- "Could not initialize class" errors

### **Investigation Steps**

```bash
# 1. Check current metaspace usage (Java 8+)
jstat -gc <pid>
# Look at: M (Metaspace), CCS (Compressed Class Space)

# 2. Check metaspace limit
jinfo -flag MetaspaceSize <pid>
jinfo -flag MaxMetaspaceSize <pid>

# 3. Count loaded classes
jcmd <pid> GC.class_stats | wc -l

# Or use jstat
jstat -class <pid>
# Loaded: Number of classes loaded
# Bytes: Number of Kbytes loaded
# Unloaded: Number of classes unloaded
# Bytes: Number of Kbytes unloaded

# 4. Find classloader leaks
# Generate heap dump and analyze in MAT
# Histogram → Group by Class Loader
# Look for duplicate classes from different classloaders

# 5. Check which classes are loaded
jcmd <pid> GC.class_histogram | head -50

# 6. Monitor class loading over time
watch -n 5 'jstat -class <pid>'
```

### **Common Root Causes**

**1. Classloader Leak (Most Common)**

```java
// ❌ BAD - Classloader not released after undeploy
// Common in:
// - Application servers (Tomcat, JBoss) with hot redeployment
// - OSGi applications
// - Dynamic class loading frameworks

// Typical scenario:
// 1. Deploy application (ClassLoader A created)
// 2. Application registers listener/thread that references classes
// 3. Undeploy application (ClassLoader A should be GC'd)
// 4. But listener/thread still holds reference to ClassLoader A
// 5. ClassLoader A and all its classes stay in memory
// 6. Repeat 10 times → 10 classloaders in memory → Metaspace OOM

// Example leak:
@WebListener
public class AppListener implements ServletContextListener {
    
    private static final ThreadPoolExecutor executor = 
        Executors.newFixedThreadPool(10);  // Static! Never cleaned up
    
    @Override
    public void contextInitialized(ServletContextEvent event) {
        // Start background task
        executor.submit(() -> {
            while (true) {
                doWork();  // Holds reference to this classloader
                Thread.sleep(1000);
            }
        });
    }
    
    @Override
    public void contextDestroyed(ServletContextEvent event) {
        // Missing: executor.shutdown()
        // This thread keeps running, holding classloader reference!
    }
}

// ✅ GOOD - Properly cleanup resources
@WebListener
public class AppListener implements ServletContextListener {
    
    private ThreadPoolExecutor executor;
    
    @Override
    public void contextInitialized(ServletContextEvent event) {
        executor = (ThreadPoolExecutor) Executors.newFixedThreadPool(10);
        
        executor.submit(() -> {
            while (!Thread.currentThread().isInterrupted()) {
                doWork();
                Thread.sleep(1000);
            }
        });
    }
    
    @Override
    public void contextDestroyed(ServletContextEvent event) {
        if (executor != null) {
            executor.shutdownNow();
            try {
                executor.awaitTermination(10, TimeUnit.SECONDS);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }
    }
}
```

**2. Too Many Dynamically Generated Classes**

```java
// ❌ BAD - Unlimited dynamic proxy generation
public class DynamicProxyService {
    
    private Map<String, Object> proxies = new ConcurrentHashMap<>();
    
    public Object getProxy(String name, Class<?> interfaceClass) {
        return proxies.computeIfAbsent(name, k -> {
            // Each call generates a new class!
            return Proxy.newProxyInstance(
                interfaceClass.getClassLoader(),
                new Class<?>[] { interfaceClass },
                (proxy, method, args) -> {
                    System.out.println("Invoking: " + method.getName());
                    return null;
                }
            );
        });
    }
}

// With frameworks like:
// - Hibernate (entity proxies)
// - Spring (CGLIB proxies)  
// - Jackson (generated serializers)
// - Groovy/JRuby (dynamic compilation)

// ✅ GOOD - Limit and cache generated classes
public class DynamicProxyService {
    
    // Use bounded cache
    private final Cache<String, Object> proxies = Caffeine.newBuilder()
        .maximumSize(1000)
        .build();
    
    public Object getProxy(String name, Class<?> interfaceClass) {
        return proxies.get(name, k -> 
            Proxy.newProxyInstance(
                interfaceClass.getClassLoader(),
                new Class<?>[] { interfaceClass },
                (proxy, method, args) -> {
                    System.out.println("Invoking: " + method.getName());
                    return null;
                }
            )
        );
    }
}
```

**3. Too Many Classes in Application**

```java
// Sometimes the application legitimately has many classes

// Check: How many classes are loaded?
jcmd <pid> GC.class_stats | wc -l

// If > 50,000 classes, you might need more metaspace

// ✅ Solution: Increase metaspace limit
// For Java 8+:
java -XX:MetaspaceSize=256m \
     -XX:MaxMetaspaceSize=512m \
     -jar app.jar

// For Java 7:
java -XX:PermSize=256m \
     -XX:MaxPermSize=512m \
     -jar app.jar

// Kubernetes deployment:
env:
  - name: JAVA_OPTS
    value: "-XX:MaxMetaspaceSize=512m"
```

**4. Reflection and Annotation Processing**

```java
// ❌ BAD - Excessive reflection in hot path
@RestController
public class DynamicController {
    
    @GetMapping("/invoke/{method}")
    public Object invoke(@PathVariable String method) throws Exception {
        // This can generate classes for each invocation
        Method m = SomeService.class.getMethod(method);
        return m.invoke(serviceInstance);
    }
}

// ✅ GOOD - Cache reflection results
@RestController
public class DynamicController {
    
    private final Map<String, Method> methodCache = new ConcurrentHashMap<>();
    
    @PostConstruct
    public void init() {
        // Pre-populate cache at startup
        for (Method method : SomeService.class.getMethods()) {
            methodCache.put(method.getName(), method);
        }
    }
    
    @GetMapping("/invoke/{method}")
    public Object invoke(@PathVariable String method) throws Exception {
        Method m = methodCache.get(method);
        if (m == null) {
            throw new IllegalArgumentException("Unknown method");
        }
        return m.invoke(serviceInstance);
    }
}
```

### **Solution Checklist**

```bash
# 1. For classloader leaks:
☐ Properly shutdown thread pools on context destroy
☐ Unregister JDBC drivers
☐ Clear ThreadLocal variables
☐ Stop scheduled tasks
☐ Close file handles and streams
☐ Remove event listeners

# 2. For dynamic class generation:
☐ Use caching for generated classes
☐ Limit the number of generated classes
☐ Review framework configurations (Spring, Hibernate)

# 3. For large applications:
☐ Increase metaspace size appropriately
☐ Monitor metaspace usage over time
☐ Consider splitting into microservices

# 4. For development/testing:
☐ Set MaxMetaspaceSize to prevent unlimited growth
☐ Use -XX:+TraceClassLoading to debug
☐ Analyze heap dump for duplicate classes
```

---

## <a name="scenario-thread"></a>**7. Scenario 4: Unable to Create New Native Thread**

### **Error Message**

```
java.lang.OutOfMemoryError: unable to create new native thread
```

### **What This Means**

- JVM tried to create a new thread but OS refused
- NOT a heap memory issue!
- Each thread consumes: Stack memory + OS resources

### **Symptoms**

- Application stops processing new requests
- "Cannot create thread" errors in logs
- High thread count in monitoring
- System becomes unresponsive

### **Investigation Steps**

```bash
# 1. Count threads in Java process
jstack <pid> | grep "java.lang.Thread.State" | wc -l

# Or
ps -eLf | grep <pid> | wc -l

# 2. Check OS thread limit (Linux)
ulimit -u  # Max user processes
cat /proc/sys/kernel/threads-max  # System-wide limit

# 3. Get thread dump
jstack <pid> > threads.txt

# Analyze thread dump:
# Count thread states
grep "java.lang.Thread.State" threads.txt | sort | uniq -c

# 4. Find thread names and what they're doing
cat threads.txt | grep -A 2 "Thread-"

# 5. Check for common thread leaks
grep "pool-" threads.txt | wc -l  # Thread pool threads
grep "Abandoned connection cleanup thread" threads.txt | wc -l  # DB threads

# 6. In Kubernetes
kubectl exec <pod-name> -n <namespace> -- sh -c 'ps -eLf | wc -l'
```

### **Root Cause Analysis**

**Calculate Thread Memory**

```bash
# Each thread uses:
Thread Stack Size (-Xss) × Number of Threads

# Example:
# Stack size: 1 MB (default on 64-bit)
# Threads: 5000
# Memory: 5000 × 1 MB = 5 GB just for stacks!

# Check current stack size:
jinfo -flag ThreadStackSize <pid>

# Total process memory equation:
Heap + Metaspace + (ThreadCount × StackSize) + DirectMemory + Native ≤ Container Memory
```

**Why Thread Creation Fails**

```
1. OS Process Limit Reached
   ulimit -u = 4096
   Current threads = 4000
   → Almost at limit

2. Insufficient Memory for Stack
   Container memory: 2 GB
   Heap: 1.5 GB
   Stack size: 1 MB per thread
   Current threads: 600
   600 MB for stacks + 1.5 GB heap = 2.1 GB
   → Exceeds container limit

3. System-Wide Thread Limit
   /proc/sys/kernel/threads-max = 30000
   System threads = 29500
   → Almost at system limit
```

### **Common Root Causes**

**1. Thread Pool Not Bounded**

```java
// ❌ BAD - Unbounded thread pool
@Configuration
public class ThreadPoolConfig {
    
    @Bean
    public Executor taskExecutor() {
        // This can create unlimited threads!
        return Executors.newCachedThreadPool();
    }
}

// What happens:
// 1. High traffic → 1000 concurrent requests
// 2. Each request creates a thread
// 3. 1000 threads created
// 4. More traffic → more threads
// 5. Eventually: unable to create new native thread

// ✅ GOOD - Bounded thread pool with queue
@Configuration
public class ThreadPoolConfig {
    
    @Bean
    public Executor taskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(10);
        executor.setMaxPoolSize(50);
        executor.setQueueCapacity(1000);
        executor.setThreadNamePrefix("async-");
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        executor.initialize();
        return executor;
    }
}
```

**2. Thread Leaks**

```java
// ❌ BAD - Threads never stop
public class BackgroundTaskManager {
    
    public void startTask() {
        // Creates new thread every time!
        new Thread(() -> {
            while (true) {  // Runs forever
                doWork();
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    // Ignores interrupt!
                    e.printStackTrace();
                }
            }
        }).start();
    }
}

// After 1000 calls: 1000 threads running forever!

// ✅ GOOD - Use scheduled executor with proper shutdown
public class BackgroundTaskManager {
    
    private final ScheduledExecutorService scheduler = 
        Executors.newScheduledThreadPool(1);
    
    public void startTask() {
        scheduler.scheduleAtFixedRate(() -> {
            doWork();
        }, 0, 1, TimeUnit.SECONDS);
    }
    
    @PreDestroy
    public void shutdown() {
        scheduler.shutdown();
        try {
            if (!scheduler.awaitTermination(10, TimeUnit.SECONDS)) {
                scheduler.shutdownNow();
            }
        } catch (InterruptedException e) {
            scheduler.shutdownNow();
            Thread.currentThread().interrupt();
        }
    }
}
```

**3. Database Connection Pool Threads**

```java
// ❌ BAD - Too many database connections
spring:
  datasource:
    hikari:
      maximum-pool-size: 500  # Too many!

// Each connection = background thread
// 500 connections = 500+ threads

// ✅ GOOD - Reasonable pool size
spring:
  datasource:
    hikari:
      maximum-pool-size: 20  # Reasonable for most apps
      minimum-idle: 5
      max-lifetime: 1800000
      connection-timeout: 30000
```

**4. HTTP Client Threads**

```java
// ❌ BAD - New thread per request
public class ExternalApiClient {
    
    public void callApi(Request request) {
        new Thread(() -> {
            HttpClient client = HttpClient.newHttpClient();
            // Make request...
        }).start();
    }
}

// ✅ GOOD - Reuse HTTP client and executor
@Component
public class ExternalApiClient {
    
    private final HttpClient httpClient;
    private final ExecutorService executor;
    
    public ExternalApiClient() {
        this.executor = Executors.newFixedThreadPool(10);
        this.httpClient = HttpClient.newBuilder()
            .executor(executor)
            .connectTimeout(Duration.ofSeconds(10))
            .build();
    }
    
    public CompletableFuture<HttpResponse<String>> callApi(Request request) {
        return httpClient.sendAsync(
            HttpRequest.newBuilder()
                .uri(request.getUri())
                .build(),
            HttpResponse.BodyHandlers.ofString()
        );
    }
    
    @PreDestroy
    public void shutdown() {
        executor.shutdown();
    }
}
```

### **Solutions**

**Immediate Fix**

```bash
# 1. Increase OS thread limit
ulimit -u 65535

# Make permanent: /etc/security/limits.conf
* soft nproc 65535
* hard nproc 65535

# 2. In Kubernetes, set higher limits
# deployment.yaml
spec:
  template:
    spec:
      containers:
      - name: app
        resources:
          limits:
            memory: "4Gi"  # More memory for stacks

# 3. Reduce stack size (use with caution!)
env:
  - name: JAVA_OPTS
    value: "-Xss512k"  # Default is 1MB
```

**Long-term Fix**

```java
// 1. Audit all thread creation
// Search codebase for:
// - new Thread(
// - Executors.newCachedThreadPool()
// - Executors.newFixedThreadPool() without proper sizing

// 2. Implement thread pool monitoring
@Component
public class ThreadPoolMonitor {
    
    @Autowired
    private List<ThreadPoolTaskExecutor> executors;
    
    @Scheduled(fixedDelay = 60000)
    public void monitorPools() {
        for (ThreadPoolTaskExecutor executor : executors) {
            ThreadPoolExecutor tpe = executor.getThreadPoolExecutor();
            
            log.info("Pool: {}, Active: {}/{}, Queue: {}/{}, Completed: {}",
                executor.getThreadNamePrefix(),
                tpe.getActiveCount(),
                tpe.getMaximumPoolSize(),
                tpe.getQueue().size(),
                ((LinkedBlockingQueue<?>) tpe.getQueue()).remainingCapacity(),
                tpe.getCompletedTaskCount()
            );
            
            // Alert if pool is saturated
            if (tpe.getActiveCount() >= tpe.getMaximumPoolSize() * 0.9) {
                alertService.sendAlert("Thread pool near capacity: " + 
                    executor.getThreadNamePrefix());
            }
        }
    }
    
    @Scheduled(fixedDelay = 300000)
    public void checkTotalThreads() {
        ThreadMXBean threadBean = ManagementFactory.getThreadMXBean();
        int threadCount = threadBean.getThreadCount();
        
        log.info("Total JVM threads: {}", threadCount);
        
        if (threadCount > 1000) {
            log.error("WARNING: Thread count is very high: {}", threadCount);
            
            // Get thread dump for analysis
            long[] threadIds = threadBean.getAllThreadIds();
            Map<String, Integer> threadsByName = new HashMap<>();
            
            for (long threadId : threadIds) {
                ThreadInfo info = threadBean.getThreadInfo(threadId);
                if (info != null) {
                    String name = info.getThreadName();
                    String prefix = name.split("-")[0];
                    threadsByName.merge(prefix, 1, Integer::sum);
                }
            }
            
            log.error("Thread breakdown: {}", threadsByName);
        }
    }
}
```

---

## <a name="scenario-gc-overhead"></a>**8. Scenario 5: GC Overhead Limit Exceeded**

### **Error Message**

```
java.lang.OutOfMemoryError: GC overhead limit exceeded
```

### **What This Means**

- JVM is spending > 98% of time in GC
- Less than 2% of heap is being freed
- Application is essentially frozen doing GC
- This is a pre-emptive OOM to prevent complete freeze

### **Symptoms**

- Application becomes extremely slow before crash
- Very high CPU usage (GC threads)
- Frequent Full GC events in logs
- Eventually crashes with OOM

### **Investigation**

```bash
# 1. Check GC activity
jstat -gcutil <pid> 1000 10

# If you see:
# FGC (Full GC count) increasing rapidly
# FGCT (Full GC time) very high
# E, O (Eden, Old) at 95%+ after GC
# → This is GC overhead

# 2. Analyze GC logs
grep "Full GC" gc.log | tail -20

# Bad pattern example:
# [Full GC 1.5GB->1.4GB(1.5GB), 2.341 secs]  ← Only freed 100MB
# [Full GC 1.5GB->1.45GB(1.5GB), 2.512 secs] ← Only freed 50MB
# [Full GC 1.5GB->1.48GB(1.5GB), 2.734 secs] ← Only freed 20MB
# [GC overhead limit exceeded]

# 3. Generate heap dump to see what's consuming memory
jmap -dump:live,format=b,file=heap-gc-overhead.hprof <pid>
```

### **Root Causes**

**1. Heap Too Small for Working Set**

```java
// Application needs 1.2 GB for normal operation
// But heap is set to 1.5 GB
// JVM spends all time trying to keep heap below limit

// ✅ Solution: Increase heap size
// Rule of thumb: heap should be 3x your working set
// Working set: 1.2 GB
// Heap: 3.6 GB minimum

// Before:
java -Xmx1536m -jar app.jar

// After:
java -Xmx4g -Xms4g -jar app.jar
```

**2. Memory Leak Approaching Limit**

```java
// Memory leak has filled heap to 95%
// GC can't free much memory
// Application is at breaking point

// Analyze heap dump to find leak
// Fix the leak (see Heap OOM scenario)
```

**3. Too Many Live Objects**

```java
// ❌ BAD - Loading too much data at once
@GetMapping("/report")
public Report generateReport() {
    // Loads 1 million records into memory
    List<Transaction> transactions = transactionRepository.findAll();
    
    // Processes all at once
    Report report = processAll(transactions);
    
    return report;
}

// ✅ GOOD - Stream processing
@GetMapping("/report")
public void generateReport(HttpServletResponse response) {
    response.setContentType("application/json");
    
    try (PrintWriter writer = response.getWriter()) {
        writer.write("[");
        
        // Process in batches
        int pageSize = 1000;
        int page = 0;
        boolean first = true;
        
        Page<Transaction> transactions;
        do {
            transactions = transactionRepository.findAll(
                PageRequest.of(page++, pageSize)
            );
            
            for (Transaction txn : transactions) {
                if (!first) writer.write(",");
                writer.write(toJson(txn));
                first = false;
            }
            
        } while (transactions.hasNext());
        
        writer.write("]");
    }
}
```

### **Solutions**

```bash
# 1. Disable the check if you want to let application crash naturally
# (not recommended)
java -XX:-UseGCOverheadLimit -jar app.jar

# 2. Tune GC for better throughput
# Use G1GC (default in Java 9+)
java -XX:+UseG1GC \
     -XX:MaxGCPauseMillis=200 \
     -Xmx4g \
     -jar app.jar

# 3. Increase heap size
# Calculate: current heap × (100 / (100 - current_usage_percent))
# If current: 1.5GB with 95% used after GC
# New heap: 1.5 × (100 / (100 - 95)) = 1.5 × 20 = 30GB? No!
# More realistic: 1.5 × (100 / (100 - 60)) = 3.75GB

# 4. Fix memory leaks or optimize memory usage
```

---

## <a name="leak-patterns"></a>**9. Memory Leak Patterns**

### **Pattern 1: Static Collection Growing**

```java
// ❌ LEAK: Static collection that grows forever
public class SessionManager {
    private static final Map<String, UserSession> sessions = new HashMap<>();
    
    public void addSession(String sessionId, UserSession session) {
        sessions.put(sessionId, session);
        // Never removed! Leak!
    }
}

// ✅ FIX: Use cache with eviction
public class SessionManager {
    private static final Cache<String, UserSession> sessions = Caffeine.newBuilder()
        .maximumSize(10_000)
        .expireAfterAccess(Duration.ofMinutes(30))
        .build();
    
    public void addSession(String sessionId, UserSession session) {
        sessions.put(sessionId, session);
    }
}
```

### **Pattern 2: ThreadLocal Not Cleaned**

```java
// ❌ LEAK: ThreadLocal holds large objects
public class RequestContext {
    private static final ThreadLocal<UserContext> context = new ThreadLocal<>();
    
    public static void setContext(UserContext user) {
        context.set(user);
        // If never removed, stays in thread for lifetime!
    }
}

// In thread pool, threads are reused:
// Thread-1: Request A → sets context → request ends (context not cleared)
// Thread-1: Request B → old context still there!
// After 1000 requests: 1000 contexts in 10 threads

// ✅ FIX: Always remove ThreadLocal
public class RequestContext {
    private static final ThreadLocal<UserContext> context = new ThreadLocal<>();
    
    public static void setContext(UserContext user) {
        context.set(user);
    }
    
    public static void clear() {
        context.remove();
    }
}

@Component
public class RequestContextFilter implements Filter {
    
    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) 
        throws IOException, ServletException {
        try {
            // Set context
            RequestContext.setContext(extractUserContext(request));
            chain.doFilter(request, response);
        } finally {
            // Always clean up!
            RequestContext.clear();
        }
    }
}
```

### **Pattern 3: Listener Not Removed**

```java
// ❌ LEAK: Listeners keep references to objects
public class ReportService {
    private List<ReportData> data = new ArrayList<>();
    
    public void generateReport() {
        EventBus.register(this);  // Registers listener
        
        // Generate report...
        
        // Missing: EventBus.unregister(this)
        // EventBus keeps reference to 'this'
        // 'this' keeps reference to 'data'
        // 'data' is never GC'd
    }
}

// ✅ FIX: Always unregister
public class ReportService {
    private List<ReportData> data = new ArrayList<>();
    
    public void generateReport() {
        try {
            EventBus.register(this);
            // Generate report...
        } finally {
            EventBus.unregister(this);
        }
    }
}
```

### **Pattern 4: Unclosed Resources**

```java
// ❌ LEAK: Stream not closed
public void processFile(String filename) {
    InputStream input = new FileInputStream(filename);
    // Process file...
    // Never closed! File handle leaked!
}

// ✅ FIX: Use try-with-resources
public void processFile(String filename) {
    try (InputStream input = new FileInputStream(filename)) {
        // Process file...
    } // Automatically closed
}

// ❌ LEAK: Database connections not closed
public List<User> getUsers() {
    Connection conn = dataSource.getConnection();
    Statement stmt = conn.createStatement();
    ResultSet rs = stmt.executeQuery("SELECT * FROM users");
    
    List<User> users = new ArrayList<>();
    while (rs.next()) {
        users.add(mapUser(rs));
    }
    return users;
    // conn, stmt, rs never closed!
}

// ✅ FIX: Close everything
public List<User> getUsers() {
    try (Connection conn = dataSource.getConnection();
         Statement stmt = conn.createStatement();
         ResultSet rs = stmt.executeQuery("SELECT * FROM users")) {
        
        List<User> users = new ArrayList<>();
        while (rs.next()) {
            users.add(mapUser(rs));
        }
        return users;
    } catch (SQLException e) {
        throw new RuntimeException(e);
    }
}
```

### **Pattern 5: Inner Class Holding Reference**

```java
// ❌ LEAK: Inner class holds implicit reference to outer class
public class LargeObject {
    private byte[] data = new byte[10_000_000];  // 10 MB
    
    public Runnable createTask() {
        return new Runnable() {  // Anonymous inner class
            @Override
            public void run() {
                System.out.println("Task running");
                // This inner class holds reference to LargeObject!
            }
        };
    }
}

// If task is stored somewhere long-term:
List<Runnable> tasks = new ArrayList<>();
for (int i = 0; i < 1000; i++) {
    LargeObject obj = new LargeObject();
    tasks.add(obj.createTask());  // Keeps 1000 × 10MB = 10 GB in memory!
}

// ✅ FIX: Use static inner class or lambda with explicit references
public class LargeObject {
    private byte[] data = new byte[10_000_000];
    
    public Runnable createTask() {
        // Lambda doesn't capture 'this' if not used
        return () -> System.out.println("Task running");
    }
}

// Or if you need data from outer class:
public class LargeObject {
    private byte[] data = new byte[10_000_000];
    
    public Runnable createTask(String taskName) {
        // Only capture what you need
        return () -> System.out.println("Task running: " + taskName);
        // Doesn't hold reference to entire LargeObject
    }
}
```

### **Pattern 6: Caching Without Limits**

```java
// ❌ LEAK: Unlimited cache
@Component
public class UserCache {
    private final Map<Long, User> cache = new ConcurrentHashMap<>();
    
    public User getUser(Long id) {
        return cache.computeIfAbsent(id, this::loadUser);
    }
    
    private User loadUser(Long id) {
        return userRepository.findById(id).orElse(null);
    }
}

// After 1 million user lookups: 1 million users cached!

// ✅ FIX: Use bounded cache
@Component
public class UserCache {
    private final Cache<Long, User> cache = Caffeine.newBuilder()
        .maximumSize(10_000)
        .expireAfterWrite(Duration.ofHours(1))
        .recordStats()
        .build();
    
    public User getUser(Long id) {
        return cache.get(id, this::loadUser);
    }
    
    private User loadUser(Long id) {
        return userRepository.findById(id).orElse(null);
    }
    
    @Scheduled(fixedDelay = 60000)
    public void logStats() {
        CacheStats stats = cache.stats();
        log.info("Cache stats - hits: {}, misses: {}, size: {}",
            stats.hitCount(), stats.missCount(), cache.estimatedSize());
    }
}
```

---

## <a name="anti-patterns"></a>**10. Anti-Patterns That Cause OOM**

### **Anti-Pattern 1: String Concatenation in Loop**

```java
// ❌ Creates thousands of temporary String objects
public String buildCsv(List<Record> records) {
    String csv = "";
    for (Record record : records) {
        csv += record.getId() + "," + record.getName() + "\n";
    }
    return csv;
}

// Each += creates a new String:
// Iteration 1: "" + data → String 1
// Iteration 2: String 1 + data → String 2 (String 1 now garbage)
// Iteration 3: String 2 + data → String 3 (String 2 now garbage)
// ...
// With 1M records: creates ~1M temporary strings

// ✅ Use StringBuilder
public String buildCsv(List<Record> records) {
    StringBuilder csv = new StringBuilder(records.size() * 50);
    for (Record record : records) {
        csv.append(record.getId())
           .append(",")
           .append(record.getName())
           .append("\n");
    }
    return csv.toString();
}
```

### **Anti-Pattern 2: Loading Entire Dataset**

```java
// ❌ Loads everything into memory
@GetMapping("/users/export")
public List<UserDTO> exportUsers() {
    List<User> users = userRepository.findAll();  // 10 million users!
    return users.stream()
        .map(this::toDTO)
        .collect(Collectors.toList());
}

// ✅ Use pagination or streaming
@GetMapping("/users/export")
public void exportUsers(HttpServletResponse response) throws IOException {
    response.setContentType("application/json");
    
    try (JsonGenerator json = objectMapper.getFactory()
            .createGenerator(response.getOutputStream())) {
        
        json.writeStartArray();
        
        int page = 0;
        int pageSize = 1000;
        Page<User> users;
        
        do {
            users = userRepository.findAll(PageRequest.of(page++, pageSize));
            
            for (User user : users) {
                json.writeObject(toDTO(user));
            }
            
        } while (users.hasNext());
        
        json.writeEndArray();
    }
}
```

### **Anti-Pattern 3: Regex on Large Strings**

```java
// ❌ Regex with catastrophic backtracking
public boolean validate(String input) {
    // This regex can take exponential time!
    return input.matches("(a+)+b");
}

// With malicious input: "aaaaaaaaaaaaaaaaaaaaaaaac"
// Can cause CPU spike and memory issues

// ✅ Use simple validation or bounded regex
public boolean validate(String input) {
    // Simple length check first
    if (input.length() > 1000) {
        return false;
    }
    
    // Use simpler regex
    return input.matches("a+b");
}
```

### **Anti-Pattern 4: Reflection in Hot Path**

```java
// ❌ Reflection in loop
public void processRecords(List<Record> records) {
    for (Record record : records) {
        try {
            Method method = record.getClass().getMethod("process");
            method.invoke(record);
        } catch (Exception e) {
            // Handle
        }
    }
}

// ✅ Cache reflection results
public class RecordProcessor {
    private final Map<Class<?>, Method> methodCache = new ConcurrentHashMap<>();
    
    public void processRecords(List<Record> records) {
        for (Record record : records) {
            Method method = methodCache.computeIfAbsent(
                record.getClass(),
                clazz -> {
                    try {
                        return clazz.getMethod("process");
                    } catch (NoSuchMethodException e) {
                        throw new RuntimeException(e);
                    }
                }
            );
            
            try {
                method.invoke(record);
            } catch (Exception e) {
                // Handle
            }
        }
    }
}
```

### **Anti-Pattern 5: Autoboxing in Loops**

```java
// ❌ Creates millions of Integer objects
public long sum(List<Integer> numbers) {
    Long sum = 0L;  // Using Long (object) instead of long (primitive)
    for (Integer num : numbers) {
        sum += num;  // Autoboxing/unboxing on every iteration!
    }
    return sum;
}

// With 1M numbers: creates 1M Long objects

// ✅ Use primitives
public long sum(List<Integer> numbers) {
    long sum = 0L;  // Primitive
    for (Integer num : numbers) {
        sum += num;
    }
    return sum;
}

// ✅ Even better: use streams with primitive types
public long sum(List<Integer> numbers) {
    return numbers.stream()
        .mapToLong(Integer::longValue)
        .sum();
}
```

---

## <a name="best-practices"></a>**11. Best Practices**

### **Memory Management Best Practices**

```java
// 1. Always close resources
try (InputStream in = new FileInputStream(file);
     OutputStream out = new FileOutputStream(outFile)) {
    // Use streams
} // Automatically closed

// 2. Avoid premature optimization
// But be aware of:
// - Object creation in hot paths
// - String concatenation in loops
// - Unnecessary boxing/unboxing

// 3. Use appropriate collections
// ArrayList vs LinkedList
ArrayList<String> list1 = new ArrayList<>();  // Better for random access
LinkedList<String> list2 = new LinkedList<>();  // Better for insertions

// HashMap vs TreeMap
HashMap<String, String> map1 = new HashMap<>();  // O(1) lookup
TreeMap<String, String> map2 = new TreeMap<>();  // O(log n) but sorted

// 4. Set initial capacity when size is known
List<String> list = new ArrayList<>(expectedSize);
Map<String, String> map = new HashMap<>(expectedSize);
StringBuilder sb = new StringBuilder(expectedSize);

// 5. Use WeakReference for caches
Map<String, WeakReference<CachedData>> cache = new WeakHashMap<>();

// 6. Implement proper equals() and hashCode()
// Objects used as keys must have good hashCode distribution

// 7. Use object pools for expensive objects
// But only when object creation is proven bottleneck
```

### **JVM Configuration Best Practices**

```bash
# 1. Set Xms equal to Xmx (avoids heap resizing)
-Xms4g -Xmx4g

# 2. Enable heap dump on OOM
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/opt/dumps/

# 3. Use appropriate GC
# For low latency (<100ms pauses):
-XX:+UseG1GC -XX:MaxGCPauseMillis=50

# For throughput:
-XX:+UseParallelGC

# For very large heaps (>32GB):
-XX:+UseZGC  # Java 15+

# 4. Enable GC logging
# Java 8:
-Xloggc:gc.log -XX:+PrintGCDetails -XX:+PrintGCDateStamps

# Java 9+:
-Xlog:gc*:file=gc.log:time,uptime,level,tags

# 5. Set metaspace limit
-XX:MaxMetaspaceSize=512m

# 6. Monitor with JMX
-Dcom.sun.management.jmxremote
-Dcom.sun.management.jmxremote.port=9010
-Dcom.sun.management.jmxremote.authenticate=false
-Dcom.sun.management.jmxremote.ssl=false

# 7. Enable Native Memory Tracking
-XX:NativeMemoryTracking=summary

# 8. Set reasonable thread stack size
-Xss512k  # Default is 1MB

# Complete example:
java -Xms4g -Xmx4g \
     -XX:+UseG1GC \
     -XX:MaxGCPauseMillis=200 \
     -XX:+HeapDumpOnOutOfMemoryError \
     -XX:HeapDumpPath=/opt/dumps/ \
     -Xlog:gc*:file=gc.log:time,uptime,level,tags \
     -XX:MaxMetaspaceSize=512m \
     -XX:NativeMemoryTracking=summary \
     -jar application.jar
```

---

## <a name="monitoring"></a>**12. Monitoring & Alerting**

### **Key Metrics to Monitor**

```yaml
# Prometheus metrics for Java/Spring Boot

# 1. Heap Memory
- jvm_memory_used_bytes{area="heap"}
- jvm_memory_max_bytes{area="heap"}
- (jvm_memory_used_bytes{area="heap"} / jvm_memory_max_bytes{area="heap"}) * 100

# 2. Non-Heap Memory (Metaspace, etc.)
- jvm_memory_used_bytes{area="nonheap"}
- jvm_memory_max_bytes{area="nonheap"}

# 3. GC Metrics
- jvm_gc_pause_seconds_count
- jvm_gc_pause_seconds_sum
- jvm_gc_memory_allocated_bytes_total
- jvm_gc_memory_promoted_bytes_total

# 4. Thread Metrics
- jvm_threads_live_threads
- jvm_threads_daemon_threads
- jvm_threads_peak_threads
- jvm_threads_states_threads{state="blocked"}
- jvm_threads_states_threads{state="waiting"}

# 5. Class Loading
- jvm_classes_loaded_classes
- jvm_classes_unloaded_classes_total

# 6. Buffer Pools (Direct Memory)
- jvm_buffer_memory_used_bytes{id="direct"}
- jvm_buffer_count_buffers{id="direct"}
```

### **Alert Rules**

```yaml
groups:
  - name: jvm_alerts
    interval: 30s
    rules:
      # High heap usage
      - alert: JVMHeapUsageHigh
        expr: (jvm_memory_used_bytes{area="heap"} / jvm_memory_max_bytes{area="heap"}) > 0.85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "JVM heap usage above 85%"
          description: "{{ $labels.instance }} heap usage is {{ $value | humanizePercentage }}"
      
      # Critical heap usage
      - alert: JVMHeapUsageCritical
        expr: (jvm_memory_used_bytes{area="heap"} / jvm_memory_max_bytes{area="heap"}) > 0.95
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "JVM heap usage above 95% - OOM imminent"
          description: "{{ $labels.instance }} heap usage is {{ $value | humanizePercentage }}"
      
      # High GC time
      - alert: JVMGCTimeHigh
        expr: rate(jvm_gc_pause_seconds_sum[5m]) > 0.1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "JVM spending >10% time in GC"
          description: "{{ $labels.instance }} GC time is {{ $value | humanizePercentage }}"
      
      # Frequent Full GC
      - alert: JVMFrequentFullGC
        expr: rate(jvm_gc_pause_seconds_count{action="end of major GC"}[5m]) > 0.01
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Frequent Full GC events detected"
      
      # High thread count
      - alert: JVMThreadCountHigh
        expr: jvm_threads_live_threads > 1000
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "JVM has more than 1000 threads"
          description: "{{ $labels.instance }} has {{ $value }} threads"
      
      # Thread leak detection
      - alert: JVMThreadLeak
        expr: increase(jvm_threads_live_threads[30m]) > 100
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Possible thread leak detected"
          description: "Thread count increased by {{ $value }} in 30 minutes"
      
      # Metaspace usage high
      - alert: JVMMetaspaceUsageHigh
        expr: (jvm_memory_used_bytes{area="nonheap",id="Metaspace"} / jvm_memory_max_bytes{area="nonheap",id="Metaspace"}) > 0.90
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Metaspace usage above 90%"
```

### **Monitoring Dashboard (Grafana)**

```json
{
  "dashboard": {
    "title": "JVM Memory Monitoring",
    "panels": [
      {
        "title": "Heap Memory Usage",
        "targets": [
          {
            "expr": "jvm_memory_used_bytes{area='heap'}",
            "legendFormat": "Used"
          },
          {
            "expr": "jvm_memory_max_bytes{area='heap'}",
            "legendFormat": "Max"
          }
        ]
      },
      {
        "title": "GC Pause Time",
        "targets": [
          {
            "expr": "rate(jvm_gc_pause_seconds_sum[5m])",
            "legendFormat": "{{gc}}"
          }
        ]
      },
      {
        "title": "Thread Count",
        "targets": [
          {
            "expr": "jvm_threads_live_threads",
            "legendFormat": "Live Threads"
          }
        ]
      },
      {
        "title": "Object Allocation Rate",
        "targets": [
          {
            "expr": "rate(jvm_gc_memory_allocated_bytes_total[5m])",
            "legendFormat": "Allocation Rate"
          }
        ]
      }
    ]
  }
}
```

### **Custom Monitoring in Application**

```java
@Component
@Slf4j
public class MemoryMonitor {
    
    private final MeterRegistry meterRegistry;
    
    public MemoryMonitor(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
        
        // Register custom gauges
        Gauge.builder("custom.memory.usage.percent", this::getHeapUsagePercent)
            .description("Heap memory usage percentage")
            .register(meterRegistry);
        
        Gauge.builder("custom.old.gen.usage.percent", this::getOldGenUsagePercent)
            .description("Old generation usage percentage")
            .register(meterRegistry);
    }
    
    private double getHeapUsagePercent() {
        MemoryMXBean memoryBean = ManagementFactory.getMemoryMXBean();
        MemoryUsage heapUsage = memoryBean.getHeapMemoryUsage();
        return ((double) heapUsage.getUsed() / heapUsage.getMax()) * 100;
    }
    
    private double getOldGenUsagePercent() {
        for (MemoryPoolMXBean pool : ManagementFactory.getMemoryPoolMXBeans()) {
            if (pool.getType() == MemoryType.HEAP && 
                pool.getName().contains("Old") || pool.getName().contains("Tenured")) {
                MemoryUsage usage = pool.getUsage();
                return ((double) usage.getUsed() / usage.getMax()) * 100;
            }
        }
        return 0;
    }
    
    @Scheduled(fixedDelay = 60000)
    public void logMemoryStats() {
        MemoryMXBean memoryBean = ManagementFactory.getMemoryMXBean();
        MemoryUsage heapUsage = memoryBean.getHeapMemoryUsage();
        MemoryUsage nonHeapUsage = memoryBean.getNonHeapMemoryUsage();
        
        log.info("Memory Stats - Heap: {} MB / {} MB ({}%), Non-Heap: {} MB",
            heapUsage.getUsed() / 1024 / 1024,
            heapUsage.getMax() / 1024 / 1024,
            String.format("%.2f", getHeapUsagePercent()),
            nonHeapUsage.getUsed() / 1024 / 1024
        );
        
        // Alert if high
        if (getHeapUsagePercent() > 90) {
            log.error("⚠️ WARNING: Heap usage above 90%!");
            // Trigger alert
            triggerAlert("High Heap Usage", getHeapUsagePercent());
        }
    }
    
    @Scheduled(fixedDelay = 300000)  // Every 5 minutes
    public void logGCStats() {
        for (GarbageCollectorMXBean gc : ManagementFactory.getGarbageCollectorMXBeans()) {
            long count = gc.getCollectionCount();
            long time = gc.getCollectionTime();
            
            log.info("GC Stats - {}: count={}, time={}ms, avg={}ms",
                gc.getName(),
                count,
                time,
                count > 0 ? time / count : 0
            );
        }
    }
    
    @Scheduled(fixedDelay = 300000)  // Every 5 minutes
    public void checkForMemoryLeak() {
        MemoryMXBean memoryBean = ManagementFactory.getMemoryMXBean();
        MemoryUsage heapUsage = memoryBean.getHeapMemoryUsage();
        
        long usedMemory = heapUsage.getUsed();
        
        // Store in time series
        recordMemoryUsage(usedMemory);
        
        // Check if memory is steadily increasing
        if (isMemoryLeakDetected()) {
            log.error("⚠️ Possible memory leak detected!");
            triggerAlert("Possible Memory Leak", usedMemory);
        }
    }
    
    private void recordMemoryUsage(long usedMemory) {
        // Implementation
    }
    
    private boolean isMemoryLeakDetected() {
        // Check if memory usage is steadily increasing over time
        // Implementation
        return false;
    }
    
    private void triggerAlert(String message, double value) {
        // Send alert via Slack, PagerDuty, etc.
    }
}
```

---

## **13. Quick Reference Checklist**

### **When OOM Occurs**

```bash
☐ 1. Identify OOM type from error message
☐ 2. Check if heap dump was generated
☐ 3. If not, generate heap dump (if process still running)
☐ 4. Collect GC logs
☐ 5. Get thread dump
☐ 6. Review application logs before crash
☐ 7. Check monitoring dashboards
☐ 8. Analyze heap dump in MAT/VisualVM
☐ 9. Identify root cause (leak vs insufficient memory)
☐ 10. Implement fix
☐ 11. Test fix in staging
☐ 12. Deploy to production
☐ 13. Monitor closely
☐ 14. Document incident and learnings
```

### **Preventive Measures**

```bash
☐ Set appropriate JVM flags (-Xmx, -Xms, -XX:MaxMetaspaceSize)
☐ Enable heap dump on OOM
☐ Enable GC logging
☐ Set up memory monitoring and alerting
☐ Perform regular heap dump analysis in staging
☐ Load test with realistic data volumes
☐ Review code for common leak patterns
☐ Use static analysis tools (SonarQube, SpotBugs)
☐ Implement resource cleanup (close streams, remove listeners)
☐ Use bounded caches
☐ Clean up ThreadLocal variables
☐ Properly manage thread pools
☐ Regular performance testing
☐ Monitor production metrics
```

---

## **14. Tools Summary**

| Tool | Purpose | Command |
|------|---------|---------|
| jps | List Java processes | `jps -lv` |
| jstat | GC and memory stats | `jstat -gcutil <pid> 1000` |
| jmap | Heap dump | `jmap -dump:format=b,file=heap.hprof <pid>` |
| jstack | Thread dump | `jstack <pid>` |
| jcmd | All-in-one diagnostic | `jcmd <pid> help` |
| jinfo | Configuration | `jinfo -flags <pid>` |
| MAT | Heap dump analysis | Eclipse Memory Analyzer |
| VisualVM | Profiling | VisualVM |
| GCViewer | GC log analysis | GCViewer |
| jhat | Heap browser | `jhat heap.hprof` |

---

## **15. Common OOM Scenarios Quick Reference**

| Symptom | Likely Cause | First Action |
|---------|--------------|--------------|
| Heap OOM during peak hours | Insufficient heap | Increase -Xmx |
| Heap OOM with steady memory growth | Memory leak | Analyze heap dump |
| Metaspace OOM after redeploy | Classloader leak | Check for unclosed resources |
| Unable to create native thread | Thread leak | Check thread count, find leak |
| GC overhead limit exceeded | Heap too small or leak | Increase heap or fix leak |
| OOM with "request size bytes" | Native memory issue | Check NMT, increase container memory |
| OOM with high old gen usage | Objects not being GC'd | Find GC roots preventing collection |

---

**Created for Senior Java Developers and SREs**  
*Last Updated: November 18, 2025*

**Remember:** OOM issues are symptoms, not causes. Always dig deeper to find the root cause!

**Pro Tips:**
1. Always enable `-XX:+HeapDumpOnOutOfMemoryError` in production
2. Keep heap dumps from production incidents for analysis
3. Test with realistic data volumes in staging
4. Monitor memory trends, not just absolute values
5. Fix memory leaks, don't just increase heap size
6. Profile in production-like environments
7. Review code regularly for common patterns
8. Educate team on memory management
9. Have a runbook for OOM incidents
10. Learn from each incident and document
