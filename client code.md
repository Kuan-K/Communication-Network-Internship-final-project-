### client code

```
# client.py
import socket
import time

# === 你的環境設定 ===
proxy_ip   = '20.89.64.170'   # 跑 proxy.exe 的那台主機
proxy_port = 5405             # proxy listening port

server_ip  = '20.2.91.148'    # Ubuntu server
ack_port   = 1234             # server 用來收 ACK 的 port

TOTAL = 100

# 給「資料」用的 socket（走 proxy）
data_sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)

# 給 ACK 用的 socket（直接跟 server 溝通）
ack_sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
ack_sock.bind(('', ack_port))       # 在 client 這台開 1234 等 ACK
ack_sock.settimeout(1.0)

print("Client: send HELLO to server for ACK address")
# 直接對 server:1234 丟 HELLO，讓 server 知道 client 的 IP/port
ack_sock.sendto(b'HELLO', (server_ip, ack_port))

for seq in range(1, TOTAL + 1):
    while True:
        send_ts = time.time()
        msg = f"{seq}|{send_ts:.6f}"   # 序號 + timestamp
        print(f"[SEND] packet {seq}")
        data_sock.sendto(msg.encode(), (proxy_ip, proxy_port))

        try:
            data, addr = ack_sock.recvfrom(1024)
            text = data.decode().strip()
            parts = text.split()

            if len(parts) == 2 and parts[0] == "ACK":
                ack_num = int(parts[1])
                if ack_num == seq:
                    print(f"   -> got ACK {ack_num}")
                    break    # 送下一個 seq
                else:
                    print(f"   -> got ACK {ack_num}, expect {seq}, keep waiting")
            else:
                print(f"   -> unknown msg: {text}")

        except socket.timeout:
            print(f"   -> timeout, resend {seq}")

print("Client finished.")
```
