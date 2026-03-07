# 使用盒子优选ip并上传到github


前提是在盒子中安装好git 和测速软件
https://github.com/XIU2/CloudflareSpeedTest

然后使用如下脚本即可测速，并且把IP上传到github

```
#!/bin/bash

# 1. 进入工作目录
cd /opt/data/cloudflarespeedtest || { echo "Directory not found"; exit 1; }

# 定义日志函数
log() {
    local message="$(date '+%Y-%m-%d %H:%M:%S') - $1"
    echo "$message"
}

csv_file="result.csv"
output_ip_file="youxuanip.txt"

log "Start Cloudflare speed test (SL: 3MB, TL: 400ms)..."

# --- 修正测速命令：删除重复 -url，保留您的参数 ---
./cfst -url https://cdn.cloudflare.steamstatic.com/steam/apps/256843155/movie_max.mp4 -sl 3 -tl 500 -dn 10

# 检查测速结果
if [ ! -s "$csv_file" ]; then
    log "Error: Speed test failed, $csv_file is empty."
    exit 1
fi

log "Test finished. Extracting all 10 best IPs..."

# --- 核心修改：提取全部 10 个优选 IP（跳过第一行表头） ---
# tail -n +2 表示从第2行开始读取到最后
best_ips=$(tail -n +2 "$csv_file" | cut -d',' -f1)

if [ -n "$best_ips" ]; then
    # 将所有 IP 写入本地文件（覆盖模式）
    echo "$best_ips" > "$output_ip_file"
	chmod +x "$output_ip_file"
    log "SUCCESS: 10 IPs saved to $output_ip_file"

    # --- 核心补充：自动上传到 GitHub ---
    log "Starting GitHub auto-push..."
    
    # 强制忽略证书报错（防止 OpenWrt 报错）
    git config --global http.sslVerify false
    
    # 确保只添加这一个文件
    git add "$output_ip_file"
    
    # 提交更改，附带当前时间
    git commit -m "Auto Update IPs: $(date '+%Y-%m-%d %H:%M:%S')"
    
    # 推送到远程 master 分支
    # 注意：运行此脚本前必须先执行下方的 git remote set-url 命令
    if git push origin master -f; then
        log "GitHub Sync Success! 🚀"
    else
        log "Error: GitHub Push Failed! ❌"
    fi
else
    log "Error: Could not extract IPs from $csv_file"
    exit 1
fi

```


下次准备把乌鸦大佬的区域IP拿出来给自己测速使用

https://ddx.snu.cc/TW

https://ddx.snu.cc/HK

https://ddx.snu.cc/JP

https://ddx.snu.cc/KR

https://ddx.snu.cc/SG

https://ddx.snu.cc/US
