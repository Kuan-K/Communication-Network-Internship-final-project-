### server code

```
import socket
import time

host      = '0.0.0.0'
data_port = 5405
ack_port  = 1234
TOTAL     = 100

data_sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
data_sock.bind((host, data_port))

ack_sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
ack_sock.bind((host, ack_port))

print(f"Server listening on data:{data_port}, ack:{ack_port}")

# 等 HELLO 決定要回 ACK 到哪裡
hello, client_addr = ack_sock.recvfrom(1024)

expected_seq = 1

while expected_seq <= TOTAL:
    data, addr = data_sock.recvfrom(65535)
    now = time.time()

    msg = data.decode().strip()
    try:
        seq_str, ts_str = msg.split("|")
        seq = int(seq_str)
        send_ts = float(ts_str)
    except ValueError:
        continue

    # Stop-and-wait 的情況理論上 seq 不會超前 expected_seq
    if seq == expected_seq:
        # 只對「第一次收到的正確下一個封包」印出
        print(f"[RECV] packet {seq}")
        ack_sock.sendto(f"ACK {seq}".encode(), client_addr)
        expected_seq += 1
    elif seq < expected_seq:
        # 重複封包：回 ACK 但不印出
        ack_sock.sendto(f"ACK {seq}".encode(), client_addr)
    else:
        # 理論上不會發生；如果發生也先回目前已確認的 seq-1
        ack_sock.sendto(f"ACK {expected_seq - 1}".encode(), client_addr)

print("Server got all packets 1 ~ 100 in order.")
```
