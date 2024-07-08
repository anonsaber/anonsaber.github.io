在广域网上提供的公共 DNS 实际上是有 QOS 限制的，并且由于中国大陆特殊的网络情况，经常容易获得被污染的 DNS 解析。因此，在学习和工作中，架设一个 DNS 服务器是很有必要的。

> SmartDNS 是一个运行在本地的 DNS 服务器，它接受来自本地客户端的 DNS 查询请求，然后从多个上游 DNS 服务器获取 DNS 查询结果，并将访问速度最快的结果返回给客户端，以此提高网络访问速度。 SmartDNS 同时支持指定特定域名 IP 地址，并高性匹配，可达到过滤广告的效果; 支持 DOT(DNS over TLS) 和 DOH(DNS over HTTPS)，更好的保护隐私。
> 

## 分流配置

首先使用下面的脚本生成针对于中国大陆域名列表的分流规则

```bash
#!/bin/bash

# Trap any error and perform cleanup
cleanup() {
    echo "Cleaning up..."
    rm -rf "$REPO_DIR"
}
trap cleanup EXIT

# Define variables for repository URL, directory and output file
REPO_URL="https://github.com/felixonmars/dnsmasq-china-list.git"
REPO_DIR="dnsmasq-china-list"
OUTPUT_FILE="/tmp/china.domain.smartdns.conf"

# Clone the repository with a depth of 1 to fetch only the latest commit
git clone --depth=1 "$REPO_URL" "$REPO_DIR" || exit 1

# Change into the cloned directory
cd "$REPO_DIR"

# Run make command with specified server
make SERVER=cn_network smartdns-domain-rules || exit 1

# Update smartdns.conf files and concatenate them into a single output file
# Use process substitution with cat to avoid creating intermediate files
cat <(sed '/-speed-check-mode ping,tcp:80 -nameserver cn_network/s//-nameserver cn_network/' *.smartdns.conf) > "$OUTPUT_FILE"

# Output success message with output file location
echo "SmartDNS configuration saved to $OUTPUT_FILE" 

# Cleanup
cd ..
rm -rf "$REPO_DIR"

# Check for the existence of the specified configuration file
if [ -f "/etc/smartdns/china.domain.smartdns.conf" ]; then
    echo "Verification complete: The configuration file exists."

    # Proceed to check the operational status of the smartdns service
    if systemctl is-active --quiet smartdns; then
        echo "Service status confirmed: smartdns is actively running."
        echo "Initiating configuration update from temporary file."
        cat /tmp/china.domain.smartdns.conf > /etc/smartdns/china.domain.smartdns.conf
        echo "Configuration file has been successfully updated."

        # Restart the smartdns service to apply new configuration settings
        echo "Restarting the smartdns service to enact changes."
        systemctl restart smartdns
        echo "Service restart complete. Process concluded successfully."
    else
        echo "Service check result: smartdns is not running. Update aborted."
    fi
else
    echo "File check error: Configuration file does not exist. Operation aborted."
fi
```

一切正常的话，你得到一个 `/etc/smartdns/china.domain.smartdns.conf` 配置文件。

然后配置 `/etc/smartdns/smartdns.conf` ，你需要根据具体情况对配置进行调整。其中 subnet 配置项是 EDNS 配置，需要对应的上游 DNS 服务器支持。

```
conf-file /etc/smartdns/china.domain.smartdns.conf

bind 0.0.0.0:5353
force-AAAA-SOA yes
cache-size 4096
prefetch-domain yes
serve-expired yes
serve-expired-ttl 0
speed-check-mode none
rr-ttl-min 60
rr-ttl-max 86400
log-level info

# ----- Bootstap DNS -----
server 119.29.29.29 -bootstrap-dns

# ----- Default Group -----
server-tls 1.1.1.1:853 -subnet 123.123.123.123/24
server-tls 8.8.8.8:853 -subnet 123.123.123.123/24
server-https https://1.1.1.1/dns-query -subnet 123.123.123.123/24

# ----- Domestic Group: cn_network -----
server 117.50.10.10 -group cn_network -exclude-default-group -subnet 123.123.123.123/24
server 119.29.29.29 -group cn_network -exclude-default-group -subnet 123.123.123.123/24
server-tls 1.12.12.12:853 -group cn_network -exclude-default-group -subnet 123.123.123.123/24
server-tls 223.5.5.5:853 -group cn_network -exclude-default-group -subnet 123.123.123.123/24
server-https https://1.12.12.12/dns-query -group cn_network -exclude-default-group -subnet 123.123.123.123/24
```

## 性能测试

使用 dnsperf 进行 dns 测试。

首先你需要安装 dnsperf，在 ubuntu 20.04/22.04 上可以通过 apt 进行安装。然后你需要准备一个文件用来测试，他里面应该有庞大的域名列表，类似于这样。

```
citysportsblog.com A
runningfromthereels.com A
manhattanrestaurantmonth.com A
homeremediesrelief.com A
articulate-online.com A
oblateyouthservice.com A
ultimacleaning.com A
isitrial.com A
siwaqq.com A
arborescent.com A
smarttouchcrm.com A
adsreceptors.com A
...
```

然后执行如下命令进行性能测试。

<aside>
💡 我使用的 CPU 是8核的 Intel(R) Xeon(R) CPU D-1521 @ 2.40GHz，并在虚拟机中使用了 2 个核心进行测试。

</aside>

```bash
dnsperf -p 5353 -s 127.0.0.1 -q 200 -d dnsperf-300w.txt -T 50000

DNS Performance Testing Tool
Version 2.9.0

[Status] Command line: dnsperf -p 5353 -s 127.0.0.1 -q 200 -d dnsperf-300w.txt -T 50000
[Status] Sending queries (to 127.0.0.1:5353)
[Status] Started at: Thu May  2 13:09:48 2024
[Status] Stopping after 1 run through file
[Status] Testing complete (end of file)

Statistics:

  Queries sent:         3000000
  Queries completed:    3000000 (100.00%)
  Queries lost:         0 (0.00%)

  Response codes:       NOERROR 2942011 (98.07%), SERVFAIL 31712 (1.06%), NXDOMAIN 26277 (0.88%)
  Average packet size:  request 28, response 62
  Run time (s):         100.762912
  Queries per second:   29772.859284

  Average Latency (s):  0.006694 (min 0.000023, max 2.400308)
  Latency StdDev (s):   0.025536
```

这个 QPS 差不多在 3w 左右，足以应对 1500 人左右的办公网络环境了。

<!-- ##{"timestamp":1678896000}## -->