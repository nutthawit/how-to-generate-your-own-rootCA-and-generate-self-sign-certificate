## ขั้นตอนปกติในการ Request SSL Certificate จาก CA

* โดยปกติการขอ (request) SSL certificate จาก CA (Certificate Authority) เช่น Verisign หรือ GoDaddy ผู้ขอจะต้องส่ง Certificate Sign Request (CSR) ไปให้กับองค์กร CA (Verisign หรือ GoDaddy) แล้วทางองค์กรจะเซ็นใบรับรอง ด้วยไฟล์ root certificate และ private key ของพวกเขากลับมาให้เรา

* โดยปกติแล้วองค์กร CA ที่เป็นที่รู้จัก (CA สากล) และยอมรับจาก Browser ต่างๆ จะมี root CA ของพวกเขาฝังอยู่ใน Browser เพื่อให้ Browser สามารถตรวจสอบว่า Certificate ของ Web server ต่างๆ ถูกเซ็นรับรองอย่างถูกต้องจากองค์กรที่เชื่อถือได้

* ดังนั้นจึงเป็นเหตุผลว่าทำไม self-signed certificate จะไม่ถูกเชื่อถือจาก Browser เพราะว่ามันไม่ได้ถูกเซ็นรับรองจากองค์กร CA ที่เป็นสากล

* แต่ก็ยังมีวิธีที่ทำให้ Browser เชื่อถือ certificate ที่เราสร้างขึ้นมาได้นั้นคือ การที่เราตั้งตัวเองเป็น CA โดยการสร้าง root certificate และ private key ของตัวเองแล้วนำ root certificate ที่เราสร้างไปไว้ที่ Brower หรือ อุปกรณ์ต่างๆ ที่จะเชื่อมต่อเข้า Web server ของเรา แล้วทำการสร้างไฟล์ Certificate Sign Request ที่ระบุโดเมนของ Web site ที่ต้องการจะเซ็นใบรับรอง (sign) และนำไปเซ็นรับรองด้วยไฟล์ root certificate และ private key ที่สร้างขึ้นมา

## การตั้งตัวเป็น CA

1. อันดับแรกสร้าง private key ธรรมดาขึ้นมาหนึ่งอัน

```bash
openssl genrsa -des3 -out rootCA.key 2048
```

#### *หากไม่ต้องการใส่ passphrase ให้ตัดออพชั่น -des3 ออก แต่แนะให้ใส่ไว้เพื่อป้องกันกรณีที่มีคนอื่นได้ private key ของเราไปและต้องการจะนำไปใช้ต่อ*

2. สร้าง root certificate ด้วยคำสั่งดังนี้

```bash
openssl req -x509 -new -nodes -key rootCA.key -sha256 -days 365 -out rootCA.crt
```

เมื่อรันคำสั่งแล้วจะมี prompt ขึ้นมาให้ใส่รหัสผ่านของ private key หลังจากนั้นให้ระบุข้อมูลต่างของ certificate เช่น Country Name, Oraganization Name, Common Name (Server FQDN) และอื่นๆ เป็นต้น

หลังจากเสร็จขั้นตอนที่ 1 และ 2 แล้ว ตอนนี้คุณจะได้ไฟล์มาทั้งหมด 2 ไฟล์ คือ rootCA.key (ไฟล์ private key) และ rootCA.crt (ไฟล์ root certificate) นั้นหมายความว่าตอนนี้คุณได้ตั้งองค์กร CA เล็กๆของคุณเองขึ้นมาแล้ว

3. นำ root certificate ไปติดตั้งที่ Browser หรือ อุปกรณ์ต่างๆ ตามขั้นตอนของ Brower หรือ อุปกรณ์นั้นๆ

## สร้าง CA-signed certificate สำหรับแต่ละเว็บไซต์

1. สร้าง private key ขึ้นมาหนึ่งอัน

```bash
openssl genrsa -out git.wannasin.org.key 2048
```

#### *git.wannasin.org เป็นชื่อโดเมนของเว็บไซต์ที่สมมติขึ้นมา ให้เปลี่ยนเป็นโดเมนตามที่คูณต้องการ*

2. สร้างไฟล์ certificate sign request (CSR) จาก private key

```bash
openssl req -new -key git.wannasin.org.key -out git.wannasin.org.csr
```

เมื่อรันคำสั่งแล้วให้ระบุข้อมูลต่างๆของเว็บไซต์ เช่น Country Name, Oraganization Name, Common Name (Server FQDN) และอื่นๆ

3. สร้างไฟล์ certificate (.crt) จากไฟล์ CSR ที่เราสร้างข้างต้น โดยการสร้างไฟล์ขึ้นมาไฟล์นึงชื่อ git.wannasin.org.ext และใส่เนื้อหาดังนี้

```ini
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
subjectAltName = @alt_names

[alt_names]
DNS.1 = git.wannasin.org
```

3. สร้างไฟล์ certificate (.crt) โดยรันคำสั่งดังนี้

```bash
openssl x509 -req -in git.wannasin.org.csr -CA rootCA.crt -CAkey rootCA.key -CAcreateserial -out git.wannasin.org.crt -days 7 -sha256 -extfile git.wannasin.org.ext
```

หลังจากรันคำสั่งจะได้ไฟล์มาทั้งหมด 3 ไฟล์ คือ git.wannasin.org.csr.key (private key), git.wannasin.org.csr.csr (certificate signing request) และ git.wannasin.org.csr.crt (signed certificate).

**ณ ตอนนี้คุณสามารถนำไฟล์ private key และไฟล์ certificate ที่ได้มาไปวางที่ Web server เพื่อทำให้เว็บไซต์คุณเป็น HTTPS ได้แล้ว**

**หลังจากนี้หากคุณมีเว็บไซต์ใหม่และต้องการสร้าง certificate ให้ทำซ้ำในหัวข้อ 'สร้าง CA-signed certificate สำหรับแต่ละเว็บไซต์' ได้เลย โดยไม่จำเป็นต้องสร้าง rootCA ใหม่**


