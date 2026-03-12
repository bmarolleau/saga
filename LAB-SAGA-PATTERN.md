# Lab: Understanding the SAGA Pattern with Apache Camel
**Duration:** 25 minutes

## Overview
Learn how the SAGA pattern handles distributed transactions across microservices using Long Running Actions (LRA) for compensation in case of failures.

![alt text](assets/saga-unexptected.png)
---

## Step 1: Cluster Login and Project Setup

Connect to your OpenShift cluster and create your project:

```bash
# Login to cluster (adapt with your credentials)
oc login --token=YOUR-TOKEN --server=https://c113-e.eu-de.containers.cloud.ibm.com:30516

![alt text](assets/image-login.png)

# Create your project (use your team number)
oc new-project teamX
```

---

## Step 2: Build and Deploy the Application

Clone the repository and deploy the SAGA example:

```bash
git clone https://github.com/apache/camel-spring-boot-examples.git
cd camel-spring-boot-examples/saga
./install-ocp.sh
```

Wait for all pods to be ready (this may take 2-3 minutes).

---

## Step 3: Create Routes for External Access

Expose the services to make them accessible:

```bash
oc expose service camel-example-spring-boot-saga-app
oc expose service camel-example-spring-boot-saga-flight
oc expose service camel-example-spring-boot-saga-payment
oc expose service camel-example-spring-boot-saga-train
```

Verify routes are created:
```bash
oc get routes | grep camel-example-spring-boot-saga
```

---

## Step 4: Monitor Application Logs

Open **4 separate terminal windows** to monitor each service in real-time:

**Terminal 1 - Main App:**
```bash
oc logs -f $(oc get pods -l app.kubernetes.io/name=camel-example-spring-boot-saga-app -o name)
```

**Terminal 2 - Flight Service:**
```bash
oc logs -f $(oc get pods -l app.kubernetes.io/name=camel-example-spring-boot-saga-flight -o name)
```

**Terminal 3 - Train Service:**
```bash
oc logs -f $(oc get pods -l app.kubernetes.io/name=camel-example-spring-boot-saga-train -o name)
```

**Terminal 4 - Payment Service:**
```bash
oc logs -f $(oc get pods -l app.kubernetes.io/name=camel-example-spring-boot-saga-payment -o name)
```

![alt text](assets/term-image.png)
---

## Step 5: Test SAGA with Random Payment Failures

The application simulates random payment failures to demonstrate compensation.

**Execute the SAGA transaction:**
```bash
curl -X POST "http://$(oc get route camel-example-spring-boot-saga-app -o go-template --template='{{.spec.host}}')/api/saga?id=1"
```

**Run this command multiple times (5-10 times)** and observe the different outcomes in your log terminals.

### What to Observe:

Each transaction follows this sequence:
1. **Train booking** is attempted first
2. **Flight booking** is attempted second
3. **Payment** is processed last

The payment service has a **random failure rate** built-in. Watch for these scenarios:

---

## Step 6: Understanding SAGA Compensation Patterns

### Scenario A: Train Payment Fails ❌
**Sequence:**
- Train booking: ✅ Reserved
- Train payment: ❌ **FAILS**
- LRA Status: **CANCELLED**

**Compensation:**
- Train reservation is **compensated** (cancelled)
- Flight is never attempted

**Log Pattern:**
```
Train: Reserving train...
Train: Payment failed!
LRA: Compensation triggered
Train: Cancelling train reservation...
```

---

### Scenario B: Flight Payment Fails ❌
**Sequence:**
- Train booking: ✅ Reserved
- Train payment: ✅ Success
- Flight booking: ✅ Reserved
- Flight payment: ❌ **FAILS**
- LRA Status: **CANCELLED**

**Compensation:**
- Flight reservation is **compensated** (cancelled)
- Train reservation is **compensated** (cancelled) - **This is the SAGA pattern in action!**

**Log Pattern:**
```
Train: Reserving train... SUCCESS
Train: Payment confirmed
Flight: Reserving flight... SUCCESS
Flight: Payment failed!
LRA: Compensation triggered
Flight: Cancelling flight reservation...
Train: Cancelling train reservation...  ← Rollback!
```

---

### Scenario C: All Payments Succeed ✅
**Sequence:**
- Train booking: ✅ Reserved
- Train payment: ✅ Success
- Flight booking: ✅ Reserved
- Flight payment: ✅ Success
- LRA Status: **COMPLETED**

**No Compensation Needed:**
- All bookings are confirmed
- Transaction is complete

**Log Pattern:**
```
Train: Reserving train... SUCCESS
Train: Payment confirmed
Flight: Reserving flight... SUCCESS
Flight: Payment confirmed
LRA: Transaction completed successfully
```

---

## Key Takeaways

1. **SAGA Pattern**: Manages distributed transactions without requiring distributed locks
2. **LRA (Long Running Actions)**: Coordinates compensation across services
3. **Compensation Logic**: Each service implements both forward and backward operations
4. **Eventual Consistency**: System reaches a consistent state through compensation, not rollback

---

## Optional: Adjust Failure Rate

To change the payment failure probability, examine and modify the payment service source code:

**View the source code:**
[`saga-payment-service/src/main/java/org/apache/camel/example/saga/PaymentRoute.java`](saga-payment-service/src/main/java/org/apache/camel/example/saga/PaymentRoute.java)

Look for the random threshold logic that determines payment success/failure rate.

**To rebuild and redeploy after modifications:**

1. **Build the payment service:**
   ```bash
   cd saga-payment-service
   mvn clean package -DskipTests
   cd ..
   ```

2. **Deploy only the payment service:**
   ```bash
   mvn oc:build oc:deploy -pl saga-payment-service
   ```

3. **Verify the new pod is running:**
   ```bash
   oc get pods -l app.kubernetes.io/name=camel-example-spring-boot-saga-payment -w
   ```

---

---

## 🎯 Bonus Exercise: LRA Coordinator Detective (5 minutes)

**Challenge:** Become an LRA detective and track the lifecycle of your distributed transactions!

### Mission Briefing

The LRA Coordinator is the "brain" that orchestrates all compensations. Let's spy on it to understand how it manages transaction states.

### Step 1: Access the LRA Coordinator

```bash
# Get the LRA Coordinator route
oc get route lra-coordinator

# Or create it if it doesn't exist
oc expose service lra-coordinator
```

### Step 2: Monitor Active LRAs

Open the LRA Coordinator dashboard in your browser:
```bash
echo "http://$(oc get route lra-coordinator -o go-template --template='{{.spec.host}}')"
```

### Step 3: Transaction Analysis

**Run this script to generate multiple transactions:**
```bash
for i in {1..10}; do
  echo "Transaction $i..."
  curl -X POST "http://$(oc get route camel-example-spring-boot-saga-app -o go-template --template='{{.spec.host}}')/api/saga?id=$i" &
  sleep 0.5
done
wait
```

**Your Mission:** Analyze the LRA Coordinator dashboard and determine how many transactions are in each state:
- ✅ Completed
- ❌ Cancelled
- ⏳ Active (still running)

Calculate the overall success rate and document your findings.

### Bonus Points 🌟

**Check the LRA Coordinator logs:**
```bash
oc logs -f $(oc get pods -l app=lra-coordinator -o name)
```

**Questions to explore:**
- How long does the coordinator keep transaction history?
- What happens if you restart the LRA Coordinator pod during a transaction?
- Can you find the LRA transaction IDs in the service logs?

### Pro Tip 💡

Each LRA has a unique ID. Try to match an LRA ID from the coordinator with the corresponding logs in your service terminals!

```bash
# Example: Search for a specific LRA ID in logs (replace with actual LRA ID from coordinator)
oc logs $(oc get pods -l app.kubernetes.io/name=camel-example-spring-boot-saga-train -o name) | grep "12345678-1234-1234-1234-123456789abc"
```

---

### Answers to Exploration Questions

**Q: How long does the coordinator keep transaction history?**

A: The LRA Coordinator keeps transaction history in memory by default. The retention depends on the coordinator's configuration, but typically:
- Completed transactions: Retained for a configurable period (default: until coordinator restart)
- Failed/Cancelled transactions: Retained longer for debugging purposes
- Active transactions: Kept until completion or timeout

For production systems, you should configure persistent storage or integrate with a monitoring solution to retain historical data beyond pod restarts.

**Q: What happens if you restart the LRA Coordinator pod during a transaction?**

A: When the LRA Coordinator pod restarts during an active transaction:
1. **Active LRAs are lost** - The coordinator loses in-memory state
2. **Services may be left in inconsistent states** - Participants don't receive compensation signals
3. **Timeout mechanisms activate** - Services should implement timeout logic to handle coordinator failures
4. **Manual intervention may be required** - You may need to manually compensate or rollback affected services

This demonstrates why production LRA Coordinators should use persistent storage and high availability configurations.

**Q: Can you find the LRA transaction IDs in the service logs?**

A: Yes! LRA transaction IDs appear in service logs when:
- A saga is initiated (look for "LRA started" or similar messages)
- Compensation is triggered (look for "Compensating" messages)
- The saga completes or fails

Example search commands:
```bash
# Search in the main app
oc logs $(oc get pods -l app.kubernetes.io/name=camel-example-spring-boot-saga-app -o name) | grep -i "lra"

# Search in flight service
oc logs $(oc get pods -l app.kubernetes.io/name=camel-example-spring-boot-saga-flight -o name) | grep -i "lra"

# Search in payment service
oc logs $(oc get pods -l app.kubernetes.io/name=camel-example-spring-boot-saga-payment -o name) | grep -i "lra"

# Search in train service
oc logs $(oc get pods -l app.kubernetes.io/name=camel-example-spring-boot-saga-train -o name) | grep -i "lra"
```

The LRA ID is a UUID that uniquely identifies each distributed transaction across all participating services.

---

**Time's up! Share your findings with the class 🎉**


## Cleanup

```bash
oc delete project teamX
```

---

## Credits / References

- Based on the [Camel SAGA example](https://github.com/apache/camel-spring-boot-examples/tree/main/saga)

- Read the reference ! [The Saga Pattern in Apache Camel by Nicola Ferraro](https://www.nicolaferraro.me/2018/04/25/saga-pattern-in-apache-camel/)

**End of Lab**