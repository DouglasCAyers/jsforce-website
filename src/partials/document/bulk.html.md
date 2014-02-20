---
---

## Bulk API

JSforce package also supports Bulk API. It is not only mapping each Bulk API endpoint in low level,
but also introducing utility interface in bulk load operations.


### Load From Records

First, assume that you have record set in array object to insert into Salesforce.

```javascript
//
// Records to insert in bulk.
//
var accounts = [
{ Name : 'Account #1', ... }, 
{ Name : 'Account #2', ... }, 
{ Name : 'Account #3', ... }, 
...
];
```

You can use `SObject#create(record)`, but it consumes API quota per record,
so not practical for large set of records. We can use bulk API interface to load them.

Similar to Salesforce Bulk API, first create bulk job by `Bulk#createJob(sobjectType, operation)`
through `bulk` API object in connection object.

Next, create a new batch in the job, by calling `Bulk-Job#createBatch()` through the job object created previously.

```javascript
var job = conn.bulk.createJob("Account", "insert");
var batch = job.createBatch();
```

Then bulk load the records by calling `Bulk-Batch#execute(input)` of created batch object, passing the records in `input` argument.
 
When the batch is queued in Salesforce, it is notified by `queue` event, and you can get job ID and batch ID.

```javascript
batch.execute(accounts);
batch.on("queue", function(batchInfo) { // fired when batch request is queued in server.
  batchId = batchInfo.id);
  jobId = batchInfo.jobId);
  // ...
});
```

After the batch is queued and job / batch ID is created, wait the batch completion by polling.

When the batch process in Salesforce has been completed, it is notified by `response` event with batch result information.

```javascript
var job = conn.bulk.job(jobId);
var batch = job.batch(batchId);
batch.poll(1000 /* interval(ms) */, 20000 /* timeout(ms) */); // start polling
batch.on("response", function(rets) { // fired when batch finished and result retrieved
  if (err) { return console.error(err); }
  for (var i=0; i < rets.length; i++) {
    if (rets[i].success) {
      console.log("#" + (i+1) + " loaded successfully, id = " + rets[i].id);
    } else {
      console.log("#" + (i+1) + " error occurred, message = " + rets[i].errors.join(', '));
    }
  }
  // ...
});
```

Alternatively, you can use `Bulk#load(sobjectType, operation, input)` interface to achieve the above process in one method call.

```javascript
conn.bulk.load("Account", "insert", accounts, function(err, rets) {
  if (err) { return console.error(err); }
  for (var i=0; i < rets.length; i++) {
    if (rets[i].success) {
      console.log("#" + (i+1) + " loaded successfully, id = " + rets[i].id);
    } else {
      console.log("#" + (i+1) + " error occurred, message = " + rets[i].errors.join(', '));
    }
  }
  // ...
});
```

Following are same calls but in different interfaces :

```javascript
conn.sobject("Account").insertBulk(accounts, function(err, rets) {
  // ...
});
```

```javascript
conn.sobject("Account").bulkload("insert").execute(accounts, function(err, rets) {
  // ...
});
```

### Load From CSV File

It also supports bulk loading from CSV file. Just use CSV file input stream as `input` argument 
in `Bulk#load(sobjectType, operation, input)`, instead of passing records in array.

```javascript
//
// Create readable stream for CSV file to upload
//
var csvFileIn = require('fs').createReadStream("path/to/Account.csv");
//
// Call Bulk#load(sobjectType, operation, input) - use CSV file stream as "input" argument
//
conn.bulk.load("Account", "insert", csvFileIn, function(err, rets) {
  if (err) { return console.error(err); }
  for (var i=0; i < rets.length; i++) {
    if (rets[i].success) {
      console.log("#" + (i+1) + " loaded successfully, id = " + rets[i].id);
    } else {
      console.log("#" + (i+1) + " error occurred, message = " + rets[i].errors.join(', '));
    }
  }
  // ...
});
```

`Bulk-Batch#stream()` returns a Node.js standard writable stream which accepts batch input.
You can pipe input stream to it afterward.


```javascript
var batch = conn.bulk.load("Account", "insert");
batch.on("response", function(rets) { // fired when batch finished and result retrieved
  if (err) { return console.error(err); }
  for (var i=0; i < rets.length; i++) {
    if (rets[i].success) {
      console.log("#" + (i+1) + " loaded successfully, id = " + rets[i].id);
    } else {
      console.log("#" + (i+1) + " error occurred, message = " + rets[i].errors.join(', '));
    }
  }
);
//
// When input stream becomes available, pipe it to batch stream.
//
csvFileIn.pipe(batch.stream());
```
  

### Update / Delete Queried Records

If you want to update / delete records in Salesforce which match specified condition in bulk, 
now you don't have to write a code which download & upload records information. 
`Query#update(mapping)` / `Query#destroy()` will directly manipulate records.


```javascript
/* @interactive */
// DELETE FROM Account WHERE CreatedDate = TODAY
conn.sobject('Account')
    .find({ CreatedDate : jsforce.Date.TODAY })
    .destroy(function(err, rets) {
      if (err) { return console.error(err); }
      console.log(rets);
      // ...
    });
```

```javascript
/* @interactive */
// UPDATE Opportunity 
// SET CloseDate = '2013-08-31'
// WHERE Account.Name = 'Salesforce.com'
conn.sobject('Opportunity')
    .find({ 'Account.Name' : 'Salesforce.com' })
    .update({ CloseDate: '2013-08-31' }, function(err, rets) {
      if (err) { return console.error(err); }
      console.log(rets);
      // ...
    });
```

In `Query#update(mapping)`, you can include simple templating notation in mapping record.

```javascript
/* @interactive */
//
// UPDATE Task 
// SET Description = CONCATENATE(Subject || ' ' || Status)
// WHERE ActivityDate = TODAY
//
conn.sobject('Task')
    .find({ ActivityDate : jsforce.Date.TODAY })
    .update({ Descritpion: '${Subject}  ${Status}' }, function(err, rets) {
      if (err) { return console.error(err); }
      console.log(rets);
      // ...
    });
```

To achieve further complex mapping, `Query#update(mapping)` accepts mapping function in `mapping` argument.

```javascript
/* @interactive */
conn.sobject('Task')
    .find({ ActivityDate : jsforce.Date.TODAY })
    .update(function(rec) {
      return {
        Descritpion: rec.Subject + ' ' + rec.Status
      }
    }, function(err, rets) {
      if (err) { return console.error(err); }
      console.log(rets);
      // ...
    });
```

If you are creating query object from SOQL by using `Connection#query(soql)`,
the bulk delete/update operation cannot be achieved because no sobject type information available initially.
You can avoid it by passing optional argument `sobjectType` in `Query#destroy(sobjectType)` or `Query#update(mapping, sobjectType)`.

```javascript
/* @interactive */
conn.query("SELECT Id FROM Account WHERE CreatedDate = TODAY")
    .destroy('Account', function(err, rets) {
      if (err) { return console.error(err); }
      console.log(rets);
      // ...
    });
```

```javascript
/* @interactive */
conn.query("SELECT Id FROM Task WHERE ActivityDate = TODAY")
    .update({ Descritpion: '${Subject}  ${Status}' }, 'Task', function(err, rets) {
      if (err) { return console.error(err); }
      console.log(rets);
      // ...
    });
```

NOTE: Be careful when using this feature not to break/lose existing data in Salesforce.
Careful testing is recommended before applying the code to your production environment.

