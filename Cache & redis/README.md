# Redis
### Redis in an in-memory database which uses advanced data structures and algorithms to store and retrieve data efficiently. It is very costly as it stores in RAM, that's why its not an alternative to databases. Also Redis can store data into disk while shut down and retrieve those in reboot unlike a regular program which losses its resources after process is terminated. 

### Intro
- Redis keys and values are strings
- It can store up to 2^32 keys, same for different data structures in redis (i.e. lists, sets, etc.)
- Maximum size for a key/value is 512 MB

## Commands
https://redis.io/docs/latest/commands/
#### Get & Set
- `SET key value` - Set the value of a key
- `GET key` - Get the value of a key. Result: `"value"`
- `SET a b` - Set the value of a key a to b. 
- `GET a` - Get the value of a key a. Result: `"b"`
- `SET name "Shartaz Feeham"` - This time we are using double quotes to store a string with spaces
- `SET Serial_Feeham 10` - Set the value of a key Serial_Feeham to 10
- `GET Serial_Feeham` - Get the value of a key Serial_Feeham. Result: `"10"` -> The value is converted into string.
- `EXISTS key` - Check if a key exists. Result: `1) (integer) 1` -> 1 means the key exists, 0 means it doesn't.

#### Batch
- `keys *` - Get all keys. Result: `1) "a" 2) "Serial_Feeham" 3) "name"`
- `keys a*` - Get all keys starting with a. Result: `1) "a"` -> REGEX is supported.
- `scan 0` - returns with pagination and a keyword that can be used to get the next page.

#### Remove
- `DEL key` - Delete a key. Note: RegEX is not allowed in delete commands. 
- `DEL a b c` - Delete multiple keys. 
- `flushdb` - Delete all currently stored keys. 

#### Expiring 
- `SET key value EX 10` - Set the value of a key with an expiration time of 10 seconds. **Note: EX sets in seconds, PX sets in milliseconds.**
- `SET key value exat 19654343666` expire at a specific time (Unix/epoch timestamp).
- `TTL key` - Get the expiration time of a key. Result: `1) (integer) 10`
- `EXPIRE key 60` - Set the expiration time of a key to 60 seconds. **Note: This will override(does not sum/subtract) the previous expiration time.**
- `SET key value2 keepttl` - Set the value of a key and keep the previous expiration time, without the keepttl option, the expiration time will be erased and the key will never expire/deleted.

#### Updates
- `SET key value NX` - Set the value of a key only if the key does not exist.
- `SET key value XX` - Set the value of a key only if the key already exists.
- `SET key value2 keepttl` - Set the value of a key and keep the previous expiration time, without the keepttl option, the expiration time will be erased and the key will never expire/deleted.
- `incr key` - Increment the value of a key by 1. Note: The value must be an integer, also if the key does not exist, it will be created with the value of 1 (meaning that incremented from 0 to 1).
- `decr key` - Decrement the value of a key by 1. Note: The value must be an integer, also if the key does not exist, it will be created with the value of -1 (meaning that decremented from 0 to -1).
- `incrby key 10` - Increment the value of a key by 10. Note: The value must be an integer.
- `incrbyfloat key 1.5` - Increment the value of a key by 1.5. Note: The value must be a float.

### A simple rate limiting scenario using redis
Let's assume that we have a recommendation microservice that performs heavy calculation and recommends. And we have
another external service what we use as a pay per call service. Now the requirement is we want to limit the number of 
calls to the services to 10 calls per minute. This is how we can do it:
1. `set callCount 10 ex 6000`
2. A hit to redis: `get callCount` result: 10, so we can call the service. Then apply this: `decr callCount`
3. Then 8 more hits in the same way. 
4. Then in the last hit: result: 1, so we can call the service.
5. LIMITING: After the last hit the counter will turn 0, and we will not be able to call the service. 
6. When the time expires, we need to reset the counter to 10.

- ### Redis supports many data structures such as lists, sets, hashes, stack, queue, priority queue, etc.
- ### Transactions (using multi command) are supported in Redis. It supports (pessimistic) locking by using watch command.

