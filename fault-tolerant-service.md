# Building a Fault-Tolerant Service Supervisor in Rust with Leader Election

In distributed systems, ensuring a service remains operational is a common requirement. A simple supervisor process can restart a failed service, but this introduces a new problem: the supervisor itself becomes a Single Point of Failure (SPOF). If the supervisor crashes, the service it monitors will not be restarted.

To build a truly fault-Tolerant system, the supervisor *itself* must be distributed and fault-tolerant. This article outlines an architecture for a robust supervisor system in Rust, using a distributed **consensus store (like `etcd`)** to manage leader election and prevent "split-brain" scenarios.

## Architecture Overview

The architecture consists of two primary components:

1. **A Consensus Store (e.g., `etcd`, `Consul`):** This is a separate, highly-available, distributed key-value store. It acts as the single source of truth and the arbiter for leadership.

2. **Supervisor Peers (Rust Application):** Multiple identical instances of our Rust supervisor application are run on different machines. They do not communicate directly but instead coordinate their actions through the consensus store.

At any given time, only one supervisor peer will be the **Leader**. All other instances are **Followers**. The Leader is solely responsible for running and monitoring the service. The Followers lie dormant, continuously attempting to become the leader in case the current one fails.

## 1. The Main Loop: Leader Election and Follower State

The core of our supervisor is a single, continuous loop (`supervisor_loop`). When an instance starts, it enters this loop and immediately tries to become the leader. This loop handles both leader election and the follower "waiting" state.

### How it Works:

1. **Grant a Lease:** Each supervisor instance first asks the consensus store for a new, short-term lease (e.g., 5 seconds). This lease has a unique ID.

2. **Atomic Transaction:** The instance attempts an atomic transaction: "IF the key `/supervisor/leader_lock` does not exist (its version is 0), THEN create it with my lease ID."

3. **The Winner (Leader):** The `etcd` cluster guarantees that only one instance will succeed in this transaction. This instance is now the **Leader**.

4. **The Losers:** All other instances fail the transaction (because the key now exists). They become **Followers**.

### Rust Implementation (Conceptual)

Here is the core logic for attempting to gain leadership using the `etcd-client` crate.

```rust
use etcd_client::{Client, Compare, CompareOp, Put, Txn};

/// The key in etcd to use for the leader lock.
const LEADER_LOCK_KEY: &str = "/supervisor/leader_lock";
const LEADER_VALUE: &str = "leader-info"; // Can store hostname, PID, etc.
const LEASE_TTL: i64 = 5; // 5-second lease

/// Attempts to acquire the leader lock.
/// Returns Ok(lease_id) on success, or Err() on failure.
async fn try_become_leader(
    client: &mut Client,
) -> Result<i64, Box<dyn std::error::Error>> {
    // 1. Grant a new lease
    let lease_resp = client.lease_grant(LEASE_TTL, None).await?;
    let lease_id = lease_resp.id();

    // 2. Build the atomic transaction
    let txn = Txn::new()
        .when(vec![
            // Check if key does not exist (version == 0)
            Compare::version(LEADER_LOCK_KEY, CompareOp::Equal, 0),
        ])
        .then(vec![
            // If it doesn't exist, create it with our lease
            Put::new(LEADER_LOCK_KEY, LEADER_VALUE, Some(lease_id)),
        ])
        .otherwise(vec![]); // Do nothing if lock is already taken

    // 3. Execute the transaction
    let txn_resp = client.kv_client().txn(txn).await?;

    // 4. Check if we won
    if txn_resp.succeeded {
        println!("(PID: {}) Acquired leadership (Lease: {})", std::process::id(), lease_id);
        Ok(lease_id)
    } else {
        // We failed to get the lock, revoke the lease we granted
        client.lease_revoke(lease_id).await?;
        Err("Failed to acquire lock (already held by another)".into())
    }
}
```

## 2. The Leader's Duty: Service Supervision & Health Checking

Once an instance becomes the Leader, it has two critical responsibilities:

1.  **Maintain Leadership:** It must continuously send "keep-alive" heartbeats to `etcd` to renew its lease. If it crashes, the heartbeats stop, the lease expires, and the lock is released for followers to claim.
2.  **Monitor the Service:** This is its primary function. This monitoring should be done at two levels.

### Level 1: Process Liveness

This is a passive check to see if the service process is still running. The supervisor spawns the service as a child process and waits for it to exit. If it exits (due to a crash or normal termination), the supervisor restarts it.

### Level 2: Service Health (Active Probing)

A process can be "running" but "unhealthy" (e.g., deadlocked, frozen, or unable to connect to its database). A robust supervisor must actively probe the service's health.

This is typically done by requiring the service to expose a health-check mechanism:

* **HTTP Endpoint:** The service exposes a `/health` endpoint. The supervisor polls this endpoint every few seconds. A `200 OK` response means healthy; any other response or a timeout indicates a failure.
* **Heartbeat File:** The service writes the current timestamp to a file (e.g., `/tmp/service.heartbeat`) every 5 seconds. The supervisor reads this file; if the timestamp is too old, the service is considered frozen.

If the service fails its health check, the Leader must proactively kill the unhealthy process and restart it.

### Rust Implementation (Conceptual)

This example demonstrates how a leader can manage both process liveness and active health checking concurrently using `tokio::select!`.

```rust
use tokio::process::Command;
use tokio::time::{self, Duration};
use futures::StreamExt; // Required for stream.next()

// A placeholder for our active health check logic
async fn perform_health_check() -> Result<(), &'static str> {
    // In a real app, this would be an HTTP call, a TCP connect,
    // or checking a heartbeat file.
    //
    // let resp = reqwest::get("http://localhost:8080/health").await;
    // if resp.is_ok() && resp.unwrap().status().is_success() {
    //     Ok(())
    // } else {
    //     Err("Health check failed")
    // }
    
    // Simulate a successful check for this example
    println!("(LEADER) Health check OK.");
    Ok(())
}

/// The main loop for the Leader.
/// Manages the service process and its health.
async fn run_as_leader(mut client: Client, lease_id: i64) {
    // 1. Spawn a background task to keep the lease alive
    let (mut keeper, mut stream) = match client.lease_keep_alive(lease_id).await {
        Ok((k, s)) => (k, s),
        Err(e) => {
            eprintln!("Failed to start lease keep-alive: {}. Relinquishing.", e);
            return;
        }
    };
    
    tokio::spawn(async move {
        while let Some(Ok(_)) = stream.next().await {
            // Lease renewed
        }
        eprintln!("Lease keep-alive stream ended. Leadership lost.");
    });

    // 2. Start the service supervision loop
    loop {
        println!("(LEADER) Starting supervised service...");
        let mut child = Command::new("path/to/your/service")
            .spawn()
            .expect("Failed to start service");

        let mut health_check_timer = time::interval(Duration::from_secs(10));
        let mut consecutive_failures = 0;

        // Concurrently wait for the process to exit OR for health checks to fail
        let exit_reason = tokio::select! {
            // Level 1 Check: Process Liveness
            exit_status = child.wait() => {
                format!("Service process exited with status: {:?}", exit_status)
            },

            // Level 2 Check: Service Health
            _ = async {
                loop {
                    health_check_timer.tick().await;
                    match perform_health_check().await {
                        Ok(_) => consecutive_failures = 0,
                        Err(e) => {
                            eprintln!("(LEADER) Health check failed: {}", e);
                            consecutive_failures += 1;
                            if consecutive_failures >= 3 {
                                break; // Stop this loop, triggering the select!
                            }
                        }
                    }
                }
            } => {
                // This block is reached only if the health check loop breaks
                format!("Service failed {} consecutive health checks.", consecutive_failures)
            }
        };

        // If we are here, the service needs to be restarted.
        eprintln!("(LEADER) Service restart triggered. Reason: {}", exit_reason);

        // Proactively kill the (potentially) zombied process
        if let Err(e) = child.kill().await {
            eprintln!("(LEADER) Failed to kill child process: {}", e);
        }
        
        // Brief delay before restarting
        time::sleep(Duration::from_secs(2)).await;
    }
}
```

## 3. The Follower's Duty

The follower's job is simple: do nothing except try to become the leader.

Followers run in a loop, repeatedly attempting the try_become_leader function. They will fail as long as the leader is healthy and renewing its lease. The moment the leader fails and its lease expires, the lock becomes available, and one of the waiting followers will successfully acquire it, promoting itself to the new leader.

```rust
use etcd_client::Client;
use tokio::time::{self, Duration};
// Assume try_become_leader and run_as_leader are defined elsewhere
// and rand is available
use rand;

/// The main loop for a Follower.
async fn run_as_follower(mut client: Client) {
    loop {
        println!("(FOLLOWER) Trying to acquire leadership...");
        match try_become_leader(&mut client).await {
            Ok(lease_id) => {
                // We won the election!
                println!("(FOLLOWER) Promoted to LEADER.");
                run_as_leader(client.clone(), lease_id).await;
                // `run_as_leader` only returns if leadership is lost.
                println!("(LEADER) Relinquished leadership. Returning to follower state.");
            }
            Err(_) => {
                // Failed to get lock. Wait and retry.
                // Use jitter to prevent a thundering herd.
                let delay = Duration::from_secs(2) + Duration::from_millis(rand::random::<u64>() % 1000);
                time::sleep(delay).await;
            }
        }
    }
}
```

## Conclusion

This leader-election architecture provides a robust, fault-tolerant, and consistent method for service supervision. By leveraging a distributed consensus store like etcd, we offload the most complex part of distributed computing (achieving consensus) and build a supervisor system that is resilient to individual node failures. This pattern ensures that the supervised service remains highly available, as a new supervisor is always ready to take over should the leader fail.