### server code

```
# server.py
import socket
import time

host      = '0.0.0.0'
data_port = 5405   # 經 proxy 來的資料
ack_port  = 1234   # 直接回 ACK 給 client
TOTAL     = 100

data_sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
data_sock.bind((host, data_port))

ack_sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
ack_sock.bind((host, ack_port))

print(f"Server listening on data:{data_port}, ack:{ack_port}")

print("Server: waiting HELLO from client on ACK port...")
hello_data, client_addr = ack_sock.recvfrom(1024)
print(f"Registered client {client_addr}, msg = {hello_data.decode().strip()}")

expected_seq = 1

first_time = None
last_time  = None
last_seq   = None

def print_summary():
    print("\n=== Server Summary ===")
    if first_time is None:
        print("No packet received.\n")
    else:
        total = last_time - first_time
        print(f"First packet at  : {first_time:.3f}")
        print(f"Last packet at   : {last_time:.3f}")
        print(f"Last packet name : packet {last_seq}")
        print(f"Total time       : {total:.3f} sec\n")

try:
    while expected_seq <= TOTAL:
        data, addr = data_sock.recvfrom(65535)
        now = time.time()

        text = data.decode().strip()
        parts = text.split()
        # 期待格式：packet <seq>
        if len(parts) != 2 or parts[0] != "packet":
            continue

        seq = int(parts[1])

        if seq == expected_seq:
            print(f"[RECV] packet {seq}")

            if first_time is None:
                first_time = now
            last_time = now
            last_seq  = seq

            ack_sock.sendto(f"ACK {seq}".encode(), client_addr)
            expected_seq += 1

        elif seq < expected_seq:
            # 重複封包：再回一次 ACK
            ack_sock.sendto(f"ACK {seq}".encode(), client_addr)

        else:
            # 比預期大的封包（理論上不會發生）
            ack_sock.sendto(f"ACK {expected_seq - 1}".encode(), client_addr)

    print_summary()

except KeyboardInterrupt:
    print_summary()

finally:
    data_sock.close()
    ack_sock.close()
```
