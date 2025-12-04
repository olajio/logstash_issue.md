Your understanding is partially correct, but there are some nuances worth considering.

## Logstash Behavior During Network Issues

When Logstash can't send events downstream, the behavior depends on your configuration:

**If persistent queues are enabled**, events are written to disk and will be replayed when connectivity resumes. This is more disk I/O intensive than CPU intensive during the buffering phase.

**If using in-memory queues** (the default), events accumulate in memory until the queue is full, at which point backpressure kicks in and Logstash slows down ingestion from inputs.

The CPU spike you're seeing is more likely caused by one of these scenarios:

1. **Aggressive retry behavior** - Logstash repeatedly attempting to reconnect to Elasticsearch or downstream outputs, especially if you have multiple pipelines or outputs configured

2. **Backlog processing after recovery** - When connectivity resumes, Logstash processing the queued events *and* new incoming events simultaneously. The CPU hit comes from filter processing (especially heavy ones like grok, dissect, or ruby filters), not just the sending

3. **Garbage collection pressure** - The JVM struggling with memory management when queues fill up with accumulated events

4. **Reconnection storms** - Multiple outputs or pipelines all trying to re-establish connections simultaneously

## Questions to Ask the Team

**Identify the process:**
- Is the spiking process actually Java/Logstash, or something else on the server?
- What does `top` or `htop` show as the specific process?

**Timeline and correlation:**
- When exactly did the network issue start and when did it resolve?
- Did the CPU spike happen during the outage, or after recovery?
- Is the spike sustained or intermittent?

**Configuration details:**
- Is persistent queue enabled (`queue.type: persisted`)?
- How many pipelines are running?
- What filters are in use (grok patterns can be expensive)?
- What are the batch size and worker settings?

**Current state:**
- What do the Logstash logs show? (`/var/log/logstash/logstash-plain.log`)
- What's the current queue depth? (Check the node stats API: `curl localhost:9600/_node/stats/pipelines?pretty`)
- Is downstream connectivity restored and verified?
- What's the memory usage alongside CPU?

**Infrastructure context:**
- How many CPU cores does this server have?
- Was there any recent configuration change before this incident?

Getting the Logstash logs and the pipeline stats would be the fastest path to root cause here.
