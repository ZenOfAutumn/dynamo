# `dynamo.common.http`

HTTP 取数客户端。提供一个与具体后端无关的门面（facade）`fetch_bytes` / `close_http_client`，建立在 `HttpClient` 这一抽象基类（ABC）之上，包含两个具体子类：`AiohttpClient`（默认）与 `HttpxClient`。
后端选择：`DYN_HTTP_BACKEND={aiohttp,httpx}`。

## 为什么默认选用 aiohttp

在高并发场景下（例如一次请求扇出到 100 个图片 URL），httpx 后端会触发 `httpx.PoolTimeout`。在相同负载下，aiohttp 的扩展性显著更好——参见
[NeMo Gym aiohttp vs httpx 笔记](https://docs.nvidia.com/nemo/gym/latest/infrastructure/engineering-notes/aiohttp-vs-httpx.html)
与 [openai-python#1596](https://github.com/openai/openai-python/issues/1596)。

根本原因在于 httpx 的连接池实现。其在
[`httpcore._async.connection_pool`](https://github.com/encode/httpcore/blob/master/httpcore/_async/connection_pool.py#L303-L309)
中的维护例程“每当向连接池添加或移除一个请求时”都会触发，且每次调用复杂度为 `O(queue_size × pool_size)`，因此随着积压请求数量增长开销呈二次方上升。aiohttp 的连接器（connector）原生以 `O(1)` 入队，这也是为什么我们的 httpx 后端必须在连接池前加一个进程级信号量（`DYN_HTTP_CONCURRENCY`），以避免 `PoolTimeout` 向上层泄漏。在 aiohttp 下该信号量是多余的。

### 基准测试

500 rps × 1 万次请求，按服务端处理耗时分桶（数值越低越好，所有延迟单位为 ms）：

```
== request_rate=500 rps  requests=10000 ==

[sweep] mean_ms=50  order=['aiohttp', 'httpx']
backend  wall(s)     avg     p50     p90     p99
httpx       20.1    75.7    51.0   158.7   241.1
aiohttp     20.1    50.7    50.6    50.8    51.1

[sweep] mean_ms=100  order=['httpx', 'aiohttp']
httpx       23.2  1550.4  1484.3  2430.2  3164.9
aiohttp     20.1   101.4   100.6   100.9   101.2

[sweep] mean_ms=200  order=['aiohttp', 'httpx']
httpx       43.2 11833.2 11762.6 21167.1 23025.2
aiohttp     20.8   320.5   231.2   733.3   830.3

[sweep] mean_ms=300  order=['httpx', 'aiohttp']
httpx       62.2 21229.8 21318.0 37834.0 41728.4
aiohttp     32.2  6355.0  6395.1 11119.6 12071.7
```

在饱和点以下（`mean_ms=50`）两个后端表现等价。一旦超过饱和点，httpx 出现超线性退化，而 aiohttp 仍然保持接近设定速率的水平。

复现命令：

```bash
python -m benchmarks.multimodal.http.sweep \
    --server-processing-time-means-ms 50,100,200,300 \
    --request-rate 500 \
    --requests 10000 \
    --timeout 60
```

## 运维可调参数

完整的 `DYN_HTTP_*` 环境变量 / `--http-*` CLI 参数（连接池大小、单次调用超时覆盖、httpx 专用并发信号量、aiohttp keepalive 等）请参见
[`http_args.py`](../configuration/groups/http_args.py)。历史遗留的 `DYN_MM_HTTP_*` 环境变量仍然受支持。
