---
title: "🚀 Building a High-Throughput Async Consumer System with Redis Streams and SQLModel"
subtitle: "Redis + asyncio + SQLModel"
date: "2025-04-22"
---

>TL;DR — We’ll wire up Redis Streams to buffer messages, a consumer-group of asyncio workers to process them in parallel, and SQLModel + PostgreSQL to persist everything safely.
The full repo lives in [supermarket-py](https://github.com/quanturtle/supermarket-py); below are the key excerpts you can drop straight into your own project.

In this tutorial, we'll explore how to build a scalable and asynchronous worker system using:

* 🧰 Redis Streams as the message queue,

* 🐍 Python asyncio for concurrency,

* 🔁 SQLModel + PostgreSQL for persisting data,

* 🔗 Multiple workers running in parallel for efficient processing.

This setup is ideal for systems where hundreds of messages might flood in from producers (e.g., web scrapers, crawlers, sensors, APIs), and you want to persist them in a PostgreSQL-backed data store reliably and fast.

The code shown here is part of a larger project: [supermarket-py](https://github.com/quanturtle/supermarket-py). The worker module and code can be found here.

## 💡 Architecture Overview
Here's a bird's eye view of the system:

```
[Producers] → [Redis] → [Consumer Group] → [Workers] → [Postgres DB]
```
A consumer group allows multiple workers to consume from the same stream without duplicating work. Redis internally keeps track of which worker has read which messages, so each message is delivered only once to a single consumer.

Similar options include:

* Redis PubSub
* Kafka
* RabbitMQ

Each system has its own advantages/disadvantages/use cases, consider them carefully before choosing them for you next project.

## 🧰 Redis
Redis is an open-source, in-memory key-value data structure server used primarily as a database, cache, or message broker. It stores data in RAM, offering low-latency reads and writes, making it ideal for applications needing fast data access and manipulation.

Commands we will be using to build our application:

### XGROUP CREATE (docs)
```sh
XGROUP CREATE key group <id | $> [MKSTREAM]
```
```python
redis_conn.xgroup_create(STREAM_NAME, GROUP_NAME, id='0', mkstream=True)
```

* Creates a consumer group named GROUP_NAME for the stream STREAM_NAME.
* If the stream doesn’t exist yet, mkstream=True creates it automatically.
* id='0' means consumers will read all messages from the beginning.


### XREADGROUP (docs)
```sh
XREADGROUP GROUP group consumer [COUNT count] [BLOCK milliseconds]
  [NOACK] STREAMS key [key ...] id [id ...]
```
```python
redis_conn.xreadgroup(groupname=GROUP_NAME, 
                      consumername=consumer_name, 
                      streams={
                        STREAM_NAME: '>'
                      }, 
                      count=BATCH_SIZE, 
                      block=BLOCK_MS, )
```

* Reads up to `BATCH_SIZE` messages assigned to this worker.
* `'>'` means: “Give me messages never delivered to any other consumer.”
* `block=BLOCK_MS` waits (long-polling) for new messages instead of returning immediately.


### XADD (docs)
```sh
XADD key [NOMKSTREAM] [<MAXLEN | MINID> [= | ~] threshold
    [LIMIT count]] <* | id> field value [field value ...]
```
```python
redis_conn.xadd(STREAM_NAME, {"data": json.dumps(data)})
```

* Adds a single entry to a stream


### PIPELINE (docs)
```python
pipeline = redis_conn.pipeline()

for row in table:
    pipeline.xadd(STREAM_NAME, {"data": json.dumps(row)})

pipe.execute()
```

* Batch Redis commands
* Avoid network overhead of sending individual messages


### XACK (docs)
```sh
XACK key group id [id ...]
```
```python
redis_conn.xack(STREAM_NAME, GROUP_NAME, *message_ids)
```

* Confirms to Redis that the consumer has successfully processed the message.
* This removes it from the "pending" queue of unacknowledged messages.
* Without this step, Redis will think the message is still being processed and may redeliver it later (e.g., after a timeout or crash recovery).

We only acknowledge messages after they are inserted into the database successfully. This ensures at-least-once delivery.


### 🧰 redis-py
Create a connection to a running Redis server:

```python
import os, json
import redis.asyncio as redis

REDIS_HOST = os.getenv("REDIS_HOST", "localhost")
REDIS_PORT = int(os.getenv("REDIS_PORT", 6379))

redis_conn: redis.Redis = redis.Redis(
    host=REDIS_HOST,
    port=REDIS_PORT,
    decode_responses=True
)
```
The best way of getting a Redis server up and running is with docker.

Sample Redis configuration for a docker compose:
```docker
  redis:
    image: redis:7.4-alpine
    restart: always
    container_name: my-redis
    ports:
      - "${REDIS_PORT:-6379}:6379"
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
```

### ⚙️ Worker Loop

Create groups and streams:
```python
GROUP_NAME = "inserters"
STREAMS = ["stream_name_1", "stream_name_2"]

async def setup_consumer_groups():
    for stream in STREAMS:
        try:
            await redis_conn.xgroup_create(stream, GROUP_NAME, id="0", mkstream=True)
            print(f"Created group '{GROUP_NAME}' on stream '{stream}'.")
        except redis.ResponseError as e:
            # BUSYGROUP == already exists → safe to ignore
            if "BUSYGROUP" in str(e):
                pass
            else:
                raise
```

Each worker is a long-running asyncio coroutine:
```python
# consumer.py
async def run_worker(consumer_name: str, shutdown_event: asyncio.Event):
    await setup_consumer_groups()

    try:
        while not shutdown_event.is_set():
            try:
                response = await redis_conn.xreadgroup(
                    GROUP_NAME,
                    consumer_name,
                    { "stream_name_1": ">" },
                    count=100,  # batch of 100
                    block=5000, # block reads for 5s
                )
                if response:
                    await process_messages(consumer_name, response)

            except Exception:
                print("Unhandled error in worker loop; continuing.")
                await asyncio.sleep(2)

    finally:
        print("Exiting worker.")
```

Create an event that tells workers to stop and shutdown:
```python
# main.py
shutdown_event = asyncio.Event()

def handle_shutdown_signal(sig, frame):
    if not shutdown_event.is_set():
        print(f'Received signal {sig}. Initiating graceful shutdown...')
        shutdown_event.set()

    else:
        print(f'Received signal {sig} again. Shutdown in progress.')
```

And coordinate through a main.py:
```python
# main.py
async def main():
    loop = asyncio.get_running_loop()

    for sig in (signal.SIGINT, signal.SIGTERM):
        loop.add_signal_handler(sig, handle_shutdown_signal, sig, None)

    hostname = socket.gethostname()

    for i in range(NUM_WORKERS):
        consumer_name = f'{WORKER_NAME_PREFIX}-{hostname}-{i+1}'

        task = asyncio.create_task(run_worker(consumer_name, shutdown_event), name=f'Worker-{consumer_name}')
        running_tasks.append(task)

    await shutdown_event.wait()

    timeout_seconds = 30.0

    try:
        done, pending = await asyncio.wait(running_tasks, timeout=timeout_seconds)

        if pending:
            for task in pending:
                task.cancel()

            await asyncio.gather(*pending, return_exceptions=True)

        else:
             print('All worker tasks completed gracefully.')

    except Exception as e:
        print('An error occurred while waiting for worker tasks.')

    try:
        if redis_conn:
            await redis_conn.aclose()
            print('Closed Redis connection.')
        
        else:
            print('Redis connection already closed or not initialized.')

    except Exception as e:
        print(f'Error closing Redis connection: {e}')

    try:
        if db_engine:
            await db_engine.dispose()
        
        else:
             print('Database engine not initialized.')
    
    except Exception as e:
        print(f'Error disposing database engine: {e}')

    print('Application shutdown complete.')


if __name__ == '__main__':
    asyncio.run(main())
```

To run multiple consumers in parallel and shut them down cleanly, we use:
```python
NUM_WORKERS = int(os.getenv('NUM_CONSUMER_WORKERS', '3')) 

for i in range(NUM_WORKERS): 
    task = asyncio.create_task(run_worker(consumer_name, shutdown_event))
```
When a shutdown signal (SIGINT, SIGTERM) is received:

* A shared shutdown_event is set.
* All workers see the event and exit their loops.
* We then await all workers for up to 30 seconds to finish up.

Finally, we clean up resources:
```python
await redis_conn.aclose() 
await db_engine.dispose()
```

### 📥 Processing and Database Writes

Create a `AsyncSession` generator that can be used by each worker to create an async session to insert the data into the database:
```python
# models/database.py

from sqlalchemy.ext.asyncio import create_async_engine, AsyncEngine, async_sessionmaker, AsyncSession

AsyncSessionLocal = async_sessionmaker(
    bind=engine,
    class_=AsyncSession,
    expire_on_commit=False,
)
```

Messages are stored as JSON in a `"data"` field. A valid entry is parsed and mapped into a SQLModel ORM instance:
```python
        try:
            async with AsyncSessionLocal() as session:
                for obj in incoming_stream.values():
                    session.add_all(obj)

                await session.commit()
```

Finally, we acknowledge that the message was inserted:
```python
await redis_conn.xack(STREAM_NAME, GROUP_NAME, *ids_to_ack)
```

Acknowledgment only happens after a successful DB insert, ensuring at-least-once delivery.

Don’t forget to also apply `XTRIM` to manage how long our stream can get and don’t let it grow indefinitely.

Full process_messages function:
```python
STREAM_MODEL_MAP: Dict[str, Type] = {
    STREAM_NAME_1: DatabaseTable1,
    STREAM_NAME_2: DatabaseTable2,
}
    
async def process_messages(consumer_name: str, messages: List) -> int:
    if not messages:
        return 0

    bucket: Dict[Type, List] = {DatabaseTable1: [],}
    ack_success: List[str] = []
    ack_malformed: List[str] = []

    for stream, entries in messages:
        model_cls = STREAM_MODEL_MAP.get(stream)
        
        if model_cls is None:
            print(f"Unknown stream '{stream}' – skipping.")
            continue

        for entry_id, fields in entries:
            raw = fields.get("data")
        
            if raw is None:
                print(f"{entry_id}: missing 'data' field, not ACKing.")
                ack_malformed.append(entry_id)
                continue

            try:
                data = json.loads(raw)
                data['created_at'] = datetime.fromisoformat(data['created_at'])
                
                bucket[model_cls].append(model_cls(**data))
                ack_success.append(entry_id)
            
            except Exception as e:
                print(f"{entry_id}: parse/validation error: {e} – skipping.")
                ack_malformed.append(entry_id)

    total_inserted = sum(len(entries) for entries in bucket.values())

    if total_inserted:
        print(f"Inserting batch of {total_inserted} objects into DB...")

        try:
            async with AsyncSessionLocal() as session:
                for obj in bucket.values():
                    session.add_all(obj)

                await session.commit()

            print("DB commit successful.")

        except Exception:
            print("DB commit failed – nothing has been ACKed.")
            ack_success.clear()
            total_inserted = 0

    ids_to_ack = ack_malformed + ack_success
    
    if ids_to_ack:
        try:
            await redis_conn.xack(*([stream for stream in STREAM_MODEL_MAP.keys()][0:1]), GROUP_NAME, *ids_to_ack)
        
        except Exception:
            print(f"Failed to ACK {len(ids_to_ack)} messages.")

    try:
        for stream in STREAM_MODEL_MAP.keys():
            trimmed = await redis_conn.xtrim(stream, maxlen=MAX_STREAM_LENGTH, approximate=False)
            
            if trimmed:
                print(f"XTRIM {stream}: removed {trimmed} old entries.")
    

    return total_inserted
```

### That’s it! 🚀
Spin up a few producers, watch messages flow through Redis, and your Postgres table will keep up—without duplicate inserts or blocking I/O.

You can try my set up by cloning my repo and running in two separate terminals:
```sh
# spin up consumer
uv run worker/main.py
```
```sh
# send messages to redis
uv run worker/producer.py --mode stream --entity ProductURL
uv run worker/producer.py --mode bulk --entity Product
```

Run stream (uses single xadd commands):
![]()

![]()

Run bulk (uses pipeline.xadd):

![]()

![]()

Feedback & PRs welcome over at `github.com/quanturtle/supermarket‑py`