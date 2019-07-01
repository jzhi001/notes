# Redis 2.2 rdb notes

## startLoading()

```c
void startLoading(FILE *fp)
```

fields involved:

server.loading
server.loading_start_time
server.loading_total_bytes

NOTE: if fp is not available, server.loading_total_bytes is set to 1

## loadingProgress()

set server.loading_loaded_bytes to current offset

## stopLoading()

server.loading = 0
