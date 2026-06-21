掠胺颐裁握


Python网络编程与Socket通信

一、Socket基础

import socket

# TCP Socket
tcp_sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
# UDP Socket
udp_sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)


二、TCP服务器

import socket, threading

class Server:
    def __init__(self, host='0.0.0.0', port=8080):
        self.sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        self.sock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
        self.sock.bind((host, port))
        self.sock.listen(5)

    def start(self):
        while True:
            conn, addr = self.sock.accept()
            threading.Thread(target=self.handle, args=(conn, addr)).start()

    def handle(self, conn, addr):
        with conn:
            while True:
                data = conn.recv(4096)
                if not data: break
                conn.send(data)  # echo


class Client:
    def __init__(self, host='localhost', port=8080):
        self.sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        self.sock.connect((host, port))
    def send(self, msg):
        self.sock.send(msg.encode())
        return self.sock.recv(4096).decode()


三、消息协议

import struct, json

class Protocol:
    HEADER_SIZE = 4  # 4字节长度头

    @staticmethod
    def encode(data):
        body = json.dumps(data).encode()
        header = struct.pack('!I', len(body))
        return header + body

    @staticmethod
    def decode(conn):
        header = conn.recv(4)
        if not header: return None
        length = struct.unpack('!I', header)[0]
        body = b''
        while len(body) < length:
            chunk = conn.recv(length - len(body))
            if not chunk: return None
            body += chunk
        return json.loads(body.decode())


四、UDP通信

class UDPServer:
    def __init__(self, host='0.0.0.0', port=8888):
        self.sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
        self.sock.bind((host, port))
    def run(self):
        while True:
            data, addr = self.sock.recvfrom(65535)
            self.sock.sendto(b"ACK", addr)

class UDPClient:
    def __init__(self):
        self.sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
        self.sock.settimeout(5.0)


五、selectors多路复用

import selectors

class SelectorServer:
    def __init__(self, host='0.0.0.0', port=8080):
        self.sel = selectors.DefaultSelector()
        self.server = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        self.server.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
        self.server.setblocking(False)
        self.server.bind((host, port))
        self.server.listen(100)
        self.sel.register(self.server, selectors.EVENT_READ, self._accept)

    def run(self):
        while True:
            events = self.sel.select()
            for key, mask in events:
                key.data(key.fileobj, mask)

    def _accept(self, sock, mask):
        conn, addr = sock.accept()
        conn.setblocking(False)
        self.sel.register(conn, selectors.EVENT_READ, self._read)

    def _read(self, conn, mask):
        data = conn.recv(4096)
        if data:
            conn.send(data)
        else:
            self.sel.unregister(conn)
            conn.close()


六、HTTP客户端实现

class HTTPClient:
    def request(self, method, url, headers=None, body=None):
        from urllib.parse import urlparse
        parsed = urlparse(url)
        sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        sock.connect((parsed.hostname, parsed.port or 80))

        request_line = f"{method} {parsed.path or '/'} HTTP/1.1\r\n"
        default_hdrs = {'Host': parsed.hostname, 'Connection': 'close'}
        header_str = ''.join(f"{k}: {v}\r\n" for k,v in default_hdrs.items())
        sock.send((request_line + header_str + "\r\n").encode())
        # 接收响应...


七、常用工具

def get_local_ip():
    s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
    s.connect(('8.8.8.8', 80))
    return s.getsockname()[0]

def port_scan(host, start, end):
    open_ports = []
    for port in range(start, end+1):
        s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        s.settimeout(0.5)
        if s.connect_ex((host, port)) == 0:
            open_ports.append(port)
        s.close()
    return open_ports

总结：TCP适合可靠传输，UDP适合实时通信。生产环境使用selectors或asyncio处理并发。注意消息边界问题。

诒翁覆湃嚷诚芬巴壮滩仆驮拾秦侥

m.evw.msfsx.cn/57511.Doc
m.evw.msfsx.cn/35539.Doc
m.evw.msfsx.cn/66808.Doc
m.evw.msfsx.cn/9.Doc
m.evw.msfsx.cn/28044.Doc
m.evq.msfsx.cn/71355.Doc
m.evq.msfsx.cn/44208.Doc
m.evq.msfsx.cn/51173.Doc
m.evq.msfsx.cn/33555.Doc
m.evq.msfsx.cn/13733.Doc
m.evq.msfsx.cn/75755.Doc
m.evq.msfsx.cn/55991.Doc
m.evq.msfsx.cn/53195.Doc
m.evq.msfsx.cn/73597.Doc
m.evq.msfsx.cn/95159.Doc
m.evq.msfsx.cn/37177.Doc
m.evq.msfsx.cn/33917.Doc
m.evq.msfsx.cn/33935.Doc
m.evq.msfsx.cn/99593.Doc
m.evq.msfsx.cn/31953.Doc
m.evq.msfsx.cn/71915.Doc
m.evq.msfsx.cn/57951.Doc
m.evq.msfsx.cn/20002.Doc
m.evq.msfsx.cn/99157.Doc
m.evq.msfsx.cn/68006.Doc
m.ecm.msfsx.cn/33753.Doc
m.ecm.msfsx.cn/91939.Doc
m.ecm.msfsx.cn/19955.Doc
m.ecm.msfsx.cn/04400.Doc
m.ecm.msfsx.cn/80400.Doc
m.ecm.msfsx.cn/86248.Doc
m.ecm.msfsx.cn/19779.Doc
m.ecm.msfsx.cn/64628.Doc
m.ecm.msfsx.cn/79537.Doc
m.ecm.msfsx.cn/97395.Doc
m.ecm.msfsx.cn/71739.Doc
m.ecm.msfsx.cn/77133.Doc
m.ecm.msfsx.cn/99311.Doc
m.ecm.msfsx.cn/77777.Doc
m.ecm.msfsx.cn/04006.Doc
m.ecm.msfsx.cn/77791.Doc
m.ecm.msfsx.cn/44044.Doc
m.ecm.msfsx.cn/86802.Doc
m.ecm.msfsx.cn/97775.Doc
m.ecm.msfsx.cn/68848.Doc
m.ecn.msfsx.cn/08088.Doc
m.ecn.msfsx.cn/68482.Doc
m.ecn.msfsx.cn/75135.Doc
m.ecn.msfsx.cn/73153.Doc
m.ecn.msfsx.cn/71711.Doc
m.ecn.msfsx.cn/79111.Doc
m.ecn.msfsx.cn/99177.Doc
m.ecn.msfsx.cn/08624.Doc
m.ecn.msfsx.cn/20048.Doc
m.ecn.msfsx.cn/06488.Doc
m.ecn.msfsx.cn/39551.Doc
m.ecn.msfsx.cn/57597.Doc
m.ecn.msfsx.cn/53537.Doc
m.ecn.msfsx.cn/73973.Doc
m.ecn.msfsx.cn/97519.Doc
m.ecn.msfsx.cn/60008.Doc
m.ecn.msfsx.cn/33595.Doc
m.ecn.msfsx.cn/39593.Doc
m.ecn.msfsx.cn/80640.Doc
m.ecn.msfsx.cn/62402.Doc
m.ecb.msfsx.cn/48688.Doc
m.ecb.msfsx.cn/22208.Doc
m.ecb.msfsx.cn/66884.Doc
m.ecb.msfsx.cn/20028.Doc
m.ecb.msfsx.cn/82028.Doc
m.ecb.msfsx.cn/93371.Doc
m.ecb.msfsx.cn/35359.Doc
m.ecb.msfsx.cn/15179.Doc
m.ecb.msfsx.cn/95599.Doc
m.ecb.msfsx.cn/17153.Doc
m.ecb.msfsx.cn/97171.Doc
m.ecb.msfsx.cn/19775.Doc
m.ecb.msfsx.cn/59339.Doc
m.ecb.msfsx.cn/95535.Doc
m.ecb.msfsx.cn/19393.Doc
m.ecb.msfsx.cn/93317.Doc
m.ecb.msfsx.cn/37331.Doc
m.ecb.msfsx.cn/93713.Doc
m.ecb.msfsx.cn/46228.Doc
m.ecb.msfsx.cn/33551.Doc
m.ecv.msfsx.cn/08606.Doc
m.ecv.msfsx.cn/86628.Doc
m.ecv.msfsx.cn/71377.Doc
m.ecv.msfsx.cn/48662.Doc
m.ecv.msfsx.cn/22808.Doc
m.ecv.msfsx.cn/80206.Doc
m.ecv.msfsx.cn/15931.Doc
m.ecv.msfsx.cn/00026.Doc
m.ecv.msfsx.cn/59793.Doc
m.ecv.msfsx.cn/64646.Doc
m.ecv.msfsx.cn/06648.Doc
m.ecv.msfsx.cn/88624.Doc
m.ecv.msfsx.cn/95551.Doc
m.ecv.msfsx.cn/42682.Doc
m.ecv.msfsx.cn/77573.Doc
m.ecv.msfsx.cn/95519.Doc
m.ecv.msfsx.cn/08662.Doc
m.ecv.msfsx.cn/00206.Doc
m.ecv.msfsx.cn/35735.Doc
m.ecv.msfsx.cn/35751.Doc
m.ecc.msfsx.cn/62822.Doc
m.ecc.msfsx.cn/62284.Doc
m.ecc.msfsx.cn/59715.Doc
m.ecc.msfsx.cn/99975.Doc
m.ecc.msfsx.cn/08800.Doc
m.ecc.msfsx.cn/59513.Doc
m.ecc.msfsx.cn/39915.Doc
m.ecc.msfsx.cn/37733.Doc
m.ecc.msfsx.cn/28222.Doc
m.ecc.msfsx.cn/59951.Doc
m.ecc.msfsx.cn/22220.Doc
m.ecc.msfsx.cn/75353.Doc
m.ecc.msfsx.cn/28604.Doc
m.ecc.msfsx.cn/95557.Doc
m.ecc.msfsx.cn/60226.Doc
m.ecc.msfsx.cn/99571.Doc
m.ecc.msfsx.cn/46040.Doc
m.ecc.msfsx.cn/91179.Doc
m.ecc.msfsx.cn/53371.Doc
m.ecc.msfsx.cn/35775.Doc
m.ecx.msfsx.cn/20880.Doc
m.ecx.msfsx.cn/44644.Doc
m.ecx.msfsx.cn/35717.Doc
m.ecx.msfsx.cn/77337.Doc
m.ecx.msfsx.cn/95199.Doc
m.ecx.msfsx.cn/02624.Doc
m.ecx.msfsx.cn/28048.Doc
m.ecx.msfsx.cn/15571.Doc
m.ecx.msfsx.cn/15153.Doc
m.ecx.msfsx.cn/79913.Doc
m.ecx.msfsx.cn/64606.Doc
m.ecx.msfsx.cn/91573.Doc
m.ecx.msfsx.cn/84260.Doc
m.ecx.msfsx.cn/60882.Doc
m.ecx.msfsx.cn/88060.Doc
m.ecx.msfsx.cn/73531.Doc
m.ecx.msfsx.cn/79531.Doc
m.ecx.msfsx.cn/84228.Doc
m.ecx.msfsx.cn/20426.Doc
m.ecx.msfsx.cn/24020.Doc
m.ecz.msfsx.cn/53399.Doc
m.ecz.msfsx.cn/33193.Doc
m.ecz.msfsx.cn/79157.Doc
m.ecz.msfsx.cn/77951.Doc
m.ecz.msfsx.cn/42820.Doc
m.ecz.msfsx.cn/48606.Doc
m.ecz.msfsx.cn/71157.Doc
m.ecz.msfsx.cn/73919.Doc
m.ecz.msfsx.cn/73139.Doc
m.ecz.msfsx.cn/95115.Doc
m.ecz.msfsx.cn/40668.Doc
m.ecz.msfsx.cn/00262.Doc
m.ecz.msfsx.cn/15313.Doc
m.ecz.msfsx.cn/51557.Doc
m.ecz.msfsx.cn/24422.Doc
m.ecz.msfsx.cn/40004.Doc
m.ecz.msfsx.cn/82002.Doc
m.ecz.msfsx.cn/80002.Doc
m.ecz.msfsx.cn/66440.Doc
m.ecz.msfsx.cn/44086.Doc
m.ecl.msfsx.cn/80402.Doc
m.ecl.msfsx.cn/97917.Doc
m.ecl.msfsx.cn/91971.Doc
m.ecl.msfsx.cn/77173.Doc
m.ecl.msfsx.cn/64004.Doc
m.ecl.msfsx.cn/06420.Doc
m.ecl.msfsx.cn/64404.Doc
m.ecl.msfsx.cn/51573.Doc
m.ecl.msfsx.cn/84806.Doc
m.ecl.msfsx.cn/24042.Doc
m.ecl.msfsx.cn/28880.Doc
m.ecl.msfsx.cn/80262.Doc
m.ecl.msfsx.cn/48406.Doc
m.ecl.msfsx.cn/20428.Doc
m.ecl.msfsx.cn/20806.Doc
m.ecl.msfsx.cn/46200.Doc
m.ecl.msfsx.cn/00604.Doc
m.ecl.msfsx.cn/53777.Doc
m.ecl.msfsx.cn/00028.Doc
m.ecl.msfsx.cn/88684.Doc
m.eck.msfsx.cn/20600.Doc
m.eck.msfsx.cn/62422.Doc
m.eck.msfsx.cn/68240.Doc
m.eck.msfsx.cn/48646.Doc
m.eck.msfsx.cn/59791.Doc
m.eck.msfsx.cn/31957.Doc
m.eck.msfsx.cn/91759.Doc
m.eck.msfsx.cn/79757.Doc
m.eck.msfsx.cn/97553.Doc
m.eck.msfsx.cn/22820.Doc
m.eck.msfsx.cn/53517.Doc
m.eck.msfsx.cn/99755.Doc
m.eck.msfsx.cn/28022.Doc
m.eck.msfsx.cn/71597.Doc
m.eck.msfsx.cn/93119.Doc
m.eck.msfsx.cn/71359.Doc
m.eck.msfsx.cn/66084.Doc
m.eck.msfsx.cn/00468.Doc
m.eck.msfsx.cn/60046.Doc
m.eck.msfsx.cn/62046.Doc
m.ecj.msfsx.cn/46888.Doc
m.ecj.msfsx.cn/84860.Doc
m.ecj.msfsx.cn/59979.Doc
m.ecj.msfsx.cn/62806.Doc
m.ecj.msfsx.cn/40668.Doc
m.ecj.msfsx.cn/06422.Doc
m.ecj.msfsx.cn/55717.Doc
m.ecj.msfsx.cn/35551.Doc
m.ecj.msfsx.cn/99159.Doc
m.ecj.msfsx.cn/77197.Doc
m.ecj.msfsx.cn/73793.Doc
m.ecj.msfsx.cn/97939.Doc
m.ecj.msfsx.cn/88668.Doc
m.ecj.msfsx.cn/86226.Doc
m.ecj.msfsx.cn/22600.Doc
m.ecj.msfsx.cn/60860.Doc
m.ecj.msfsx.cn/59317.Doc
m.ecj.msfsx.cn/31339.Doc
m.ecj.msfsx.cn/75713.Doc
m.ecj.msfsx.cn/84080.Doc
m.ech.msfsx.cn/26440.Doc
m.ech.msfsx.cn/51351.Doc
m.ech.msfsx.cn/55975.Doc
m.ech.msfsx.cn/02046.Doc
m.ech.msfsx.cn/62862.Doc
m.ech.msfsx.cn/66666.Doc
m.ech.msfsx.cn/91939.Doc
m.ech.msfsx.cn/59593.Doc
m.ech.msfsx.cn/99595.Doc
m.ech.msfsx.cn/77377.Doc
m.ech.msfsx.cn/99119.Doc
m.ech.msfsx.cn/17933.Doc
m.ech.msfsx.cn/95975.Doc
m.ech.msfsx.cn/57955.Doc
m.ech.msfsx.cn/75531.Doc
m.ech.msfsx.cn/11717.Doc
m.ech.msfsx.cn/68224.Doc
m.ech.msfsx.cn/64664.Doc
m.ech.msfsx.cn/00868.Doc
m.ech.msfsx.cn/44880.Doc
m.ecg.msfsx.cn/99517.Doc
m.ecg.msfsx.cn/02002.Doc
m.ecg.msfsx.cn/35575.Doc
m.ecg.msfsx.cn/06882.Doc
m.ecg.msfsx.cn/04646.Doc
m.ecg.msfsx.cn/93517.Doc
m.ecg.msfsx.cn/44048.Doc
m.ecg.msfsx.cn/06268.Doc
m.ecg.msfsx.cn/88822.Doc
m.ecg.msfsx.cn/26626.Doc
m.ecg.msfsx.cn/26828.Doc
m.ecg.msfsx.cn/46862.Doc
m.ecg.msfsx.cn/02800.Doc
m.ecg.msfsx.cn/19797.Doc
m.ecg.msfsx.cn/73773.Doc
m.ecg.msfsx.cn/06628.Doc
m.ecg.msfsx.cn/17119.Doc
m.ecg.msfsx.cn/33737.Doc
m.ecg.msfsx.cn/20064.Doc
m.ecg.msfsx.cn/88466.Doc
m.ecf.msfsx.cn/62844.Doc
m.ecf.msfsx.cn/68866.Doc
m.ecf.msfsx.cn/11151.Doc
m.ecf.msfsx.cn/46024.Doc
m.ecf.msfsx.cn/53397.Doc
m.ecf.msfsx.cn/37577.Doc
m.ecf.msfsx.cn/19117.Doc
m.ecf.msfsx.cn/15599.Doc
m.ecf.msfsx.cn/02262.Doc
m.ecf.msfsx.cn/59177.Doc
m.ecf.msfsx.cn/17777.Doc
m.ecf.msfsx.cn/59171.Doc
m.ecf.msfsx.cn/97959.Doc
m.ecf.msfsx.cn/31933.Doc
m.ecf.msfsx.cn/39715.Doc
m.ecf.msfsx.cn/97995.Doc
m.ecf.msfsx.cn/53535.Doc
m.ecf.msfsx.cn/06428.Doc
m.ecf.msfsx.cn/33553.Doc
m.ecf.msfsx.cn/86244.Doc
m.ecd.msfsx.cn/19513.Doc
m.ecd.msfsx.cn/11913.Doc
m.ecd.msfsx.cn/46084.Doc
m.ecd.msfsx.cn/57997.Doc
m.ecd.msfsx.cn/06408.Doc
m.ecd.msfsx.cn/97537.Doc
m.ecd.msfsx.cn/13591.Doc
m.ecd.msfsx.cn/62844.Doc
m.ecd.msfsx.cn/17311.Doc
m.ecd.msfsx.cn/66468.Doc
m.ecd.msfsx.cn/53735.Doc
m.ecd.msfsx.cn/71971.Doc
m.ecd.msfsx.cn/17975.Doc
m.ecd.msfsx.cn/79153.Doc
m.ecd.msfsx.cn/35735.Doc
m.ecd.msfsx.cn/40668.Doc
m.ecd.msfsx.cn/73353.Doc
m.ecd.msfsx.cn/79773.Doc
m.ecd.msfsx.cn/57533.Doc
m.ecd.msfsx.cn/80864.Doc
m.ecs.msfsx.cn/39731.Doc
m.ecs.msfsx.cn/73373.Doc
m.ecs.msfsx.cn/55959.Doc
m.ecs.msfsx.cn/91571.Doc
m.ecs.msfsx.cn/77759.Doc
m.ecs.msfsx.cn/79371.Doc
m.ecs.msfsx.cn/93393.Doc
m.ecs.msfsx.cn/20466.Doc
m.ecs.msfsx.cn/37593.Doc
m.ecs.msfsx.cn/35993.Doc
m.ecs.msfsx.cn/95357.Doc
m.ecs.msfsx.cn/79313.Doc
m.ecs.msfsx.cn/33777.Doc
m.ecs.msfsx.cn/17111.Doc
m.ecs.msfsx.cn/15397.Doc
m.ecs.msfsx.cn/62668.Doc
m.ecs.msfsx.cn/97151.Doc
m.ecs.msfsx.cn/57793.Doc
m.ecs.msfsx.cn/53139.Doc
m.ecs.msfsx.cn/95553.Doc
m.eca.msfsx.cn/04088.Doc
m.eca.msfsx.cn/31999.Doc
m.eca.msfsx.cn/17719.Doc
m.eca.msfsx.cn/55577.Doc
m.eca.msfsx.cn/35977.Doc
m.eca.msfsx.cn/40226.Doc
m.eca.msfsx.cn/82284.Doc
m.eca.msfsx.cn/66460.Doc
m.eca.msfsx.cn/80846.Doc
m.eca.msfsx.cn/17719.Doc
m.eca.msfsx.cn/37191.Doc
m.eca.msfsx.cn/39715.Doc
m.eca.msfsx.cn/02224.Doc
m.eca.msfsx.cn/62484.Doc
m.eca.msfsx.cn/02280.Doc
m.eca.msfsx.cn/17115.Doc
m.eca.msfsx.cn/11337.Doc
m.eca.msfsx.cn/40622.Doc
m.eca.msfsx.cn/77191.Doc
m.eca.msfsx.cn/53319.Doc
m.ecp.msfsx.cn/39977.Doc
m.ecp.msfsx.cn/91391.Doc
m.ecp.msfsx.cn/42800.Doc
m.ecp.msfsx.cn/02446.Doc
m.ecp.msfsx.cn/51377.Doc
m.ecp.msfsx.cn/11711.Doc
m.ecp.msfsx.cn/57973.Doc
m.ecp.msfsx.cn/59915.Doc
m.ecp.msfsx.cn/11377.Doc
m.ecp.msfsx.cn/75793.Doc
m.ecp.msfsx.cn/71119.Doc
m.ecp.msfsx.cn/77779.Doc
m.ecp.msfsx.cn/37735.Doc
m.ecp.msfsx.cn/93319.Doc
m.ecp.msfsx.cn/44880.Doc
m.ecp.msfsx.cn/13135.Doc
m.ecp.msfsx.cn/37797.Doc
m.ecp.msfsx.cn/91775.Doc
m.ecp.msfsx.cn/53337.Doc
m.ecp.msfsx.cn/77957.Doc
m.eco.msfsx.cn/48228.Doc
m.eco.msfsx.cn/37935.Doc
m.eco.msfsx.cn/80646.Doc
m.eco.msfsx.cn/35757.Doc
m.eco.msfsx.cn/15195.Doc
m.eco.msfsx.cn/73755.Doc
m.eco.msfsx.cn/86202.Doc
m.eco.msfsx.cn/71357.Doc
m.eco.msfsx.cn/77391.Doc
m.eco.msfsx.cn/57513.Doc
m.eco.msfsx.cn/15311.Doc
m.eco.msfsx.cn/02024.Doc
m.eco.msfsx.cn/55979.Doc
m.eco.msfsx.cn/62622.Doc
m.eco.msfsx.cn/11375.Doc
m.eco.msfsx.cn/79159.Doc
m.eco.msfsx.cn/73353.Doc
m.eco.msfsx.cn/75919.Doc
m.eco.msfsx.cn/17993.Doc
m.eco.msfsx.cn/77395.Doc
m.eci.msfsx.cn/97797.Doc
m.eci.msfsx.cn/75557.Doc
m.eci.msfsx.cn/55777.Doc
m.eci.msfsx.cn/31775.Doc
m.eci.msfsx.cn/82286.Doc
m.eci.msfsx.cn/28626.Doc
m.eci.msfsx.cn/22868.Doc
m.eci.msfsx.cn/00042.Doc
m.eci.msfsx.cn/82486.Doc
m.eci.msfsx.cn/26262.Doc
