เอกสารนี้จะพูดถึงหลักการปฎิบัติที่พึงประสงค์ในการสร้าง image ที่มีคุณภาพ

Docker สร้าง Images โดยอัตโนมัติจากการอ่านคำสั่งจาก `Dockerfile` -- ไฟล์ Text ที่บรรจุคำสั่งต่างๆตามลำดับ ในการสร้าง Image ขึ้นมา
`Dockerfile` มีรูปแบบและคำสั่งที่เฉพาะเจาะจง ซึ่งสามารถได้ที่ [Dockerfile reference](../../engine/reference/builder.md)

Docker image ประกอบไปด้วยชั้นที่อ่านได้เท่านั้น หลายชั้นรวมกัน ซึ่งแต่ละชั้นก็แสดงถึงคำสั่งแต่ละคำสั่ง
ชั้นเหล่านี้วางทับกันไปเรื่อยๆ และแต่ละชั้นก็เปลี่ยนแปลงไปเรื่อยๆจากชั้นก่อนหน้า ลองพิจารณา `Dockerfile` นี้

```dockerfile
FROM ubuntu:18.04
COPY . /app
RUN make /app
CMD python /app/app.py
```
แต่ละคำสั่งจะสร้างหนึ่งชั้นขึ้นมา
- `FROM` สร้างชั้นจาก ubuntu:18.04` Docker image
- `COPY` เพิ่มจากไฟล์โฟลเดอร์ปัจจุบันของ Docker Client
- `RUN` สร้างแอพพลิเคชั่นด้วยคำสั่ง `make`.
- `CMD` ระบุว่าจะสั่งคำสั่งอะไรใน container

ทุกครั้งที่ใช้งาน image และสร้าง container ขึ้นมา _ชั้นที่สามารถเขียนได้_ (ชั้นของ container) จะถูกสร้างขึ้นบนชั้นก่อนหน้า
การเปลี่ยนแปลงที่เกิดขึ้นบน container ที่ทำงานอยู่ เช่น การสร้างไฟล์ใหม่ การแก้ไขไฟล์ และลบไฟล์
ทั้งหมดที่กล่าวมาจะถูกเขียนลงบนชั้นนี้ 

ท่านสามารถเรียนรู้เพิ่มเติมเกี่ยวกับ layers (และการสร้างและเก็บ images) ได้ตามลิงค์ด้านล่าง
[About storage drivers](../../storage/storagedriver/index.md).

## ข้อแนะนำและข้อเสนอแนะพื้นฐาน

### สร้าง container ชั่วคราว

image ที่อธิบายด้วย `Dockerfile` ควรสร้าง Container ที่ชั่วคราวให้มากที่สุด
ชั่วคราวในที่นี้หมายถึง container สามารถถูกหยุด ทำลาย และสร้างใหม่ โดยใช้การติดตั้งให้น้อยที่สุดเท่าที่จะเป็นไปได้

อ่าน [Processes](https://12factor.net/processes) จากระเบียบวิธี _The Twelve-factor App_
เพื่อให้เข้าใจถึงจุดมุ่งหมายในการใช้งาน container ให้ไม่มีสถานะ

### เข้าใจ build context

เมื่อ`docker build`ถูกสั่ง directoryปัจจุบันคือ build context 
ตามค่าเริ่มต้น Dockerfile ควรจะอยู่ที่นี่ แต่ก็สามารถระบุได้ว่าอยู่ที่อื่นด้วยกสนใช้ (`-f`)
แต่ไม่ว่า Dockerfile จะอยู่ที่ไหน ไฟล์และ directories ต่างๆ ใน directory ปัจจุบันก็จะถูกส่งไปยัง
Docker daemon เป็น build context

> ตัวอย่าง build context
>
> สร้าง directory เพื่อใช้งานเป็น build context แล้ว `cd` เข้าไป
> เขียน "hello" ลงไปในไฟล์ hello
> สร้าง Dockerfile ที่รันคำสั่ง cat กับไฟล์ hello
> สร้าง image จาก  build context นี้  (`.`):
>
> ```shell
> mkdir myproject && cd myproject
> echo "hello" > hello
> echo -e "FROM busybox\nCOPY /hello /\nRUN cat /hello" > Dockerfile
> docker build -t helloapp:v1 .
> ```
>
> ย้าย Dockerfile และ hello ไปยัง directories ที่แตกต่างกัน และสร้าง image เวอร์ชั่นใหม่ (โดยไม่ใช่ cache จากการสร้างครั้งที่แล้ว)
> ใช้ `-f` เพื่อชั้ไปยัง Dockerfile และระบุ directory ของ build context: 
>
> ```shell
> mkdir -p dockerfiles context
> mv Dockerfile dockerfiles && mv hello context
> docker build --no-cache -t helloapp:v2 -f dockerfiles/Dockerfile context
> ```

การรวมไฟล์ที่ไม่ใช้งานใน build context จะทำให้ image ที่สร้างขึ้นมีขนาดใหญ่ขึ้น
ขนาดที่ใหญ่ขึ้นทำให้ใช้เวลาในการสร้าง image, เวลาในการ pull และ push และขนาด container ตอนทำงานมากขึ้นไปอีก
เพื่อดูว่า build context มีขนาดเท่าไร มองหาคำเหล่านี้เมื่อคุณ build จาก Dockerfile

```none
Sending build context to Docker daemon  187.8MB
```

### ส่ง Dockerfile ผ่าน `stdin`

Docker มีความสามารถในการสร้าง image จากการส่งต่อ Dockerfile ผ่าน `stdin`
ด้วย _build context บนเครื่อง หรือจากที่อื่น_ การส่ง Dockerfile ผ่าน `stdin`มีประโยชน์ในการ Build ครั้งเดียวที่ไม่ต้องการเขียน Dockerfile ลง disk
หรือในสถานการณ์อื่นๆที่หลังจาก build แล้วไม่ต้องการให้มี Dockerfile เหลืออยู่

> ตัวอย่างนี้มาจาก [เอกสารนี้](http://tldp.org/LDP/abs/html/here-docs.html) เพื่อความสะดวก
> แต่วิธีอื่นๆที่ให้ `Dockerfile` ผ่าน `stdin` ก็สามารถใช้ได้เช่นกัน
> 
> ดังเช่นข้างล่าง สองคำสั่งนี้เหมือนกัน:
> 
> ```bash
> echo -e 'FROM busybox\nRUN echo "hello world"' | docker build -
> ```
> 
> ```bash
> docker build -<<EOF
> FROM busybox
> RUN echo "hello world"
> EOF
> ```
> 
> ผู้ใช้สามารถเปลี่ยนวิธีจากตัวอย่างนี้ ให้เหมาะสมกับงานที่ทำ หรือแล้วแต่ถนัด


#### สร้าง image จาก Dockerfile ผ่าน stdin โดยไม่ส่ง build context

ใช้คำสั่งนี้ในการสร้าง image จาก Dockerfile ผ่าน stdin โดยไม่ส่งไฟล์อื่นๆเป็น build context
เครื่องหมาย ยัติภังค์ (`-`) ใช้เป็น `PATH` และบอก Dockerfile ว่าให้อ่าน build context (ที่มีแค่ Dockerfile)
จาก `stdin` เท่านั้น ไม่ต้องไปอ่านจาก directory:

```bash
docker build [OPTIONS] -
```

ตัวอย่างต่อไปโชว์การ build image จาก Dockerfile ที่ผ่าน `stdin` 
ไม่มีไฟล์อื่นถูกส่งเป็น build context ให้กับ docker daemon

```bash
docker build -t myimage:latest -<<EOF
FROM busybox
RUN echo "hello world"
EOF
```
การไม่ให้ build context มีประโยชน์ในสถานการณ์ที่ `Dockerfile` ไม่ต้องการที่จะ Copy ไฟล์ลงไปใน image
การทำเช่นนี้จะทำให้การ build เร็วขึ้น เนื่องจากไม่มีการส่งไฟล์ไปหา daemon

ถ้าผู้ใช้ต้องการทำให้ความเร็วในการ build มากขึ้น โดยการไม่ส่งบางไฟล์จาก build context ลองดูเอกสารนี้[exclude with .dockerignore](#exclude-with-dockerignore)

> **หมายเหตุ**: การ build ที่ใช้ตำสั่ง `COPY` หรือ `ADD` จะไม่สำเร็จ 
> ถ้าใช้คำสั่งตามแบบด้านบน ข้างล่างแสดงให้เห็นถึงตัวอย่างเหตุการณ์นี้:
> 
> ```bash
> # สร้าง directory ในการทำงาน
> mkdir example
> cd example
> 
> # สร้างไฟล์ตัวอย่าง
> touch somefile.txt
> 
> docker build -t myimage:latest -<<EOF
> FROM busybox
> COPY somefile.txt .
> RUN cat /somefile.txt
> EOF
> 
> # การ build จะไม่สำเร็จ
> ...
> Step 2/3 : COPY somefile.txt .
> COPY failed: stat /var/lib/docker/tmp/docker-builder249218248/somefile.txt: no such file or directory
> ```

#### Build จาก build context บนเครื่อง โดยใช้ Dockerfile จาก stdin

ใช้คำสั่งนี้ในการสร้าง image จากไฟล์บนเครื่อง แต่ใช้ `Dockerfile` จาก `stdin`
คำสั่งนี้ใช้ `-f` (หรือ `--file`) เพื่อระบุ `Dockerfile` ที่จะใช้โดยใช้ ยัติภังค์ (`-`) แทนชื่อของไฟล์
เพื่อให้อ่าน `Dockerfile` จาก `stdin`:

```bash
docker build [OPTIONS] -f- PATH
```

ตัวอย่างข้างล่างใช้ directory ปัจจุบัน (`.`) เป็น build context แล้วสร้าง image จาก `Dockerfile`
ที่มาจาก `stdin` ใช้[เอกสารนี้](http://tldp.org/LDP/abs/html/here-docs.html).


```bash
# สร้าง directory ในการทำงาน
mkdir example
cd example

# สร้างไฟล์ตัวอย่าง
touch somefile.txt

# สร้าง image โดยใช้ directory ปัจจุบันเป็น context และส่ง Dockerfile ผ่าน stdin
docker build -t myimage:latest -f- . <<EOF
FROM busybox
COPY somefile.txt .
RUN cat /somefile.txt
EOF
```

#### สร้างจาก context ทางไกล โดยใช้ Dockerfile จาก stdin}

ใช้คำสั่งนี้ในการสร้าง image จาก file ที่อยู่บน `git` repository โดยใช้ `Dockerfile` จาก `stdin`
ใช้ `-f` (หรือ `--file`) เพื่อระบุ `Dockerfile` ที่จะใช้โดยใช้ ยัติภังค์ (`-`) แทนชื่อของไฟล์
เพื่อให้อ่าน `Dockerfile` จาก `stdin`:

```bash
docker build [OPTIONS] -f- PATH
```

คำสั่งนี้มีประโยชน์ในการสร้าง image จาก repository ที่ไม่มี `Dockerfile`
หรือคุณต้องการสร้าง image ด้วย `Dockerfile` ที่คุณสร้างเอง

ตัวอย่างด้านล่างแสดงการ build image โดยใช้ `Dockerfile` จาก `stdin` และเพิ่มไฟล์ `hello.c` จาก ["hello-world" Git repository on GitHub](https://github.com/docker-library/hello-world).

```bash
docker build -t myimage:latest -f- https://github.com/docker-library/hello-world.git <<EOF
FROM busybox
COPY hello.c .
EOF
```

> **เบื้องหลังการทำงาน**
>
> ในทุกการสร้าง image จาก remote repository, docker จะทำการ `git clone` repository นั้นๆบนเครื่อง
> และส่งไฟล์เลห่านั้นเป็น build context ให้กับ daemon
> เนื่องจากการทำงานเป็นแบบนี้ทำให้บนเครื่องต้องลง `git`