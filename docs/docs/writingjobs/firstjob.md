# Writing Your First Mantis Job

We'll be doing the classic word count example for streaming data. For this job we'll stream Tweets from Twitter, perform some application logic on the stream, and then write the data to a sink for consumption by other Mantis jobs. If you want to follow along check out the [Twitter Sample](https://github.com/Netflix/mantis-examples/tree/master/twitter-sample) project in our [mantis-examples](https://github.com/Netflix/mantis-examples/) repository.

There are a few things to keep in mind when implementing a Mantis Job.

* It is just Java. We need to implement a few interfaces but ultimately we're just writing Java.
* Mantis jobs are composed of a source, n stages, and a sink.
* Mantis makes heavy use of RxJava as a DSL for implementing processing logic.

## The Source
The source is responsible for ingesting data to be processed within the job. Many Mantis jobs will subscribe to other jobs and can simply use a templatized source such as `io.mantisrx.connectors.job.source.JobSource` which handles all the minutae of connecting to other jobs for us. If however your job exists on the edge of Mantis it will need to pull data in via a custom source. Since we're reading from the Twitter API we'll need to do this ourselves.

Our `TwitterSource` must implement `io.mantisrx.runtime.source.Source` which requires us to implement `call` and optionally `init`. Mantis provides some guarantees here in that `init` will be invoked exactly once and before `call` which will be invoked at least once. This makes `init` the ideal location to perform one time setup and configuration for the source and `call` the ideal location for performing work on the incoming stream. The objective of this entire class is to have `call` return an `Observable<Observable<T>>` which will be passed as a parameter to the first stage of our job.

Let's deconstruct the `init` method first. Here we will extract our parameters from the `Context` -- this allows us to write more generic sources which can be templatized and reused across many jobs. This is a very common pattern for writing Mantis jobs and allows you to iterate quickly testing various configurations as jobs can be resubmitted easily with new parameters.
  
```
/**
  * Init method is called only once during initialization. It is the ideal place to perform one time
  * configuration actions.
  *
  * @param context Provides access to Mantis system information like JobId, Job parameters etc
  * @param index   This provides access to the unique workerIndex assigned to this container. It also provides
  *                the total number of workers of this job.
  */
@Override
public void init(Context context, Index index) {

    String consumerKey = (String) context.getParameters().get(CONSUMER_KEY_PARAM);
    String consumerSecret = (String) context.getParameters().get(CONSUMER_SECRET_PARAM);
    String token = (String) context.getParameters().get(TOKEN_PARAM);
    String tokenSecret = (String) context.getParameters().get(TOKEN_SECRET_PARAM);
    String terms = (String) context.getParameters().get(TERMS_PARAM);

    Authentication auth = new OAuth1(consumerKey,
            consumerSecret,
            token,
            tokenSecret);

    StatusesFilterEndpoint endpoint = new StatusesFilterEndpoint();

    String[] termArray = terms.split(",");

    List<String> termsList = Arrays.asList(termArray);

    endpoint.trackTerms(termsList);

    client = new ClientBuilder()
            .name("twitter-source")
            .hosts(Constants.STREAM_HOST)
            .endpoint(endpoint)
            .authentication(auth)
            .processor(new StringDelimitedProcessor(twitterObservable))
            .build();


    client.connect();
}
```

Our `call` method is very simple thanks to the fact that our twitter client writes to a custom `BlockingQueue` adapter that we've written. We simply need to return an `Observable<Observable<T>>`.

```
@Override
public Observable<Observable<String>> call(Context context, Index index) {
    return Observable.just(twitterObservable.observe());
}

```

## The Stage

Our interfaces are functional interfaces and can consequently be implemented inline with a lambda function if the user so desires. We'll take advantage of this to define the stage inline with the job definition in the `TwitterJob` class.

```
@Override
public Job<String> getJobInstance() {
    return MantisJob
            
            // Invoke our TwitterSource
            .source(new TwitterSource())


            //
            // Define Our Stage
            // 

            // Much like our Source the stage takes a Context, but the second parameter is an Observable<T> (String in this case)
            // 
            .stage((context, dataO) -> dataO

                    // Deserialize data
                    .map(JsonUtility::jsonToMap)

                    // Filter for English Tweets
                    .filter((eventMap) -> {
                        if(eventMap.containsKey("lang") && eventMap.containsKey("text")) {
                            String lang = (String)eventMap.get("lang");
                            return "en".equalsIgnoreCase(lang);
                        }
                        return false;
                    })

                    // Extract Tweet body
                    .map((eventMap) -> (String)eventMap.get("text"))

                    // Tokenize Tweet Body
                    .flatMap((text) -> Observable.from(tokenize(text)))

                    // On a hopping window of 10 seconds
                    .window(10, TimeUnit.SECONDS)

                    // Reduce the windows into word/count pairs.
                    .flatMap((wordCountPairObservable) -> wordCountPairObservable
                            // count how many times a word appears
                            .groupBy(WordCountPair::getWord)
                            .flatMap((groupO) -> groupO.reduce(0, (cnt, wordCntPair) -> cnt + 1)
                                    .map((cnt) -> new WordCountPair(groupO.getKey(), cnt))))
                            .map(WordCountPair::toString)
                            .doOnNext((cnt) -> log.info(cnt))
                    , StageConfigs.scalarToScalarConfig())

            //
            // Reuse built in sink that eagerly subscribes and delivers data over SSE
            //

            .sink(Sinks.eagerSubscribe(Sinks.sse((String data) -> data)))
            .metadata(new Metadata.Builder()
                    .name("TwitterSample")
                    .description("Connects to a Twitter feed")
                    .build())
            .create();
}
```

## The Sink
The sink is handled on the single line below inline with the job definition. The job of the Sink is to make the data available to external systems which can range from ElasticSearch, S3, Hive, Kafka, and commonly Server Sent Events which other jobs can subscribe to. A more sophisticated sink might perform tasks such as serialization or handling MQL queries for downstream clients -- ours is just a simple SSE sink that we eagerly subscribe to.

```
.sink(Sinks.eagerSubscribe(Sinks.sse((String data) -> data)))
```

## The Job

All of this needs to be strung together and this is done via the `MantisJobProvider` class which defines our overall job and requires us to implement the `getJobInstance` method seen above in our implementation of the stage. The full class (package definition and imports elided) is below. We include a `main` method which invokes the local job executor in order to allow you to test your job locally.


```
/**
 * This sample demonstrates connecting to a twitter feed and counting the number of occurrences of words within a 10
 * sec hopping window.
 * Run the main method of this class and then look for a the SSE port in the output
 * E.g
 * <code> Serving modern HTTP SSE server sink on port: 8650 </code>
 * You can curl this port <code> curl localhost:8650</code> to view the output of the job.
 *
 * To run via gradle
 * ../gradlew execute --args='consumerKey consumerSecret token tokensecret'
 */
@Slf4j
public class TwitterJob extends MantisJobProvider<String> {

    @Override
    public Job<String> getJobInstance() {
        return MantisJob
                .source(new TwitterSource())
                // Simply echoes the tweet
                .stage((context, dataO) -> dataO
                        .map(JsonUtility::jsonToMap)
                        // filter out english tweets
                        .filter((eventMap) -> {
                            if(eventMap.containsKey("lang") && eventMap.containsKey("text")) {
                                String lang = (String)eventMap.get("lang");
                                return "en".equalsIgnoreCase(lang);
                            }
                            return false;
                        }).map((eventMap) -> (String)eventMap.get("text"))
                        // tokenize the tweets into words
                        .flatMap((text) -> Observable.from(tokenize(text)))
                        // On a hopping window of 10 seconds
                        .window(10, TimeUnit.SECONDS)
                        .flatMap((wordCountPairObservable) -> wordCountPairObservable
                                // count how many times a word appears
                                .groupBy(WordCountPair::getWord)
                                .flatMap((groupO) -> groupO.reduce(0, (cnt, wordCntPair) -> cnt + 1)
                                        .map((cnt) -> new WordCountPair(groupO.getKey(), cnt))))
                                .map(WordCountPair::toString)
                                .doOnNext((cnt) -> log.info(cnt))
                        , StageConfigs.scalarToScalarConfig())
                // Reuse built in sink that eagerly subscribes and delivers data over SSE
                .sink(Sinks.eagerSubscribe(Sinks.sse((String data) -> data)))
                .metadata(new Metadata.Builder()
                        .name("TwitterSample")
                        .description("Connects to a Twitter feed")
                        .build())
                .create();
    }

    private List<WordCountPair> tokenize(String text) {
        StringTokenizer tokenizer = new StringTokenizer(text);
        List<WordCountPair> wordCountPairs = new ArrayList<>();
        while(tokenizer.hasMoreTokens()) {
            String word = tokenizer.nextToken().replaceAll("\\s*", "").toLowerCase();
            wordCountPairs.add(new WordCountPair(word,1));
        }
        return wordCountPairs;
    }


    public static void main(String[] args) {

        String consumerKey = null;
        String consumerSecret = null;
        String token = null;
        String tokenSecret = null;
        if(args.length != 4) {
            System.out.println("Usage: java com.netflix.mantis.examples.TwitterJob <consumerKey> <consumerSecret> <token> <tokenSecret");
            System.exit(0);
        } else {
            consumerKey = args[0].trim();
            consumerSecret = args[1].trim();
            token = args[2].trim();
            tokenSecret = args[3].trim();
        }

        LocalJobExecutorNetworked.execute(new TwitterJob().getJobInstance(),
                new Parameter(TwitterSource.CONSUMER_KEY_PARAM,consumerKey),
                new Parameter(TwitterSource.CONSUMER_SECRET_PARAM, consumerSecret),
                new Parameter(TwitterSource.TOKEN_PARAM, token),
                new Parameter(TwitterSource.TOKEN_SECRET_PARAM, tokenSecret)

        );
    }
}
```

# Wrapping Up

You can view the full source of the [Twitter Sample]()