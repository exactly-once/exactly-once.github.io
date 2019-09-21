The business code can be divided into multiple categories, based on what type of external resources it interacts with

## Write-only resources

The write-only resources do not offer any way of querying. Why is that important? Imagine a situation when the code requested some operation and immediately crashed. Once the system is up again it has no way to check if the crash happened right before submitting the operation or right after. In other words, the code does not know what state whe system is in. An example is a SMTP server. When the application needs to send an e-mail it sends the message to the SMTP server. If that application restarts during that operation, it cannot ask the SMTP server if it received and forwarded a given e-mail.

## Blob resources

Blob resources allow CRUD operations but do not allow storing any out-of-band information along the actual data. An example would be an image file stored in the Azure Blob Storage or S3 bucket. 

## Metadata-enabled blob resources

If the blob format allows adding meta information it gives the application much more power. An example is a json document representing a customer's order stored in the Blob Storage. Such a document can contain any out-of-band information. This ability is critical when trying to ensure exactly-once message processing.

## Multi-entity transactional resources

Some resources allow for multi-entity transactions. An example is a SQL database. Such resources allow storing out-of-band information related to message processing in separate entities.

## Query-before-submit resources

A more advanced version of write-only of resources allows the application to ask about the state of a given operation in case of an unexpected restart. An example would be a web service that allows the client to pass a reference number of each operation. Before the client calls the web service, it generates and persists the reference number. Later, when it crashes and restarts it can use that stored value to check if a given operation has been receiver and processed by the web service. If not, it can re-submit the operation.

The problem with such resources is the fact that there is a race condition between the submit and query. The query might return false because the submit message has been delayed by the network. The client thinks the operation did not make it through and re-submits. As a result, the web service receives the operation twice.

## ID-based de-duplication resources

An even more advanced category of resources performs de-duplication of submitted operations based on the reference number provided by the client. An example would be a web service that, before processing an operation, ensures it has not been processed before. In the latter case it ignores the message.

## Summary

Based on the resource category, different strategies are required to ensure correctness of message processing:

- write-only resources cannot ever guarantee exactly-once. Use best-effort de-duplication like FIFO queues or ASB to limit likelihood of duplicates
- prefer ID-based de-duplication resources to query-before-submit resources whenever possible. If not possible, use FIFO queues to limit likelihood of race condition
- when working with blob resources always replace the whole value. Blob resources cannot guarantee exacly-once semantics for appending
- ID-based de-duplication or query-before-submit resources need to be controlled via a metadata-enabled or transactional-resource in order to be able to store the reference number
- Metadata-enabled blob and transactional resources allow exactly-once processing via inbox/outbox patterns
