# uv_callback

[![Build Status](https://travis-ci.org/litesync/uv_callback.svg?branch=master)](https://travis-ci.org/litesync/uv_callback)

Module to call functions on other libuv threads.

It is an alternative to uv_async with some differences:

 * It supports coalescing and non coalescing calls
 * It supports synchronous and asynchronous calls
 * It supports the transfer of an argument to the called function
 * It supports result notification callback


# Usage Examples


## Sending progress to the main thread

In this case the calls can and must coalesce to avoid flooding the event loop if the
work is running too fast.

The call coalescing is enabled using the UV_COALESCE constant.

### In the receiver thread

```C
uv_callback_t progress;

void * on_progress(uv_callback_t *handle, void *value) {
   printf("progress: %d\n", (int)value);
}

uv_callback_init(loop, &progress, on_progress, UV_COALESCE);
```

### In the sender thread

```C
uv_callback_fire(&progress, (void*)value, NULL);
```


## Sending allocated data that must be released

In this case the calls cannot coalesce because it would cause data loss and memory leaks.

So instead of UV_COALESCE it uses UV_DEFAULT.

### In the receiver thread

```C
uv_callback_t send_data;

void * on_data(uv_callback_t *handle, void *data) {
  do_something(data);
  free(data);
}

uv_callback_init(loop, &send_data, on_data, UV_DEFAULT);
```

### In the sender thread

```C
uv_callback_fire(&send_data, data, NULL);
```


## Firing the callback synchronously

In this case the thread firing the callback will wait until the function
called on the other thread returns.

It means it will not use the current thread event loop and it does not
even need to have one (it can be used in caller threads with no event loop
but the called thread needs to have one).

The last argument is the timeout in milliseconds.

If the called thread does not respond within the specified timeout time
the function returns `UV_ETIMEDOUT`.


### In the called thread

```C
uv_callback_t send_data;

void * on_data(uv_callback_t *handle, void *args) {
  int result = do_something(args);
  free(args);
  return (void*)result;
}

uv_callback_init(loop, &send_data, on_data, UV_DEFAULT);
```

### In the caller thread

```C
int result;
uv_callback_fire_sync(&send_data, args, (void*)&result, 1000);
```


## Firing the callback and getting the result asynchronously

In this case the thread firing the callback will receive the result in its
own callback when the function called on the other thread loop returns.

Note that there are 2 callback definitions here, one for each thread.

### In the called thread

```C
uv_callback_t send_data;

void * on_data(uv_callback_t *handle, void *data) {
  int result = do_something(data);
  free(data);
  return (void*)result;
}

uv_callback_init(loop, &send_data, on_data, UV_DEFAULT);
```

### In the calling thread

```C
uv_callback_t result_cb;

void * on_result(uv_callback_t *handle, void *result) {
  printf("The result is %d\n", (int)result);
}

uv_callback_init(loop, &result_cb, on_result, UV_DEFAULT);

uv_callback_fire(&send_data, data, &result_cb);
```

# License

MIT

# Contact

contact AT litereplica DOT io
