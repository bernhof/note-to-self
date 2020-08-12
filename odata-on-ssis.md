# The troubles of OData on SSIS

Some notes from my (unfortunate) experiences with reading OData using SSIS and Visual Studio.

Working with OData on SSIS may seem a breeze at first, with the built-in [OData Connection Manager](https://docs.microsoft.com/en-us/sql/integration-services/connection-manager/odata-connection-manager) and [OData Source](https://docs.microsoft.com/en-us/sql/integration-services/data-flow/odata-source) but be warned, there are many hidden pitfalls and oddities that can't easily be worked around.

You may think that a Script Component (Source) is the way to go -- to roll your own OData Source, so to speak. At the time of writing, Service References for .NET Framework projects in Visual Studio 2017 and 2019 don't work out of the box with OData, although this may change soon through via the new [OData Connected Service extension](https://marketplace.visualstudio.com/items?itemName=laylaliu.ODataConnectedService), currently only in prerelease.

## TL;DR

**I didn't find any other option but to build the OData source manually using a Script Component (Source).**

In short, the built-in components (OData Source/Connection Manager) had insufficient fault handling features and caused inexplicable runtime issues. Despite attempting cumbersome workarounds, there was no apparent solution using built-in components and no scripting. Read on for details.

## Background

Client wanted to copy a large amount of data (several gigabytes) from a production system through its OData endpoint and into a SQL Server database for faster/easier querying and reporting. The OData endpoint had several entities from which all available data was read.

## My failed attempts

As a warning to those who consider venturing into the same mine field, I'll describe the things I tried that *didn't* work:

### Keeping it simple

First, I went with the simplest possible setup using built-in components, and ended up with a Data Flow task that looked something like this:

    OData Source 1 --> Data Type Conversion --> ODBC Destination 1
    OData Source 2 --> Data Type Conversion --> ODBC Destination 2
    etc.

This seemed to work great in Visual Studio. Hundreds of thousands of rows loaded with no issue. But when I ran it in production, the following error consistently broke the package:

> *{component name}* was unable to process the data. Unable to read data from the transport connection: An existing connection was forcibly closed by the remote host.  

[A quick Google search](https://www.google.com/search?q=%28%22ssis%22+or+%22integration%22%29+%22odata%22+%22connection+was+forcibly+closed) revealed that I was not alone, but there were no simple solutions. I suspected the server dropped the connection due to the large amounts of data and long running connections. This doesn't explain why it worked when executed through Visual Studio, but I still wanted to see if I could avoid it:

### Fiddling with OData Connection Manager settings

* **Max Received Message Size**: Couldn't find any value, large, small or in between, to solve this. If the value is too small, it will fail with other obscure errors.
* **Keep Alive**: Default is *false*, but setting this to *true* made no difference.

### Chunking via For Loop

Despite the fact that OData Source already loads data in chunks via `$skiptoken`/`nextlink`, I thought that if I broke the operation into smaller chunks, I could probably work around the issue. So I:

* Placed the Data Flow task in a For Loop
* Added tasks to count number of records before and after OData load to determine whether to continue looping
* Added variables to control how many records to skip & take (skip count was incremented on every loop)
* Modified all OData Sources to use these dynamic skip/take values as part of the queries ($skip=1000&$take=1000&...)

While this avoided the "connection forcibly closed" error, results were still disappointing:

* **Skip & take are not ideal for chunking**, since this can cause delays/timeouts on the server if there are no database indexes in place to support this type of paging (instead one should rely on `$skiptoken`/`nextlink` for server-side paging which the OData Source *does* support). The package kept failing at runtime (again, in production only) with the following error ([similar to what is described as a design-time problem on MSDN forums](https://social.technet.microsoft.com/Forums/en-US/bfe2cdd5-d6d1-40c8-be73-6d1ef6f224e7/ssis-odata-source-cannot-access-a-disposed-object-object-name-systemnethttpwebresponse?forum=sqlintegrationservices)):

    > Cannot access a disposed object.  Object name: 'System.Net.HttpWebResponse'.

* **Validation is painfully slow**. SSIS seems to validate the Data Flow task in every iteration of loop (why not just validate once??) which collided with an apparent [issue with expressionable properties](https://social.msdn.microsoft.com/Forums/sqlserver/en-US/7436d8e3-e084-4cdb-a62f-7bf0b097237f/validation-of-for-loop-very-slow-on-load?forum=sqlintegrationservices) on Data Flow tasks which causes slower than usual validation of the data flow task if expressions are used. I'd expect this to happen hundreds of times during execution.

In short, the For Loop didn't help in my situation.

## Script Component as a fallback solution

Usually I revert to Script Tasks/Components only if a problem cannot be solved in a reasonable manner using SSIS' built-in components. As a .NET developer, this isn't a big deal for me, but for others who may peek at or maintain the package with less or no developer experience, these scripts can seem like black boxes.

In this case, nothing else seemed to work. So down the rabbit hole I went.

***To be continued...***
