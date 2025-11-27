### client code

```
# client.py
import socket
import time

# === 依你的實際 IP 改這三個 ===
proxy_ip   = '20.89.64.170'   # proxy 機器
proxy_port = 5405

server_ip  = '20.2.91.148'    # server 機器
ack_port   = 1234             # server 用來收 ACK 的 port

TOTAL = 100

# 給 data 用的 socket（走 proxy）
data_sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)

# 給 ACK 用的 socket（直接跟 server 通）
ack_sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
ack_sock.bind(('', ack_port))
ack_sock.settimeout(1.0)

print("Client: send HELLO to server for ACK address")
ack_sock.sendto(b'HELLO', (server_ip, ack_port))

# 統計用
total_tx        = 0       # 送出 UDP 封包總數（含重送）
retransmissions = 0       # 重送次數
rtts            = []      # 每個封包成功那次傳輸的 RTT

for seq in range(1, TOTAL + 1):
    first_send = True

    while True:
        t0 = time.time()
        msg = f"packet {seq}"

        if first_send:
            print(f"[SEND] packet {seq}")
            first_send = False

        data_sock.sendto(msg.encode(), (proxy_ip, proxy_port))
        total_tx += 1

        try:
            data, addr = ack_sock.recvfrom(1024)
            text = data.decode().strip()
            parts = text.split()

            # 期待 ACK <num>
            if len(parts) == 2 and parts[0] == "ACK":
                ack_num = int(parts[1])
                if ack_num == seq:
                    # 收到對的 ACK，記錄 RTT
                    rtt = time.time() - t0
                    rtts.append(rtt)
                    break      # 換下一個 seq
                else:
                    # 舊的 ACK，忽略
                    continue

        except socket.timeout:
            # timeout -> 視為一次重送
            retransmissions += 1
            continue

print("Client finished sending.")

# ====== Client Summary：drop rate + 平均 delay (RTT) ======
print("\n=== Client Summary ===")
print(f"Total packets        : {TOTAL}")
print(f"Total transmissions  : {total_tx}")
print(f"Retransmissions      : {retransmissions}")

if total_tx > 0:
    drop_rate = (total_tx - TOTAL) / total_tx * 100.0
    # 這個 drop_rate 代表「有多少比例的傳送需要重送（因為丟包或太晚到）」
    print(f"Drop/timeout rate    : {drop_rate:.1f} %")
else:
    print("Drop/timeout rate    : N/A")

if rtts:
    avg_rtt = sum(rtts) / len(rtts)
    max_rtt = max(rtts)
    print(f"Average RTT          : {avg_rtt:.3f} sec")
    print(f"Max RTT              : {max_rtt:.3f} sec")
else:
    print("Average RTT          : N/A")

data_sock.close()
ack_sock.close()
```
