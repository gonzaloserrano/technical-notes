# Objective

How to quickly import data from Percona to DynamoDB using Go.

## Guidelines

Figure out how to gather the data from Percona.

Bear in mind that DynamoDB has several types of limits: http://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Limits.html#limits-capacity-units-provisioned-throughput 
- provisioned throughput limits:
    - the production table is in use so we should not interfere with the current work load (exceed the current limits, which are 20 ops).
    - the solution will need rate limiting to not exceed the max limits (40K ops)
- size limits: since the token data is small (< 4KB, which is what troughput size is based on) this won't be a problem.

Take advantadge of Go's concurrency features: https://github.com/golang/go/wiki/LearnConcurrency

Since the token repository is an interface and we have already an in-memory implementation for the tests, add a delay capability to emulate a real database latency (lets say e.g 10ms for DynamoDB).

Avoid premature optimitzations: we don't know the bottleneck (cpu, i/o, memory, network...) yet.

## Steps followed

- create a CSV with the data using mysqldump http://dev.mysql.com/doc/refman/5.7/en/mysqldump-delimited-text.html
- create a func that receives a io.Reader so we can start the first test without the need of a file
- start with a sequential algorithm and the in-memory repository, and measure times
- then add concurrency (fan-out pattern) with N workers, where N os the num of CPUs. Measure times, should be better but depends on the scenario.
- then add rate limiting (e.g a ticker with a channel and a counter works beautifully)
- make all this tunning parametrizable so you can try different values from the test

Once we have the basic algorithm tested, try with lots of data and a real database (e.g DynamoDB) and measure.

Things found during this testing:

- created a table similar to the production one with terraform very easily to make tests
- provision DynamoDB from 20 wps of throughput to 30K can take a lot of time, also depends on from where you do it weird things happen, e.g from the web console sometimes it got stuck without updating the values but then via CLI it worked. As an advice, use the CLI, eg:
- change table throughput to X:
  `aws dynamodb update-table --table-name table_name --table '{ "ReadCapacityUnits": X, "WriteCapacityUnits": X }'`
- change GSI throughput to X
  `aws dynamodb update-table --table-name table --global-secondary-index-updates '[{"Update": { "IndexName": "table_index", "ProvisionedThroughput": { "ReadCapacityUnits": X, "WriteCapacityUnits": X } }}]'`
- poll change propagation
  `watch 'aws dynamodb describe-table --table-name table | jq . | grep Write'`
- found using datadog's metrics that the network bandwith was the limit (250 Mbs) and i could get 2K wps. Since I was launching it from a frontend server which is x-large,
i checked for a better network instance at http://ec2instances.info which are the 10 Gigabit ones, chose the cheaper (x3.8xlarge) and launched a new instance with terraform (so i could use the same AMI etc)
- with the new instance I could arrive to much higher wps but got stuck at 15K i think, so i checked dynamodb's batch writing capabilities http://docs.aws.amazon.com/amazondynamodb/latest/APIReference/API_BatchWriteItem.html and with that i could get 30K wps which is 75% of default Dynamo's soft limit (40K ops).
- after reaching 750K connections i got a nice "cannot assign requested address" which i fixed with this `http.DefaultTransport.(*http.Transport).MaxIdleConnsPerHost = 250` after reading https://paperairoplane.net/?p=556

## Final results

- launched the 10M csv token file with a rate limiting of 20K wps and it took ~0.4s per group and a total time of ~8m, with 40K ops it would have taken 4m
- cost:
    - 30K wps throughput is ~30K$ / month or 50$/h, so do it quickly and then lower the throughput ASAP to normal levels.
    - c3.8x large instance is 1.6$/h, after using it you can destroy it.
