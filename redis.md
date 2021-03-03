PUB/SUB

Redis 通过 PUBLISH 、 SUBSCRIBE 等命令实现了订阅与发布模式

struct redisServer {
    // ...
    dict *pubsub_channels;
    // ...
};
