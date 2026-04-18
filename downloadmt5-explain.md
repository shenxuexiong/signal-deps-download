# GitHub Actions 工作流缓存问题解释

## 问题原因

GitHub Actions 的 runner（运行器）会在同一台机器上重复使用，每次 workflow run 之间 **工作目录不会自动清空**。

旧项目 (`signal-deps-download`) 中：
1. 最初运行下载了 Signal 库文件（libsignal-android、libsignal-client 等）到 `deps/` 目录
2. 后来修改 workflow 想下载 mt5，虽然加了 `rm -rf deps`，但 runner 可能在执行前已经加载了旧的工作目录状态
3. 结果上传的 artifact 仍然包含旧的 Signal 文件（显示 358MB）

## 为什么新项目成功

新项目 (`downloadmt5`) 从未运行过 Signal 下载工作流：
1. runner 环境是完全干净的
2. 只下载了 mt5.exe（4.9MB）
3. artifact 只包含一个文件

## 正确做法

如果要在旧项目下载新文件，必须确保：

```yaml
steps:
  - name: Download
    run: |
      # 使用全新的目录名
      rm -rf newdownload
      mkdir -p newdownload
      curl -L "URL" -o newdownload/file.exe
```

或者创建**新的 workflow 文件**，确保没有历史缓存污染。

## 经验总结

1. **每个不同的下载任务用独立的 workflow 文件**，避免混淆
2. **每个下载任务用独立的目录名**，避免残留文件混入
3. GitHub Actions 的 artifact 和 runner 缓存是分开的——artifact 不会影响后续 run，但 runner 工作目录会保留文件
