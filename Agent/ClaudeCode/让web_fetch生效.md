# 在env块写入以下代理配置

```json
    "HTTP_PROXY": "http://127.0.0.1:7890",
    "HTTPS_PROXY": "http://127.0.0.1:7890",
    "ALL_PROXY": "http://127.0.0.1:7890",
    "NO_PROXY": "localhost,127.0.0.1,::1"
```

端口号注意使用实际的，clash 7890是经典的端口数

# 在根节点块写入以下配置
```json
 "skipWebFetchPreflight": true,
```
