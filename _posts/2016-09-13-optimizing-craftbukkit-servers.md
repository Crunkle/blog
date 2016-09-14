---
layout: post
title: Optimizing CraftBukkit for large servers
---

It is always more beneficial to optimize a server before paying to upgrade hardware. A common misconception is that more RAM in a server equals better performance. This is not the case whatsoever and is simply a marketing misconception among shared hosts. It is to be noted that some changes I am going to implement are focused towards larger networks rather than vanilla gameplay thus removing unnecessary gameplay features.

I created a server benchmarking tool which gradually ramps up processing using large amounts of entities and chunks. This tool eventually throws me a calculated score based on how well the server was able to keep up before experiencing noticeable lag. I wouldn’t call this the most accurate way to benchmark all aspects of the JVM but it should be enough to detect an improvement after our changes have been implemented.

```shell
[15:57:30] [Server thread/INFO]: Benchmark Score: 2017
[16:00:39] [Server thread/INFO]: Benchmark Score: 1501
[16:07:29] [Server thread/INFO]: Benchmark Score: 2004
[16:17:26] [Server thread/INFO]: Benchmark Score: 1770
```

The averaged benchmark score for this server is 1823. This score will vary based on hardware but, as we are using the same hardware to benchmark our end result, it shouldn't matter in our case. In order to find the bottlenecks, we need to attach a profiler to the server and see what is causing the performance hit. I will be using JProfiler to analyze the JVM as it has less overhead for CPU monitoring than other competitors.

![example](https://cdn.jared.im/static/jprofiler_2016-09-13_19-53-10.png)

We shouldn’t be trying to improve any of the provided classes as they are not a direct part of the game server. This does not mean that we can’t change the server’s implementation to use more suitable data structures. The first eye catching result of this CPU snapshot is a large amount of time spent fetching chunks, blocks and entities. We need to take a look at how chunks are loaded and stored in memory to find out why they are taking a long time to retrieve.

```java
public class ChunkProviderServer implements IChunkProvider {

    private static final Logger a = LogManager.getLogger();
    public final Set<Long> unloadQueue = Sets.newHashSet();
    public final ChunkGenerator chunkGenerator;
    private final IChunkLoader chunkLoader;
    public final Long2ObjectMap<Chunk> chunks = new Long2ObjectOpenHashMap(8192);
    public final WorldServer world;

...
```

Instead of focusing on world saving, which we’ll talk about later, I want to point your attention to the map type used to store chunks. There is no point in dynamically allocating memory at runtime if our world is completely static. As long as our chunk coordinates aren’t scattered around the world, we can get every single chunk without any overhead. We need to implement a new structure which is backed by a primitive array and this structure needs to be compatible with the original type to prevent dozens of compilation errors.

```java
public class ChunkProviderServer implements IChunkProvider {

    private static final Logger a = LogManager.getLogger();
    public final Set<ChunkCoordIntPair> unloadQueue = new HashSet<ChunkCoordIntPair>();
    public final ChunkGenerator chunkGenerator;
    private final IChunkLoader chunkLoader;
    public final IndexedChunkMap chunks = new IndexedChunkMap(2048);
    public final WorldServer world;

...
```

As we are only using 22 bits of data for each chunk coordinate, chunks will only be able to span from -1024 to 1024 allowing us to move just over 16,000 blocks in any direction from the origin. This should be more than enough to hold a medium to large world. We could also create the object by looking at how many chunks the world’s region files use (which is slightly more complicated). In order to fully implement this array, we need to rewrite the access methods to use our improved array. We also need to remember to account for negative coordinates to prevent them from going out of the array’s bounds. The indexing we have implemented is a simple bitshift to store both parts of the chunk's location in a single integer.

```java
public class IndexedChunkMap {

    private final Chunk[] values;
    private final int size;
    private final int bits;

    public IndexedChunkMap(int size) {
        this.values = new Chunk[IntMath.pow(size, 2)];
        this.size = size;
        this.bits = IntMath.log2(size, RoundingMode.UP);

        if (this.bits > 16 || !IntMath.isPowerOfTwo(size)) {
            throw new UnsupportedOperationException("Size must be compatible with indexing");
        }
    }

    public void put(int x, int z, Chunk chunk) {
        x += this.size / 2;
        z += this.size / 2;

        this.values[x | z << this.bits] = chunk;
    }

    public Chunk get(int x, int z) {
        x += this.size / 2;
        z += this.size / 2;

        return this.values[x | z << this.bits];
    }

    public Collection<Chunk> values() {
        return Arrays.asList(this.values);
    }

    public int size() {
        return this.size;
    }
}
```

Now that we have our custom in-memory chunk storage done, we can look at world saving. Unless you want to rewrite a custom storage file format (which has been done before), there isn’t much need to be saving a world which is static. If no blocks are being changed and no lighting needs updating, what’s the point of saving identical chunks? On top of this, for smaller worlds, there isn’t always a need to unload the world. It would be much faster to just keep the world fully loaded at all times for immediate in-memory access, but be careful that you haven't got dozens of entities loaded alongside the chunks. You can dig deeper into this paradigm and say that you also don’t need block physics, lighting recalculations (in the case of static worlds), and a large majority of the vanilla features - but we won’t step too far for the sake of abstraction.

```java
public class WorldServer extends World implements IAsyncTaskHandler {

    public void save(boolean flag, @Nullable IProgressUpdate iprogressupdate) throws ExceptionWorldConflict {
        ;
    }

...

public class WorldNBTStorage implements IDataManager, IPlayerFileData {

    public void save(EntityHuman entityhuman) {
        ;
    }

...
```

You always can go further and disable the world NBT data entirely if you are certain that you have no need for this data. I have personally disabled a large number of things ranging from all file storage to whitelists. I want to take a quick look at entities before we wrap up and test our changes. We can see that there is a large amount of latency when entities move and collide due to block lookups and collision tests. This is a slightly more complicated issue to fix, but one solution is to cache the bounding boxes, use a similar lookup method as we did with chunks for blocks and reuse the bounding box objects instead of creating new ones on modification. I could be here for hours talking about different tweaks you can make, but after spending a few hours making changes, the work definitely pays off.

```shell
[20:29:50] [Server thread/INFO]: Benchmark Score: 11288
[20:33:37] [Server thread/INFO]: Benchmark Score: 10807
[20:36:20] [Server thread/INFO]: Benchmark Score: 12040
[20:44:31] [Server thread/INFO]: Benchmark Score: 10977
```

Now we are at a new, and heavily improved, average score of 11278. To put this in to perspective, this is over a 6x increase in benchmark performance and I was now able to load around 15,000 active entities on my local machine before dropping ticks. Keep in mind that you can always optimize something further, but you don’t want to destroy the abstraction by making everything completely static. Most of the changes were rather complicated and boring so I won't go in to depth any further and leave you with this: despite being a notorious RAM-eating monster, if you're running proprietary gamemodes, you really don't need to be spending valuable CPU time on unnecessary gameplay features. Fix the code before you fix the hardware.
