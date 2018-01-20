# Django Channels

> ###### channels is a project to make Django able to handle more than just plain HTTP requests,includings websockets and http2,as well as the ability to run code after a response has been sent for things like thumbnailing or background calculation.
>
> What is a channels?it's a ordered,first-in first-out queue with message expiry and at-most-once delivery to only one listener at a tine.

## Channel Types

There are actually two major uses for channels in this model.the first,and more obvious one,is the dispatching of work to consumers -a message gets added to a channel, and then any one of the works can pic it up and run the consumer.

## Groups

Because channels only deliver to a single listener,they can't do broadcast;if ypu want to send a message to an arbitrary of clients ,you need to keep track of which reply channels of those you wish to send to.

If i had a liveblog where i wanted to push out udates whenever a new post is saved,I could register a handler for the post\_save signal and keep a set of channels\(here,using Redis\) to send updates to:

```py
redis_conn = redis.Redis("localhost",6379)

@receiver(post_save,sender=BlogUpdate)
def send_update(sender,instance,**kwargs):
    #loop through all reply channels and send the update
    for reply_channel in redis_conn.smenbers("readers"):
        Channel(reply_channel).send({
            "text":json.dump({
                "id":instance.id,
                "context":instance.context
            })
        })
##Connected to websocket.connect
def ws_connect(message):
    redis_conn.sadd("readers",message.reply)
```

> while this will work,there's a small problem - we never remove people from the **readers **set when they disconnect.We could add a consumer that listens to **websocket.disconnect** to do that,but we'd also need to have some kind of expiry in case an interfance servers is forced to quit or loss power before it can send disconnect signals-your code will never see any disconnect notifications but the reply channels is completely invalid and messages you send there will sit threr until they expire.
>
> Because the basic design of channels is stateless, the channels server has no concept of "closing" a channel if an interface server goes away-after all,channels are meant to hold messages util a consumer comes along\(and some types of interface server,e.g. an SMS gateway,could theoretically serve any client from any interface server\).
>
> We don't particularly care if a disconnected client doesn't get message sent to the group after allit disconnected-but we do care about cluttering up the channel backend tracking all of these clients that are no longer around\(and possible,eventually getting a clooision on the reply channel name and sending someone messages not meant for them,though that would likely take weeks \)
>
> Now,We could go back into our example above and add an expiring set and keep track of expiry times and so forth,but what would be the point of a framework if it made you add boilerplate code? Instead,Channels implements this abstraction as a core concept called Groups:



