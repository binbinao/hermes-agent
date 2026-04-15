# Python 异步编程教程

## 官方文档

| 资源 | URL | 说明 |
|------|-----|------|
| **Python asyncio 官方文档** | https://docs.python.org/3/library/asyncio.html | 最权威，但较枯燥 |
| **Python 3.12+ What's New** | https://docs.python.org/3/whatsnew/3.12.html#asyncio | 新版本改进 |

## 最佳教程

| 教程 | URL | 特点 |
|------|-----|------|
| **Real Python - Async IO** | https://realpython.com/async-io-python/ | ⭐ 最佳入门教程，图文并茂 |
| **FastAPI 异步文档** | https://fastapi.tiangolo.com/async/ | 实战导向，FastAPI 用户必看 |
| **StackAbuse asyncio** | http://stackabuse.com/python-async-await-tutorial/ | 详细代码示例 |

---

## 核心概念速查

```python
# 1. 定义异步函数
import asyncio

async def hello():
    return "Hello!"

# 2. 运行协程
asyncio.run(hello())

# 3. 并发执行多个任务
async def main():
    task1 = asyncio.create_task(fetch_data(1))
    task2 = asyncio.create_task(fetch_data(2))
    results = await asyncio.gather(task1, task2)
    return results

# 4. 异步上下文管理器
async with aiohttp.ClientSession() as session:
    async with session.get('https://api.example.com') as response:
        data = await response.json()

# 5. 异步迭代器
async for item in async_generator():
    print(item)
```

---

## 学习路径

```
1. 了解协程 (Coroutine) 基础
   ↓
2. async/await 语法
   ↓
3. asyncio.run() 事件循环
   ↓
4. create_task() 并发任务
   ↓
5. asyncio.gather() 并发等待
   ↓
6. 异步上下文管理器 (async with)
   ↓
7. 实战: aiohttp 异步 HTTP 请求
```

---

## 推荐练习项目

| 项目 | 用途 |
|------|------|
| **异步爬虫** | 同时抓取多个页面 |
| **异步 API 服务器** | FastAPI/Quart |
| **异步数据库操作** | asyncpg, aiosqlite |
| **异步 WebSocket** | websockets 库 |

---

## 常见误区

| 误区 | 正确做法 |
|------|----------|
| ❌ 在同步函数中调用 async 函数 | ✅ 使用 `asyncio.run()` 或创建事件循环 |
| ❌ 以为 async 就是多线程 | ✅ 单线程，协作式并发 |
| ❌ 用 `requests` 库做异步 | ✅ 用 `aiohttp` 或 `httpx` |
| ❌ await 在循环中串行等待 | ✅ 用 `asyncio.gather()` 并行 |

---

## 实战示例：异步并发爬虫

### 项目目标

并发抓取多个 GitHub 仓库信息，对比同步 vs 异步性能差异。

### 完整代码

```python
#!/usr/bin/env python3
"""
异步爬虫实战：并发获取多个 GitHub 仓库信息
对比同步和异步实现的性能差异
"""

import asyncio
import aiohttp
import time
import json
from typing import List, Dict

# 要抓取的仓库列表
REPOS = [
    "microsoft/vscode",
    "facebook/react",
    "tensorflow/tensorflow",
    "python/cpython",
    "ansible/ansible",
    "docker/docker",
    "kubernetes/kubernetes",
    "golang/go",
]


# ============================================================
# 方法1：同步版本 (作为基准)
# ============================================================
def fetch_repo_sync(repo: str) -> Dict:
    """同步方式获取仓库信息"""
    import urllib.request
    import json

    url = f"https://api.github.com/repos/{repo}"
    req = urllib.request.Request(
        url,
        headers={"User-Agent": "Python-Async-Demo"}
    )

    with urllib.request.urlopen(req, timeout=10) as response:
        data = json.loads(response.read().decode())

    return {
        "name": data.get("full_name"),
        "stars": data.get("stargazers_count", 0),
        "forks": data.get("forks_count", 0),
        "language": data.get("language", "Unknown"),
        "description": data.get("description", "")[:50],
    }


def sync_version(repos: List[str]) -> float:
    """同步版本：顺序执行"""
    start = time.perf_counter()

    results = []
    for repo in repos:
        result = fetch_repo_sync(repo)
        results.append(result)

    elapsed = time.perf_counter() - start
    return elapsed, results


# ============================================================
# 方法2：异步版本 (高效并发)
# ============================================================
async def fetch_repo_async(session: aiohttp.ClientSession, repo: str) -> Dict:
    """异步方式获取仓库信息"""
    url = f"https://api.github.com/repos/{repo}"

    async with session.get(url, headers={"User-Agent": "Python-Async-Demo"}) as response:
        data = await response.json()

        return {
            "name": data.get("full_name"),
            "stars": data.get("stargazers_count", 0),
            "forks": data.get("forks_count", 0),
            "language": data.get("language", "Unknown"),
            "description": data.get("description", "")[:50],
        }


async def async_version(repos: List[str]) -> float:
    """异步版本：并发执行"""
    start = time.perf_counter()

    async with aiohttp.ClientSession() as session:
        # 创建所有任务
        tasks = [fetch_repo_async(session, repo) for repo in repos]
        # 并发等待所有任务完成
        results = await asyncio.gather(*tasks)

    elapsed = time.perf_counter() - start
    return elapsed, results


# ============================================================
# 方法3：带信号量的异步版本 (控制并发数)
# ============================================================
async def fetch_repo_with_semaphore(
    session: aiohttp.ClientSession,
    repo: str,
    semaphore: asyncio.Semaphore
) -> Dict:
    """带信号量控制的异步获取"""
    async with semaphore:
        return await fetch_repo_async(session, repo)


async def async_with_limit(repos: List[str], max_concurrent: int = 3) -> float:
    """限制并发数的异步版本"""
    start = time.perf_counter()

    semaphore = asyncio.Semaphore(max_concurrent)

    async with aiohttp.ClientSession() as session:
        tasks = [
            fetch_repo_with_semaphore(session, repo, semaphore)
            for repo in repos
        ]
        results = await asyncio.gather(*tasks)

    elapsed = time.perf_counter() - start
    return elapsed, results


# ============================================================
# 结果展示
# ============================================================
def print_results(title: str, elapsed: float, results: List[Dict]):
    """格式化打印结果"""
    print(f"\n{'='*60}")
    print(f" {title}")
    print(f"{'='*60}")
    print(f" 耗时: {elapsed:.2f} 秒")
    print(f" 数量: {len(results)} 个仓库")
    print(f" 速率: {len(results)/elapsed:.1f} 请求/秒")
    print("-" * 60)

    # 按 stars 排序显示
    sorted_results = sorted(results, key=lambda x: x["stars"], reverse=True)
    for i, repo in enumerate(sorted_results[:5], 1):
        print(f"  {i}. {repo['name']}")
        print(f"     ⭐ {repo['stars']:,} | 🍴 {repo['forks']:,} | {repo['language']}")


# ============================================================
# 主程序
# ============================================================
async def main():
    print(f"目标仓库: {len(REPOS)} 个")
    print(f"仓库列表: {', '.join(REPOS)}")

    # 1. 同步版本
    elapsed_sync, results_sync = sync_version(REPOS)
    print_results("同步版本 (顺序执行)", elapsed_sync, results_sync)

    # 2. 纯异步版本
    elapsed_async, results_async = await async_version(REPOS)
    print_results("异步版本 (无限制并发)", elapsed_async, results_async)

    # 3. 带限流的异步版本
    elapsed_limited, results_limited = await async_with_limit(REPOS, max_concurrent=3)
    print_results("异步版本 (限制并发=3)", elapsed_limited, results_limited)

    # 性能对比
    print(f"\n{'='*60}")
    print(" 性能对比")
    print(f"{'='*60}")
    print(f" 同步版本:        {elapsed_sync:.2f}s (基准)")
    print(f" 异步无限制:      {elapsed_async:.2f}s (加速 {elapsed_sync/elapsed_async:.1f}x)")
    print(f" 异步限制3并发:   {elapsed_limited:.2f}s (加速 {elapsed_sync/elapsed_limited:.1f}x)")


if __name__ == "__main__":
    asyncio.run(main())
```

### 运行结果示例

```
目标仓库: 8 个
仓库列表: microsoft/vscode, facebook/react, tensorflow/tensorflow, python/cpython, ansible/ansible, docker/docker, kubernetes/kubernetes, golang/go

============================================================
 同步版本 (顺序执行)
============================================================
 耗时: 8.45 秒
 数量: 8 个仓库
 速率: 0.9 请求/秒
------------------------------------------------------------
  1. microsoft/vscode
     ⭐ 155,280 | 🍴 28,340 | TypeScript
  2. kubernetes/kubernetes
     ⭐ 108,990 | 🍴 38,520 | Go
  3. docker/docker
     ⭐ 69,100 | 🍴 24,020 | Go
  ...

============================================================
 异步版本 (无限制并发)
============================================================
 耗时: 0.32 秒
 数量: 8 个仓库
 速率: 25.0 请求/秒
------------------------------------------------------------
  ...(同样的数据)

============================================================
 异步版本 (限制并发=3)
============================================================
 耗时: 1.85 秒
 数量: 8 个仓库
 速率: 4.3 请求/秒

============================================================
 性能对比
============================================================
 同步版本:        8.45s (基准)
 异步无限制:      0.32s (加速 26.4x)
 异步限制3并发:   1.85s (加速 4.6x)
```

### 关键代码解析

```python
# 1. 异步HTTP客户端
async with aiohttp.ClientSession() as session:
    async with session.get(url) as response:
        data = await response.json()

# 2. 并发执行多个任务
tasks = [fetch_repo_async(session, repo) for repo in repos]
results = await asyncio.gather(*tasks)

# 3. 控制并发数量
semaphore = asyncio.Semaphore(3)  # 最多同时3个请求
async with semaphore:
    # ... 执行任务

# 4. 超时处理
async with asyncio.timeout(10):  # 10秒超时
    await fetch_repo_async(session, repo)
```

### 依赖安装

```bash
pip install aiohttp
# 或
pip install httpx  # 同时支持同步和异步
```

### 实战要点

| 要点 | 说明 |
|------|------|
| **aiohttp vs requests** | aiohttp 是异步的，requests 是同步的 |
| **asyncio.gather** | 并发等待所有任务，收集结果 |
| **Semaphore** | 控制并发数，避免请求过于密集 |
| **asyncio.timeout** | 设置超时，避免无限等待 |
| **异常处理** | 用 try/except 包裹 await 调用 |

---

## 进阶主题

| 主题 | 资源 |
|------|------|
| **异步生成器** | `async for` 遍历异步迭代器 |
| **异步上下文管理器** | `async with` 管理资源 |
| **任务队列** | `asyncio.Queue` 生产者-消费者 |
| **事件循环嵌套** | `asyncio.run()` 嵌套调用 |
| **uvloop** | 用 libuv 加速事件循环 (C扩展) |
