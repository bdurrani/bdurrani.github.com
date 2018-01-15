---
date: 2015-07-01T00:00:00Z
title: Prefetch data from disk 
tags: ["c#"]
---

The task was simple. I have data on disk where the file sizes varied from 1kb to a couple of gigs. I needed to process the data 
and return the points from a method that returns an IEnumerable<DataPoint>. The specific data file has a custom format, so I had to use a custom API to read
the data. There was no async I/O option available unfortunately.

Sounds simple enough. Here is my first version. 
I replaced the custom API with a simple file read operation. As with the custom API, I read pairs of data in 
block sizes I can specify.

{{< highlight csharp >}} 
///<summary>
/// Contains a data point
///</summary>
public class Datum
{
    public double A { get; private set; }
    public double B { get; private set; }

    public Datum(double a, double b)
    {
        A = a;
        B = b;
    }
}

public IEnumerable<Datum> Data() 
{ 
    // lets read the data in chunks of 8kb points
    // chosen based on L1 cache size
    const long BlockSize = 1<<13;

    using (BinaryReader reader = new BinaryReader(
                        File.Open(fileName, FileMode.Open)))
    {
        var datumSize = 2 * sizeof(double);
        // number of Datums to read in a block
        var blockSize = 1 << 13;

        // read a block of data and return
        byte[] data = reader.ReadBytes(blockSize);
        while (data.Length > 0)
        {
            for (long i = 0; i < data.Length; i += datumSize)
            {
                double a = BitConverter.ToDouble(data, 0);
                double b = BitConverter.ToDouble(data, sizeof(double));
                yield return new Datum(a, b); 
            }
            data = reader.ReadBytes(blockSize);
        }
    }
} 
{{< / highlight >}}

This works ofcourse. 
I read the data in blocks, create the Datum as required, and return the data. 

The client can use `foreach` and lazily process the Datums as needed.

{{< highlight csharp >}}
foreach (var item in Data(CancellationToken.None))
{
    // process the datums...
} 
{{< / highlight >}} 

Can we do better?
What if while the client is enumerating over the Datums and processing that, we read the next block
of data and have it ready to go. We don't need to wait for the client to finish their processing before
we can read the next block.

Let's see how that would work.
First we would need to kick off a background thread dedicated to grabbing chunks of data from disk.
We can prefetch a certain amount of data and have it ready to go for when the client requests it. 

The prefetcher could look something like the following:

{{< highlight csharp >}}

private void Prefetcher(CancellationToken cancellationToken) 
{
    using (BinaryReader reader = new BinaryReader(
                File.Open(fileName, FileMode.Open))) 
    {
        try
        {
            var datumSize = 2 * sizeof(double);
            // number of Datums to read in a block
            var blockSize = 1 << 13;

            // read a block of data and return
            byte[] data = reader.ReadBytes(blockSize);
            while (data.Length > 0)
            {
                var datums = new Datum[data.Length/2]; 
                for (long i = 0; i < data.Length; i += datumSize)
                {
                    double a = BitConverter.ToDouble(data, 0);
                    double b = BitConverter.ToDouble(data, sizeof(double));
                    datums[i] = new Datum(a, b); 
                }
                // this will throw on cancellation 
                // Add will block once the capacity of the blocking collection is reached
                _prefetchDatumCollection.Add(datums, cancellationToken); 
                data = reader.ReadBytes(blockSize);
            }
        }
        catch (OperationCanceledException) 
        { 
            // don't let it bubble up. I don't want Task.Wait() to throw.  
            // Just end the task normally to let the prefetch collection clean up happen correctly 
        } 
        finally 
        { 
            _prefetchDatumCollection.CompleteAdding();  
        } 
    }
}

public IEnumerable<Datum> Data(CancellationToken cancellationToken) 
{ 
    _prefetchDatumCollection = new BlockingCollection<Datum[]>(BlockCount); 
    // using cancellationToken will prevent the task from starting if the cancel
    // happens before we get here
    var readTask = Task.Factory.StartNew(() => Prefetcher(cancellationToken) , 
            cancellationToken, TaskCreationOptions.LongRunning, TaskScheduler.Current); 

    // not using cancellation token here to prevent GetConsumingEnumerable() from throwing 
    // on cancel. Let it exit normally. It will stop normally once its read everything  
    // that was already there in the collection. The collection will mark itself as  
    // complete once the task returns.
    foreach (var datum in _prefetchDatumCollection.GetConsumingEnumerable()) 
    { 
        foreach (var item in datum) 
        { 
            yield return item; 
        } 
    } 
    try 
    { 
        // no cancellation token here. Let the task end normally. If anything else bad happens, 
        // the collection will still get cleaned up. 
        readTask.Wait(); 
    } 
    finally 
    { 
        _prefetchDatumCollection.Dispose(); 
    } 
} 
{{< / highlight >}}

That's one way to do it. One optimization would be to avoid using a 'Task' when dealing with small files.
The overhead is not worth it in that scenario. 

A great reference for parallel programming is the [Patterns of parallel programming][1]. It's a great
starting point for familiarizing yourself with the various parallel program libraries available in the .NET Framework

[1]: https://www.microsoft.com/en-ca/download/details.aspx?id=19222
