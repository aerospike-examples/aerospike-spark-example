# aerospike-spark
Example tutorial for Aerospike-and-Spark connector from Ken Tune

This example does the following:

- Creates a data set of Australian Prime Ministers and saves it to Aerospike via Spark.
- Retrieves a subset of the data using the `aeroJoin` operation.
- Filters the data by party affiliation to get only Labor Party members.
- Writes the results to a new Aerospike set.
