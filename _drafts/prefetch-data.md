---
layout: post
title: Prefetch data from disk
---

The task was simple. I have data on disk where the sizes varied from 1kb to a couple of gigs. Read the data points and return the points 
from a method that returns an IEnumerable<DataPoint>. The specific data file has a custom format, so I had to use a custom API to read
the data. 

Sounds simple enough. Here is my first version. I've simplified the custom data API and added comments to explain
the parameters

{% highlight csharp %}
public IEnumerable<Datum> Data(CancellationToken cancellationToken) 
{ 
	// lets read the data in chunks of 8kb points
	// chosen based on L1 cache size
	const long BlockSize = 1<<13;

	using (var session = CustomFileReader.OpenFile(fileName)) 
	{ 
		string groupName = "data group";
		string channelOne= ..., channelTwo = ...;	// some channel names
		// get the total number of points for the group
		var totalCount = (long) CustomFile.GetGroupCount(session, groupName); 

		long currentCount = 0L;
		while (currentCount < totalCount)
		{
			var channelOneData = CustomFile.ReadData(session, groupName, 
					channelOne,	// channel name 
					currentCount,	// offset to start reading from 
					BufferSize)	// number of data points to return from the channel
				as int[]; 
			foreach (var item in datum) 
			{ 
				yield return item; 
			} 
		}
	} 
}

{% endhighlight %}

The data reader API did not offer a version that export async I/O operations. 
Note: mention the 'Patterns of parallel programming' as a reference.
Link: https://www.microsoft.com/en-ca/download/details.aspx?id=19222

{% highlight csharp %}
private void Prefetcher(CancellationToken cancellationToken) 
{ 
    try 
    { 
	using (var session = CustomFileReader.OpenFile(fileName)) 
	{ 
	    var groupName = Id.ToString(CultureInfo.InvariantCulture); 
	    var totalCount = (long) TdmsDatasetHelper.GetGroupCount(_tdmsProvider, session, groupName); 
	    long currentCount = 0; 
	    while (currentCount < totalCount) 
	    { 
		if (cancellationToken.IsCancellationRequested) 
		{ 
		    return; 
		} 
		bool eof; 
		var valueData = 
		    _tdmsProvider.ReadData(session, groupName, TdmsDatasetHelper.ValueChannelName, 
			PFTypes.DoubleArray1D, 
			currentCount, BufferSize, out eof) as 
			double[]; 
		var overflowData = 
		    _tdmsProvider.ReadData(session, groupName, TdmsDatasetHelper.OverflowChannelName, 
			PFTypes.Int32Array1D, currentCount, BufferSize, out eof) as int[]; 
		if (valueData == null || overflowData == null) 
		{ 
		    return; 
		} 
		var datums = valueData.Zip(overflowData).Select(v => Datum.Create(v.Key, v.Value)).ToArray(); 
		// check for cancellation before doing anything with the collection. 
		if (cancellationToken.IsCancellationRequested) 
		{ 
		    return; 
		} 
		// this will throw on cancellation 
		_prefetchDatumCollection.Add(datums, cancellationToken); 
		currentCount += valueData.Length; 
	    } 
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
