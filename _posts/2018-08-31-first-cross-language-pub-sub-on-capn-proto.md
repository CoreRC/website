---
title: First Cross-language Pub/Sub on Cap'n Proto
---

[Update: you can follow the discussion on https://groups.google.com/forum/#!topic/capnproto/y2ePOd25pkE]

After months of experimentation with tons of messaging concepts, I finally decided to use Cap'n Proto RPC for the CoreRC project. However, there is little information on the web for Cap'n Proto in general, and most of them are not up-to-date. So I hope this post can help you to get started on using Cap'n Proto for designing cross-language communication concepts and primitives.

Most of the post is based on the `capnproto-rust` [demo](https://github.com/capnproto/capnproto-rust/tree/master/capnp-rpc/examples/pubsub) implementation of the `PubSub` scheme. Huge thanks to dwrensha for such a nice work ;)

## The Schema

Here is the `pubsub.capnp` schema file. Note that interfaces are like abstract classes in the Cap'n Proto world--they come with a lot of scaffolds to ease the interaction between the communicating parties. However, you only need to implement these interfaces when you are in charge of the function of the interface. That is to say, if you are the user, you only need to include the generated header, and rest assured that you can start using it.

```c++
@0xce579c3e9bb684bd;

using Cxx = import "/capnp/c++.capnp";
$Cxx.namespace("corerc");

interface Subscription {}

interface Publisher(T) {
    # A source of messages of type T.

    subscribe @0 (subscriber: Subscriber(T)) -> (subscription: Subscription);
    # Registers `subscriber` to receive published messages. Dropping the returned `subscription`
    # signals to the `Publisher` that the subscriber is no longer interested in receiving messages.
}

interface Subscriber(T) {
    pushMessage @0 (message: T) -> ();
    # Sends a message from a publisher to the subscriber. To help with flow control, the subscriber should not
    # return from this method until it is ready to process the next message.
}
```

## The Demo Server in Rust

Information about Cap'n Proto scatters everywhere in the web, but the demo implementation in `capnproto-rust` is largely self-introductory.

First we must add the implementation details to Cap'n Proto generated code: 
```rust
struct SubscriptionImpl {
    id: u64,
    subscribers: Rc<RefCell<SubscriberMap>>,
}

impl SubscriptionImpl {
    fn new(id: u64, subscribers: Rc<RefCell<SubscriberMap>>) -> SubscriptionImpl {
        SubscriptionImpl { id: id, subscribers: subscribers }
    }
}

impl Drop for SubscriptionImpl {
    fn drop(&mut self) {
        println!("subscription dropped");
        self.subscribers.borrow_mut().subscribers.remove(&self.id);
    }
}

impl subscription::Server for SubscriptionImpl {}
```
Note here how the unsubscribing logic is implemented in the destructor for `SubscriptionImpl`. When a Subscriber subscribes to a topic, the following logic is executed:
```rust
impl publisher::Server<::capnp::text::Owned> for PublisherImpl {
    fn subscribe(&mut self,
                 params: publisher::SubscribeParams<::capnp::text::Owned>,
                 mut results: publisher::SubscribeResults<::capnp::text::Owned>,)
                 -> Promise<(), ::capnp::Error>
    {
        println!("subscribe");
        self.subscribers.borrow_mut().subscribers.insert(
            self.next_id,
            SubscriberHandle {
                client: pry!(pry!(params.get()).get_subscriber()),
                requests_in_flight: 0,
            }
        );

        results.get().set_subscription(
            subscription::ToClient::new(SubscriptionImpl::new(self.next_id, self.subscribers.clone()))
                .from_server::<::capnp_rpc::Server>());

        self.next_id += 1;
        Promise::ok(())
    }
}
```
Where we first construct a `SubscriberHandle` to remember the list of current active subscribers and their IDs. After that, we construct a `subscription` with the unsubscription logic, and pass the **corresponding Client** to the subscriber. When the corresponding client got destroyed, the server object will also be destroyed, thus unsubscription becomes automatic. This methodology is in accordance with the Capn'n Proto styleguide [here](https://github.com/capnproto/capnproto/blob/master/style-guide.md), where RAII is almost enforced.

## Implementing the Client in C++

The C++ implementation is easy once you know the philosophy of Cap'n Proto. A lot of objects in Cap'n Proto actually contains pointers to other things, so it becomes important that you keep things you need in scope. for example, `auto rpcSystem = capnp::makeRpcClient(network);` actually holds the corresponding `capnp::TwoPartyVatNetwork` as a pointer, thus if you return only the `rpcSystem` in a lambda, you will get a mysterious crash that is hard to spot especially when you do not have a debug build of the library. Other than that, the implementation is pretty easy.

As in the Rust server implementation, you first need to implement your part of the business logic:
```c++
class SubscriberImpl final: public corerc::Subscriber<capnp::Text>::Server
{
    public:

    kj::Promise<void> pushMessage(PushMessageContext context) override
    {
        struct timeval tv;
        gettimeofday (&tv, NULL);

        std::cout << "Message From Publisher: " << context.getParams().getMessage().cStr() << tv.tv_usec << std::endl;
        std::cout.flush();
        return kj::READY_NOW;
    }
};
```

Then you can just start receiving messages: (Note that I explicitly declared the type of variables instead of using `auto`-this is just for clarity)

```c++
void startClient(int fd)
{
    kj::UnixEventPort::captureSignal(SIGINT);
    auto ioContext = kj::setupAsyncIo();

    auto addrPromise = ioContext.provider->getNetwork().parseAddress("127.0.0.1", 2572)
    .then([](kj::Own<kj::NetworkAddress> addr) {
        std::cerr << "Got an address." << std::endl;
        return addr->connect().attach(kj::mv(addr));
    });

    auto stream = addrPromise.wait(ioContext.waitScope);
    
    capnp::TwoPartyVatNetwork network(*stream, capnp::rpc::twoparty::Side::CLIENT);

    auto rpcSystem = capnp::makeRpcClient(network);
    
    {
        capnp::MallocMessageBuilder message;
        auto hostId = message.getRoot<capnp::rpc::twoparty::VatId>();
        hostId.setSide(capnp::rpc::twoparty::Side::SERVER);
        
        corerc::Publisher<capnp::Text>::Client publisher = rpcSystem.bootstrap(hostId).castAs<corerc::Publisher<capnp::Text>>();

        std::cerr << "Creating Client" << std::endl;

        corerc::Subscriber<capnp::Text>::Client sub = corerc::Subscriber<capnp::Text>::Client(kj::heap<SubscriberImpl>());

        capnp::Request<corerc::Publisher<capnp::Text>::SubscribeParams, corerc::Publisher<capnp::Text>::SubscribeResults> request = publisher.subscribeRequest();

        request.setSubscriber(sub);

        capnp::RemotePromise<corerc::Publisher<capnp::Text>::SubscribeResults> subscriptionPromise = request.send();
        
        capnp::Response<corerc::Publisher<capnp::Text>::SubscribeResults> subscription = subscriptionPromise.wait(ioContext.waitScope);

        ioContext.unixEventPort.onSignal(SIGINT).wait(ioContext.waitScope);
    }
}
```

## Result

Sending side:
```
$ cargo run -- server 127.0.0.1:2572
    Finished dev [unoptimized + debuginfo] target(s) in 0.09s
     Running `target/debug/coreio-core server '127.0.0.1:2572'`
subscribe
subscription dropped
```

Receiving side:
```
./test_capnp client
Got an address.
Returning rpcSystem
Creating Client
Message From Publisher: system time is: SystemTime { tv_sec: 1535697490, tv_nsec: 241009000 }241318
Message From Publisher: system time is: SystemTime { tv_sec: 1535697490, tv_nsec: 743071000 }743327
Message From Publisher: system time is: SystemTime { tv_sec: 1535697491, tv_nsec: 245849000 }246111
Message From Publisher: system time is: SystemTime { tv_sec: 1535697491, tv_nsec: 748657000 }748890
Message From Publisher: system time is: SystemTime { tv_sec: 1535697492, tv_nsec: 251489000 }251758
Message From Publisher: system time is: SystemTime { tv_sec: 1535697492, tv_nsec: 756887000 }757151
Received SIGINT, dropping connection!
```