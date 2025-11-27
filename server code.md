### server code

```
# server.py
import socket
import time

host      = '0.0.0.0'
data_port = 5405   # 經 proxy 送來的資料
ack_port  = 1234   # 直接與 client 溝通的 port
TOTAL     = 100

# 收 data 的 socket（proxy 會轉到這裡）
data_sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
data_sock.bind((host, data_port))

# 收 / 送 ACK 的 socket
ack_sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
ack_sock.bind((host, ack_port))

print(f"Server listening on data:{data_port}, ack:{ack_port}")

# === 1. 先等 client 在 ACK port 丟 HELLO，記住 client 位址 ===
print("Server: waiting HELLO from client on ACK port...")
hello_data, client_addr = ack_sock.recvfrom(1024)
print(f"Registered client {client_addr}, msg = {hello_data.decode().strip()}")

expected_seq = 1

try:
    while expected_seq <= TOTAL:
        data, addr = data_sock.recvfrom(65535)   # 從 proxy 來
        now = time.time()

        msg = data.decode().strip()
        try:
            seq_str, ts_str = msg.split("|")
            seq = int(seq_str)
            send_ts = float(ts_str)
        except ValueError:
            print(f"[RECV] bad format: {msg}")
            continue

        delay = now - send_ts
        print(f"[RECV] packet {seq:3d}, delay={delay:.3f}s")

        if seq == expected_seq:
            # 正確的下一個封包
            print(f"   -> ACCEPT {seq}")
            ack_sock.sendto(f"ACK {seq}".encode(), client_addr)
            expected_seq += 1

        elif seq < expected_seq:
            # 重複封包（舊的重送）
            print(f"   -> DUP {seq}, already have")
            ack_sock.sendto(f"ACK {seq}".encode(), client_addr)

        else:  # seq > expected_seq
            # 比預期大的封包（理論上不會發生，因為一次只送一個）
            print(f"   -> OUT OF ORDER {seq}, expect {expected_seq}")
            ack_sock.sendto(f"ACK {expected_seq - 1}".encode(), client_addr)

    print("Server got all packets 1 ~ 100 in order!")

except KeyboardInterrupt:
    print("Server interrupted.")

finally:
    data_sock.close()
    ack_sock.close()
```
