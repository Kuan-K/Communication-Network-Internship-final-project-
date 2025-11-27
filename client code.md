### client code

```
import socket
import time

proxy_ip   = '20.89.64.170'   # proxy
proxy_port = 5405

server_ip  = '20.2.91.148'    # server
ack_port   = 1234

TOTAL = 100

data_sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)

ack_sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
ack_sock.bind(('', ack_port))
ack_sock.settimeout(1.0)

# 先跟 server 在 1234 打招呼，讓它知道 client 的位址
ack_sock.sendto(b'HELLO', (server_ip, ack_port))

for seq in range(1, TOTAL + 1):

    first_send = True
    while True:
        send_ts = time.time()
        msg = f"{seq}|{send_ts:.6f}"

        # 只在這個 seq 第一次送的時候印出
        if first_send:
            print(f"[SEND] packet {seq}")
            first_send = False

        data_sock.sendto(msg.encode(), (proxy_ip, proxy_port))

        try:
            data, addr = ack_sock.recvfrom(1024)
            text = data.decode().strip()
            parts = text.split()

            if len(parts) == 2 and parts[0] == "ACK":
                ack_num = int(parts[1])
                if ack_num == seq:
                    # 拿到正確 ACK 才跳下一個 seq
                    break
                else:
                    # 拿到舊 ACK/別的 ACK，忽略再等
                    continue
        except socket.timeout:
            # timeout 就重送，但不多印任何訊息
            continue

print("Client done.")
```
