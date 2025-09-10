# Sawtooth Aggregator: Multi-Granularity Window Aggregation

## Overview

The Sawtooth Aggregator is Chronon's innovative solution for efficiently computing sliding window aggregations by using different granularities of pre-aggregated "hops" (tiles) based on data age.

## The Sawtooth Pattern

The name "sawtooth" comes from the visual pattern created when the window start boundary snaps to different hop boundaries as time progresses:

```
Window Start Precision Loss Over Time:

Precision
Gap (hours)
    24 |────────────────────────────╮
       |                            │  <- Window jumps to next daily hop
    18 |                  ╱╱╱╱╱╱╱╱╱╱│
       |                ╱╱          │
    12 |              ╱╱            │
       |            ╱╱              │  <- Precision gap grows
     6 |          ╱╱                │
       |        ╱╱                  │
     0 |──────╱╱────────────────────┴──
       └─────────────────────────────────
       Query   +6hrs  +12hrs  +18hrs  +24hrs
       Time                           (next day)
```

## Core Architecture

### Multi-Granularity Hops

The Sawtooth Aggregator uses different hop sizes for different parts of the window:

```scala
// From Resolution.scala
object FiveMinuteResolution extends Resolution {
  def calculateTailHop(window: Window): Long = {
    window.millis match {
      case x if x >= 12.DAYS  => 1.DAY    // Daily hops for large windows
      case x if x >= 12.HOURS => 1.HOUR   // Hourly hops for medium windows
      case _                  => 5.MINUTES // 5-min hops for small windows
    }
  }
  
  val hopSizes: Array[Long] = Array(
    Day.millis,      // 86,400,000 ms
    Hour.millis,     // 3,600,000 ms
    FiveMinutes      // 300,000 ms
  )
}
```

### Window Composition Strategy

For a 7-day window query, the aggregator combines:

1. **Daily Hops** (Days 1-5): Coarse-grained, efficient storage
2. **Hourly Hops** (Day 6): Medium granularity 
3. **5-Minute Hops** (Day 7 early): Fine granularity
4. **Real-time Stream** (Last few minutes): Latest data

## Implementation Details

### The SawtoothAggregator Class

```scala
class SawtoothAggregator(
  aggregations: Seq[Aggregation],
  inputSchema: StructType,
  resolution: Resolution
) {
  
  def computeWindows(
    hopsArrays: Array[Array[Any]], 
    queryTimes: Array[Long]
  ): Array[Array[Any]] = {
    // For each query time
    queryTimes.map { queryTime =>
      // Select appropriate hops based on window size
      val windowIRs = selectHopsForWindow(hopsArrays, queryTime)
      
      // Merge intermediate results
      aggregator.bulkMerge(windowIRs)
    }
  }
  
  private def selectHopsForWindow(
    hops: Array[Array[Any]], 
    queryTime: Long
  ): Seq[Any] = {
    // Complex logic to select the right mix of
    // daily, hourly, and 5-minute hops
  }
}
```

### Tail Hops for Precision

Tail hops provide fine-grained aggregations at the window boundaries to ensure exact window alignment:

```scala
case class BatchIr(
  collapsed: Array[Any],        // Bulk aggregated data
  tailHops: HopsAggregator.IrMapType  // Fine-grained boundary hops
)
```

## Performance Benefits

### Efficiency Comparison

```
Traditional Approach (Process Everything):
- 7-day window = 604,800 seconds of data
- At 10 events/sec = 6,048,000 events to process
- Query latency: ~5 seconds

Sawtooth Approach (Multi-granularity):
- Daily hops: 5 reads
- Hourly hops: 24 reads
- 5-min hops: 180 reads
- Total: ~209 reads (vs 6M events!)
- Query latency: ~5 milliseconds (1000x faster!)
```

### Storage Optimization

| Hop Granularity | Tiles per Day | Storage Overhead | Best For |
|-----------------|---------------|------------------|----------|
| 5 minutes | 288 | High | Recent data, short windows |
| 1 hour | 24 | Medium | Medium-age data |
| 1 day | 1 | Low | Old data, long windows |

## Trade-offs

The sawtooth pattern represents a careful balance:

- ✅ **Massive performance gains** (100-1000x faster queries)
- ✅ **Real-time precision** at the window head (recent data)
- ✅ **Storage efficiency** through coarse-grained historical hops
- ⚠️ **Window start imprecision** (up to hop size, mitigated by tail hops)

## Configuration

Users don't directly configure the Sawtooth Aggregator. Instead, it automatically activates based on:

1. **Window sizes** in GroupBy aggregations
2. **Resolution settings** (FiveMinuteResolution vs DailyResolution)
3. **Accuracy mode** (TEMPORAL vs SNAPSHOT)

Example triggering configuration:

```python
GroupBy(
    aggregations=[
        Aggregation(
            operation=Operation.SUM,
            windows=[
                Window(7, TimeUnit.DAYS),   # Triggers sawtooth
                Window(30, TimeUnit.DAYS)   # Also uses sawtooth
            ]
        )
    ]
)
```

## Integration with Tiling

The Sawtooth Aggregator works seamlessly with Chronon's tiling architecture:

1. **Batch Process**: Creates daily/hourly hops during batch runs
2. **Flink Streaming**: Generates 5-minute tiles in real-time
3. **Query Time**: Sawtooth Aggregator combines all hop types

## Conclusion

The Sawtooth Aggregator enables Chronon to serve long-window aggregations with:
- Sub-10ms latency
- Minimal storage overhead
- Exact window semantics (with tail hops)
- Seamless batch-streaming integration

This innovative approach makes it practical to compute features like "user spending in last 30 days" in real-time at scale.