# Blob State Building Block
- Author: Whit Waldo (@whitwaldo)
- Updated: 2025-01-05

## Overview
This is another of the [many proposals](https://github.com/dapr/dapr/issues/7339) to overhaul the state building block by introducing purpose-specific second-generation state management building blocks that turn the existing state API into a general purpose fallback and instead more easily move the needle forward to better accommodating specific scenarios.

This proposal focuses on the implementation of a state store focused on persisting and retrieving blobs. While there are blob-based storage providers for the general-purpose state API, this was imagined before Dapr supported streaming operations and as such, the API isn't well-suited to potentially passing around large sets of data. The proposed shape of this API changes some aspects of the current state store in meaningful ways that break from past designs and offer a concise and simple API that's specific to the challenge of storing and retrieving state efficiently and nothing more. Should a developer want something more from the API like querying the values with some constraint or persisting values in a more time-series-adjacent way, they're encouraged to turn to other specialized state stores for that functionality such as the proposed [Document Store](https://github.com/dapr/dapr/issues/5146).

## What's the purpose of this blob store?
Put simply, we're striving for the management of potentially large and monolithic sets of data that are streamed to and from the client. In much the same way as how the cryptography building block allows bidirectional streaming of chunked plaintext and ciphertext payloads so as to reduce resources taken by the SDK and Dapr runtime to process the request, we'd seek to do the same thing here as well.

I do not presently know if Dapr has cancellation token support, but that could be a very useful capability to have as part of this implementation in that it would allow a potentially long-running operation to cancel partway through rather than suddenly deal with an interrupted connection from the client and draw conclusions accordingly.

## How might this differ from other possibile state stores?
Two alterntaives come to mind that I wanted to call out and differntiate this use-case from: document storage and object storage. 

### Document Storage
When I think about document storage, I think of a document database like Azure's Cosmos DB or MongoDB that support data in a document-oriented format like JSON, XML or BSON. Each document can contain nested structed documents. It's ideal for applications that require flexible semi-structured data storage and are looking to access it via NoSQL and SQL-style APIs as opposed to a key/value style lookup operation. 

In comparison, this blob storage proposal wouldn't support value queries and while the data might be persisted in a queryable format, it would only be persisted and retrieved as a full block blob.

### Object Storage
Similarly, when thinking about object storage, I think of a more versatile version of blob storage that allows me to pair metadata alongside the object such as descriptions, ad-hoc properties (e.g. categorization or versioning) or tags. I imagine a more flexible set of APIs that access this and allow queries against those tags and metadata.

In comparison, this proposal seeks to offer a narrower and specific version of an object storage. This one targets the simple storage of large binary objects that are streamed to and from the provider and offer no additional query or metadata support. Rather, I propose that an object storage be built as a separate block for those developers that need such support.

## Why not use the existing state store?
The current state store API was designed before streaming was available to building blocks making it poorly-suited to the requirements of this scenario. Further, it was designed to be a general purpose solution that fit several different use cases in a way that this proposal doesn't strive to - for example, this proposal doesn't provide for transactions or consistency, so it would be poorly suited as the basis for an actor state store. That said, there are several capabilities of the existing state store that this does intend to add support for:

- List keys (acknowledging that this is still in a proposal phase)
- ETags
- TTL

And while it'd add support or each of those and will strive to keep existing implementation details, it makes sense to take this opportunity to make a clean break and implement differently if necessary because as is the problem with the existing state store, it's difficult to modify the API once published as stable and maintain the full provider compatibility.

## What about the other object storage proposal?
There's an existing storage proposal [here](https://github.com/dapr/proposals/pull/18/files#top) and I've referenced this while writing this proposal. Put simply, the existing proposal speaks to a slightly different and broader purpose than this one. I'd argue that its name reflects its different purpose: it's an `Object storage building block` and this strives to be a more specific `Blob storage building block`. While the interface used by @ItalyPaleAle would even be broadened to better accomodate the wider range of capabilities I attribute to an object store (above), I do think it's an excellent starting place for this proposal. 

## Component YAML
This component is expected to have similar attributes to existing state stores with some variation:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
    name: blobstore
spec:
    type: state.blobstore.<providerName>
    version: v1
    metadata:
        # Various properties necessary for component registration
```

### Implementation Overview
There are a few guiding principals I've stuck to while thinking through the shape of this and other state proposals:
- All SDK operations should be implemented as asynchronous operations in a way that minimize back and forth operations with the Dapr sidecar.
- The identifier will be represented as a `string` and the value as a `byte[]` so it's easily chunked. This also leave as an exercise to the developer or SDK precisely how the blob payload is created and whether it should undergo any additional operations before being written to state (e.g. formatting, encryption, compressiong, encoding, etc.). Ideally, ths various SDK maintainers can develop a uniform approach to this (a POC exists in the .NET SDK [here](https://github.com/dapr/dotnet-sdk/pull/1378)) so that data encoded in C# is easily decoded from any other language SDKs.
- The only metadata stored alongside the value should be those values accepted by the provider themselves (e.g. TBD). I would anticipate there'd be support for common values like "Last-Modified", "Content-Length", "Content-Type", "Content-MD5", "Content-Encoding", "Content-Language", and perhaps others, but this would require more research. I would propose passing these as headers as part of the intial persistence request and persisted as the provider supports it.

### gRPC APIs
In the Dapr gRPC APIs, we'd extend the `runtime.v1.Dapr` service to add new methods:

| Note: APIs will have Alpha1 suffixed to the type names while in preview

| Note: Any authentication behaviors are maintained in the component YAML configuration

```proto
service Dapr {

}
```

ListBlobs
PutBlob
GetBlob
GetBlobProperties
DeleteBlob

### HTTP APIs


