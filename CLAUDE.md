# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

这是一个用于下载被墙/受限网站内容的 GitHub Actions 工作流项目。当前配置用于下载 Binance Authenticator APK。

## 主要文件

- `download.yml` - 主工作流文件，触发后从 Binance 下载 authenticator APK 并上传为 artifact
- `.github/workflows/blank.yml` - 备用工作流文件

## 工作流使用

```bash
# 本地无法运行 GitHub Actions，需要在 GitHub 网页端触发
# 或者使用 gh cli:
gh workflow run download.yml
```

## 架构说明

- 使用 GitHub Actions `workflow_dispatch` 触发器，允许手动执行
- 下载依赖到 `deps/` 目录
- 通过 `actions/upload-artifact@v4` 上传，保留 7 天
- 目标 URL: `https://download.binance.com/authenticator/binance-authenticator.apk`

## 注意事项

- 这是一个极简项目，仅包含 GitHub Actions 工作流配置
- 无需构建、测试或 lint 命令
- 所有操作通过 GitHub Actions 执行
