
## 1. Script cho Bash - `spring_env.sh`
``` bash
#!/bin/bash

# ============================================
# Spring Boot Environment Setup - Bash
# Usage: source spring_env.sh
# ============================================

echo "📂 Đang nạp biến môi trường từ .env..."

# Đọc từng dòng trong file .env, bỏ qua comment và dòng trống
while IFS= read -r line; do
    # Bỏ qua dòng trống và comment (bắt đầu bằng #)
    if [[ -n "$line" && ! "$line" =~ ^# ]]; then
        # Tách key=value
        key=$(echo "$line" | cut -d '=' -f1)
        value=$(echo "$line" | cut -d '=' -f2-)
        # Export biến môi trường
        export "$key"="$value"
        echo "   ✅ $key=$value"
    fi
done < .env

echo "🚀 Sẵn sàng chạy: ./gradlew bootRun"
```

## 2. Script cho Fish - `spring_env.fish`
``` bash
#!/usr/bin/fish

# ============================================
# Spring Boot Environment Setup - Fish
# Usage: source spring_env.fish
# ============================================

echo "📂 Đang nạp biến môi trường từ .env..."

# Đọc từng dòng, bỏ qua comment và dòng trống
for line in (cat .env | string match -r "^[^#].+")
    set item (string split -m 1 "=" -- $line)
    set -gx $item[1] $item[2]
    echo "   ✅ $item[1]=$item[2]"
end

echo "🚀 Sẵn sàng chạy: ./gradlew bootRun"
```