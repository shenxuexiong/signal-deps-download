name: download_readid_with_logs

on:
  workflow_dispatch:
    inputs:
      app_id:
        description: 'APKPure 上的应用 ID (例如 com.readid.ready)'
        required: true
        default: 'com.readid.ready'
      app_name:
        description: '下载后保存的文件名前缀'
        required: true
        default: 'readid_ready'

jobs:
  download:
    runs-on: ubuntu-latest
    steps:
      # 1. 清理环境
      - name: 🧹 强制清理环境
        run: |
          find . -mindepth 1 ! -name '.git' ! -path './.git/*' -exec rm -rf {} +
          echo "✅ 环境已清空"

      # 2. 下载网页并提取链接（加入详细日志）
      - name: 🔍 获取链接并记录日志
        id: get_link
        run: |
          APP_ID="${{ github.event.inputs.app_id }}"
          DETAIL_PAGE="https://apkpure.com/${APP_ID}/download"
          
          echo "📝 [日志] 正在访问详情页: $DETAIL_PAGE"
          
          # 下载网页源码
          curl -s -L -H "User-Agent: Mozilla/5.0" "$DETAIL_PAGE" -o page_source.html
          
          # 记录网页大小，确认是否下载成功
          PAGE_SIZE=$(stat -c%s page_source.html)
          echo "📝 [日志] 网页源码下载完成，文件大小: $PAGE_SIZE 字节"
          
          # --- 核心：尝试提取链接 ---
          # 尝试匹配 1: 包含 download 的链接
          REAL_URL=$(grep -oP '(?<=href=")[^"]*download[^"]*\.apk(?=")' page_source.html | head -n 1)
          
          # 尝试匹配 2: 包含 dl/apk 的链接
          if [ -z "$REAL_URL" ]; then
             echo "📝 [日志] 尝试匹配方案 B..."
             REAL_URL=$(grep -oP '(?<=href=")[^"]*dl/apk[^"]*' page_source.html | head -n 1)
          fi
          
          # 补全链接
          if [[ $REAL_URL == /* ]]; then
             REAL_URL="https://apkpure.com$REAL_URL"
          fi

          # --- 记录调试信息 ---
          echo "📝 [日志] 提取到的原始链接: $REAL_URL"
          
          # 把网页源码复制到输出目录，方便后续作为 Artifact 上传查看
          cp page_source.html debug_page_source.html

          echo "url=$REAL_URL" >> $GITHUB_OUTPUT

      # 3. 下载 APK
      - name: 📥 下载 APK 文件
        run: |
          mkdir -p readid_apk_dl
          
          REAL_URL="${{ steps.get_link.outputs.url }}"
          APP_NAME="${{ github.event.inputs.app_name }}"
          
          # 记录下载日志
          echo "📝 [日志] 准备下载 APK..."
          echo "📝 [日志] 目标链接: $REAL_URL"
          
          if [ -z "$REAL_URL" ]; then
            echo "❌ [错误日志] 未能提取到下载链接！请检查 debug_page_source.html"
            # 即使失败，也创建一个空文件占位，防止后续步骤报错
            touch readid_apk_dl/ERROR_NO_LINK_FOUND.txt
          else
            # 执行下载
            HTTP_CODE=$(curl -L -w "%{http_code}" -H "User-Agent: Mozilla/5.0" "$REAL_URL" -o "readid_apk_dl/${APP_NAME}.apk")
            
            if [ "$HTTP_CODE" == "200" ]; then
               echo "✅ [日志] 下载成功！HTTP状态码: $HTTP_CODE"
            else
               echo "⚠️ [日志] 下载可能失败，HTTP状态码: $HTTP_CODE"
            fi
          fi
          
          # 列出最终文件
          echo "📝 [日志] 最终目录内容:"
          ls -lh readid_apk_dl/

      # 4. 上传所有东西 (APK + 日志 + 网页源码)
      - name: 📤 上传 Artifact (包含日志)
        uses: actions/upload-artifact@v4
        with:
          name: ${{ github.event.inputs.app_name }}-result-with-logs
          # 这里我们上传整个目录，里面包含了 APK 和 调试用的网页源码
          path: |
            readid_apk_dl/
            debug_page_source.html
          retention-days: 5
