## Chapter 11: Stream Processing

- ### What is batch processing?

    - In principle, a file or database is sufficient to connect producers and consumers: a
producer writes every event that it generates to the datastore, and each consumer
**periodically** polls the datastore to check for events that appeared since it last ran. This
is essentially what a batch process does when it processes a day’s worth of data at the
end of every day"

- ### Why we need strem processing? 

    - a lot of data is unbounded because it arrives gradually over time: your users
produced data yesterday and today, and they will continue to produce more data
tomorrow.

- ### Why we need Message System?(eg: Kafka, RabbitMQ, ZMQ)

    - "However, when moving towards continual processing with low delays, polling
becomes expensive if the datastore is not designed for this kind of usage. The more
often you poll, the lower the percentage of requests that return new events, and thus
the higher the overheads become. Instead, it is better for consumers to be notified()
when new events appear"

- ### Why storing data permanently (durable storage) is so important?

    - receiving a message is destructive if processing it causes it to be deleted from
the broker( friendly debug )

    -  add a new client
at any time, and it can read data written arbitrarily far in the past (as long as it has
not been overwritten or deleted).



## Stream Processing

    - ### 
    
        
    

