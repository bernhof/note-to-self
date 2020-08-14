# OData on SSIS

## The problem

Client wanted to copy a large amount of data (several gigabytes) from a production system through its OData endpoint and into a SQL Server database for faster/easier querying and reporting. The OData endpoint had several entities from which all available data was read.

I built a simple setup using SSIS' built-in components, and ended up with a Data Flow task that looked something like this:

    OData Source 1 --> Data Type Conversion --> ODBC Destination 1
    OData Source 2 --> Data Type Conversion --> ODBC Destination 2
    OData Source 3 --> Data Type Conversion --> ODBC Destination 3
    (and about 15 others)

This seemed to work great in Visual Studio. Hundreds of thousands of rows loaded with no issue. But when I ran it in production, the following error consistently broke the package:

> *{component name}* was unable to process the data. Unable to read data from the transport connection: An existing connection was forcibly closed by the remote host.

[A quick Google search](https://www.google.com/search?q=%28%22ssis%22+or+%22integration%22%29+%22odata%22+%22connection+was+forcibly+closed) revealed that I was not alone, but there were no simple solutions. I suspected the server dropped the connection due to the large amounts of data and long running connections.

## What worked for me

I found that if I simply limited the data flows to only two concurrent connections/OData Sources at a time, the package didn't fail.

In the end I had to do this the simplest possible way: Add a data flow task for every two OData sources, as such:

    Data Flow Task 1:
        OData Source 1 --> ...
        OData Source 2 --> ...
    Data Flow Task 2: (depends upon succesful completion of Data Flow Task 1)
        OData Source 3 --> ...
        OData Source 4 --> ...
    etc.

Seems kinda dumb, right? But it worked and was easy to reason about, if not the fastest possible way.

(If speed was a top priority, I would have had to introduce a .NET script to gain full control over parallelism and failure resilience. However, at this point one should *really* consider whether SSIS is the right tool for the job.)

## What didn't work

* The **EngineThreads** property seemed promising based on its description ([here](https://www.jamesserra.com/archive/2011/11/parallel-execution-in-ssis/) and [here](https://docs.microsoft.com/en-us/sql/integration-services/data-flow/data-flow-performance-features?view=sql-server-ver15)). It seems that setting it to a value of 2 should limit the number of concurrent transfers in a data flow containing all OData flows, but in my case, it didn't make a difference. Had it worked, this would have been the most elegant solution.

* I read a suggestion somewhere that **chunking data using a For Loop** around the Data Flow task combined with OData's **$skip** and **$top** operators could circumvent the issue. While that may be true in theory (the server didn't drop concurrent connections immediately, but only after a while) it introduced a set of other problems best avoided - among them an [issue with painfully slow validation of Data Flows using expressions](https://social.msdn.microsoft.com/Forums/sqlserver/en-US/7436d8e3-e084-4cdb-a62f-7bf0b097237f/validation-of-for-loop-very-slow-on-load?forum=sqlintegrationservices) and the fact that $skip/$top aren't ideal for server side paging; instead one should use $skiptoken and nextLink - which is what the OData Source does out of the box!
