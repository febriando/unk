
#!/usr/bin/env bash

# ==========================================
# Server Stats - Basic Linux Server Monitoring
# ==========================================

set -u

# Colors
GREEN='\033[0;32m'
CYAN='\033[0;36m'
YELLOW='\033[1;33m'
RESET='\033[0m'

print_header() {
    echo -e "\n${CYAN}=========================================="
    echo -e " $1"
    echo -e "==========================================${RESET}"
}

print_header "SERVER PERFORMANCE STATISTICS"

# ------------------------------------------
# OS INFORMATION
# ------------------------------------------
print_header "SYSTEM INFORMATION"

if [[ -f /etc/os-release ]]; then
    source /etc/os-release
    echo "OS         : ${PRETTY_NAME:-Unknown}"
fi

echo "Hostname   : $(hostname)"
echo "Uptime     : $(uptime -p 2>/dev/null || uptime)"
echo "Date       : $(date)"

# ------------------------------------------
# CPU USAGE
# ------------------------------------------
print_header "CPU USAGE"

read_cpu() {
    awk '/^cpu / {
        idle=$5
        total=0
        for(i=2;i<=NF;i++) total+=$i
        print total, idle
    }' /proc/stat
}

read -r total1 idle1 < <(read_cpu)

sleep 1

read -r total2 idle2 < <(read_cpu)

total_diff=$((total2 - total1))
idle_diff=$((idle2 - idle1))

if (( total_diff > 0 )); then
    cpu_usage=$(awk -v total="$total_diff" -v idle="$idle_diff" \
        'BEGIN { printf "%.2f", (1 - idle/total) * 100 }')
else
    cpu_usage="0.00"
fi

echo -e "Total CPU Usage: ${YELLOW}${cpu_usage}%${RESET}"

# ------------------------------------------
# MEMORY USAGE
# ------------------------------------------
print_header "MEMORY USAGE"

awk '
/MemTotal:/     { total=$2 }
/MemAvailable:/ { available=$2 }
END {
    used=total-available
    percentage=(used/total)*100

    printf "Total     : %.2f GB\n", total/1024/1024
    printf "Used      : %.2f GB\n", used/1024/1024
    printf "Available : %.2f GB\n", available/1024/1024
    printf "Usage     : %.2f%%\n", percentage
}' /proc/meminfo

# ------------------------------------------
# DISK USAGE
# ------------------------------------------
print_header "DISK USAGE"

df -hP -x tmpfs -x devtmpfs

# ------------------------------------------
# TOP 5 PROCESSES BY CPU
# ------------------------------------------
print_header "TOP 5 PROCESSES BY CPU"

ps -eo pid,comm,%cpu,%mem --sort=-%cpu | head -n 6

# ------------------------------------------
# TOP 5 PROCESSES BY MEMORY
# ------------------------------------------
print_header "TOP 5 PROCESSES BY MEMORY"

ps -eo pid,comm,%mem,%cpu --sort=-%mem | head -n 6

# ------------------------------------------
# LOAD AVERAGE
# ------------------------------------------
print_header "LOAD AVERAGE"

uptime | awk -F'load average:' '{print $2}'

# ------------------------------------------
# LOGGED IN USERS
# ------------------------------------------
print_header "LOGGED IN USERS"

who

# ------------------------------------------
# FAILED LOGIN ATTEMPTS
# ------------------------------------------
print_header "FAILED LOGIN ATTEMPTS"

if command -v lastb >/dev/null 2>&1; then
    if [[ -r /var/log/btmp ]]; then
        lastb 2>/dev/null | head -n 10
    else
        echo "Permission denied: run as root to view failed login records."
    fi
else
    echo "lastb is not available on this system."
fi

print_header "REPORT COMPLETE"

# for task https://roadmap.sh/projects/server-stats
