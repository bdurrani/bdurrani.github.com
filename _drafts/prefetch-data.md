---
layout: post
title: Prefetch data from disk
---

The task was simple. I have data on disk where the sizes varied from 1kb to a couple of gigs. Read the data points and return the points 
from a method that returns an IEnumerable<DataPoint>. The specific data file has a custom format, so I had to use a custom API to read
the data. There was no async I/O option available unfortunately.

Sounds simple enough. Here is my first version. 
I replaced the custom API with a simple file read operation. As with the custom API, I read pairs of data in 
blocks I can specify.

{% highlight csharp %}

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

{% endhighlight %}

This works ofcourse. 
I read the data in blocks, create the Datum as required, and return the data. 

The client can use foreach and lazily process the Datums as needed.

{% highlight csharp %}
foreach (var item in Data(CancellationToken.None))
{
    // process the datums
} 
{% endhighlight %} 

Can we do better?

Note: mention the [Patterns of parallel programming][1] as a reference.

{% highlight csharp %}
private void Prefetcher(CancellationToken cancellationToken) 
{ 
    try 
    { 
        using (var session = CustomFileReader.OpenFile(fileName)) 
        { 
            
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
{% endhighlight %}

[1]: https://www.microsoft.com/en-ca/download/details.aspx?id=19222
