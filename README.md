# MongoDB to Azure Cosmos DB Migration 
/* Azure Cosmos DB LoadTest Sample Code

•	An Azure Cosmos DB for NoSQL container with a throughput of 20,000 RU/s
•	Item size of 0.2 KB
•	Operations primarily to be tested: point read and upsert
•	5,000 async writes including 40% creates and 60% increments (using Patch API)
•	10 threads
•	Java SDK v4 4.36.0
•	Account configured with Direct Connectivity mode, Session Consistency
•	Test VM: South Central US
•	Azure Region hosting Azure Cosmos DB: South Central US

*/

Java SDK v4 code for Azure Cosmos DB for NoSQL Perf Test done for a customer. <br>
If you have queries on code below, drop me a note at: sugh @ microsoft dot com

Outline of load test code
----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
        // Issue N async queries in a loop, with optional delay between each query
        public void queryLoadTest() {
        for (int i=0; i<maxNumReads; i++) {
                // Get a random grade key that was already written to earlier
                int keyIdx = gradeKeyRand.nextInt(allKeys.size()); 
                        String key = allKeys.get(keyIdx);

                // Get a random grade parameter name that was already written to earlier
                List<String> gradeNames = new ArrayList<>(gradesWritten.get(key).keySet());
                int gradeNameIdx = gradeTypeRand.nextInt(gradeNames.size());
                String gradeName = gradeNames.get(gradeNameIdx);

                String pkey = "grc.com|" + gradeName + "|" + TimeUnit.DAY + "|" + key;

                // Revised approach – fire async query each time in the loop by doing subscribe()
                queryByPartitionKeyAsync("Day", "pk", pkey, GradeRec.class)
                    .subscribe();

                // Optionally add a delay between each query – needed to set to 1 ms to avoid longer cold start / sporadic spikes
                if (readDelayMs != null) {
                    Thread.sleep(readDelayMs);
                }
        }


            // Earlier I was firing all fluxes at about the same time and collecting results (scatter-gather approach)
            // Flux.merge(fluxes).collectList().block();

        }

        public class GradeRec {
              private String pk;    // domain | name | timeUnit | key
              private String id;    // timeUnitIndex
              private Number val;
              private Integer ttl;
              ...
        }


        public <T> Flux<T> queryByPartitionKeyAsync(String containerId, String pkeyField, String pkey, Class<T> itemClz) {

            CosmosQueryRequestOptions queryOpts = new CosmosQueryRequestOptions().setPartitionKey(new PartitionKey(pkey));
            List<SqlParameter> params = Arrays.asList(new SqlParameter("@pkey", pkey));
            SqlQuerySpec querySpec = new SqlQuerySpec("SELECT * FROM c WHERE c." + pkeyField + " = @pkey", params);

            TimeKeeper tk = new TimeKeeper();
            MutableDouble RUs = new MutableDouble();
            MutableInt numRecs = new MutableInt();
            return getContainer(containerId).queryItems(querySpec, queryOpts, itemClz).byPage(1000)
                .publishOn(Schedulers.boundedElastic())
                .doOnNext(resp -> {
                    RUs.add(resp.getRequestCharge());
                    numRecs.add(resp.getResults().size());
                    List<ClientSideRequestStatistics> reqStatsList = BridgeInternal.getClientSideRequestStatisticsList(resp.getCosmosDiagnostics());
                    tk.addDurationInNanos(reqStatsList.get(0).getDuration().toNanos());
                    if (m_cfg.isEnableQueryDiagnostics()) {
                        System.err.println(resp.getCosmosDiagnostics().toString());
                    }
                })
                .doOnError(e -> {
                    // s_teller.caught(e, "Query " + querySpec.getQueryText());
                    if (e instanceof CosmosException) {
                       updateErrorCounter("queryByPartitionKey", ((CosmosException) e).getStatusCode());
                    }
                })
                .doOnTerminate(() -> {
                    double RUsVal = RUs.doubleValue();
                    int numRecsVal = numRecs.intValue();
                    long durationNanos = tk.elapsedTimeNanos();
                    long durationNanosLib = tk.getTotalDurationInNanos();
                    updateMetrics("queryByPartitionKey", RUsVal, durationNanos, true);
                    updateMetrics("queryByPartitionKey-lib", RUsVal, durationNanosLib, true);
                    updateHistogram("queryByPartitionKey-numRecs", numRecsVal);
                })

                .flatMapIterable(com.azure.cosmos.models.FeedResponse::getResults);
        }

Driver Init
----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
    ThrottlingRetryOptions retryOpts = new ThrottlingRetryOptions();
    retryOpts.setMaxRetryAttemptsOnThrottledRequests(0);
    CosmosClientBuilder clientBuilder = new CosmosClientBuilder()
            .endpoint("https://<<cosmos-db-endpoint-URL")
            .key(key)
            .preferredRegions(Arrays.asList("South Central US"))
            .consistencyLevel(ConsistencyLevel.SESSION)
            .throttlingRetryOptions(retryOpts);


Customized connCfg rntbd network connectivity parameters for Azure Cosmos DB for NoSQL
----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
    DirectConnectionConfig directConnCfg = DirectConnectionConfig.getDefaultConfig();
    directConnCfg.setConnectTimeout(Duration.ofMillis(600));
    directConnCfg.setNetworkRequestTimeout(Duration.ofSeconds(5));
    directConnCfg.setIdleConnectionTimeout(Duration.ofSeconds(0));
    directConnCfg.setIdleEndpointTimeout(Duration.ofHours(1));
    directConnCfg.setMaxConnectionsPerEndpoint(350);
    clientBuilder.directMode(directConnCfg);
    CosmosAsyncClient client = clientBuilder.buildAsyncClient();
