# Cài đặt và cấu hình trình biên dịch chéo QT cho Raspberry Pi
Dựa theo bài hướng dẫn ở [đây](https://mechatronicsblog.com/cross-compile-and-deploy-qt-5-12-for-raspberry-pi/)

## Bước phụ 1 : Cài bản Ubuntu
* Tải và burn **BootUSB** (Bằng **rufus** trên **windows** hoặc **Starup Disk Creator** trên **ubuntu**)
* Sửa lỗi Boot (nếu có):
	* Chạy Ubuntu trên BootUSB
	* Kết nối mạng
	* Nếu không boot được vào Ubuntu, cần cài [Boot Repair](https://www.howtogeek.com/114884/how-to-repair-grub2-when-ubuntu-wont-boot/)
```sh
	sudo apt-add-repository ppa:yannubuntu/boot-repair
	sudo apt-get update
	sudo apt-get install -y boot-repair
	boot-repair 	
```
Chọn **Recommended Repair** 
	
## Bước phụ 2 : Cài Raspbian bản full cho Pi
* Tải về từ [trang chủ](https://www.raspberrypi.org/downloads/raspbian/)
* Ghi ra thẻ nhớ bằng phần mềm **balena Etcher** (Thẻ từ **Class10** và **8GB** trở lên)
* Mở thư mục trên thẻ nhớ, thêm file **ssh** vào (Kích hoạt SSH cho Pi)
* Cắm thẻ nhớ và cấp nguồn cho Pi

## Bước phụ 3 : Thiết lập và truy cập vào Pi từ máy tính (Co)
```sh
sudo nano /etc/hosts
```
Thiết lập ip của Pi hiện tại thành một tên miền tùy ý, dễ nhớ

Ví dụ: 

```
192.168.1.24  piboard.com
```

* truy cập vào Pi qua SSH
```sh
ssh pi@piboard.com 
```
* Thiết lập hệ thống local và bật các ngoại vi nếu cần thiết
```sh
sudo raspi-config
```
* Khởi động lại Pi
```sh
sudo reboot
```
Tổng quan các bước cài Qt Cross
------------
**1. Cài đặt các thư viện phát triển – [Pi]**
**2. Chuẩn bị thư mục đích – [Pi]**
**3. Tạo thư mục làm việc và thiết lập toolchain – [Co]**
**4. Tạo và cấu hình sysroot – [Co]**
**5. Tải về Qt – [Co]**
**6. Cấu hình Qt Everywhere biên dịch chéo cho Pi  – [Co]**
**7. Make và make install – [Co]**
**8. Thiết lập Qt Creator để biên dịch chéo cho Pi – [Co]**

[Pi]: Làm trên Raspberry Pi

[Co]: Làm trên máy tính chạy Ubuntu 

Các bước cài 
------------
## 1. Cài đặt các thư viện phát triển – [Pi]
```sh
sudo nano /etc/apt/sources.list
```
```
deb http://raspbian.raspberrypi.org/raspbian/ buster main contrib non-free rpi
# Uncomment line below then 'apt-get update' to enable 'apt-get source'
deb-src http://raspbian.raspberrypi.org/raspbian/ buster main contrib non-free rpi
```
```sh
sudo apt-get update
sudo apt-get build-dep qt4-x11
sudo apt-get build-dep libqt5gui5
sudo apt-get install libudev-dev libinput-dev libts-dev libxcb-xinerama0-dev libxcb-xinerama0
```

## 2. Chuẩn bị thư mục đích – [Pi]
```sh
sudo mkdir /usr/local/qt5pi
sudo chown pi:pi /usr/local/qt5pi
sudo chown -R pi:pi /opt
sudo chmod -R 775 /usr/lib/cups/backend/
```

## 3. Tạo thư mục làm việc và thiết lập toolchain – [Co]
```sh
mkdir ~/raspi
cd ~/raspi
sudo apt-get install git -y
git clone https://github.com/raspberrypi/tools
```

## 4 . Tạo và cấu hình sysroot – [Co]

* Trong thư mục ~/raspi:
```sh
mkdir sysroot sysroot/usr sysroot/opt
```

* Đồng bộ
```sh
rsync -avz pi@piboard.com:/lib sysroot
rsync -avz pi@piboard.com:/usr/include sysroot/usr
rsync -avz pi@piboard.com:/usr/lib sysroot/usr
rsync -avz pi@piboard.com:/opt/vc sysroot/opt
```
* Tải sysroot
```sh
wget https://raw.githubusercontent.com/riscv/riscv-poky/master/scripts/sysroot-relativelinks.py
chmod +x sysroot-relativelinks.py
sudo apt-get install python -y
./sysroot-relativelinks.py sysroot
```

## 5. Tải về Qt – [Co]
* Tải trình biên dịch [Qt](http://download.qt.io/official_releases/qt/5.12/5.12.3/single/qt-everywhere-src-5.12.3.tar.xz)
* Tải [QtCreator](http://download.qt.io/official_releases/qt/5.12/5.12.3/qt-opensource-linux-x64-5.12.3.run)
* Copy file vừa tải về thư mục **~/raspi**
* Giải nén:
```sh
tar xvf qt-everywhere-src-5.12.3.tar.xz
cd qt-everywhere-src-5.12.3
```

## 6. Cấu hình Qt Everywhere biên dịch chéo cho Pi  – [Co]

* Cấu hình qmake
```sh
gedit qtbase/mkspecs/devices/linux-rasp-pi-g++/qmake.conf
```
Tìm và thay thế:

Thay thế "**-lEGL**" --> "**-lbrcmEGL**"

Thay thế "**-LGLESv2**" --> "**-lbrcmGLESv2**"

* Chạy **./configure**

Trong thư mục qt-everywhere-src-5.12.3

Chú ý:

	* Qt5.12: thông số **-device** là **-device linux-rasp-pi3-g++**
	* Qt5.12.2 đến 5.12.5:	thông số **-device** là **-device linux-rasp-pi-g++** 

Ví dụ bản 5.12.3

```
./configure -release -opengl es2 -device linux-rasp-pi-g++ -device-option CROSS_COMPILE=$HOME/raspi/tools/arm-bcm2708/gcc-linaro-arm-linux-gnueabihf-raspbian-x64/bin/arm-linux-gnueabihf- -sysroot ~/raspi/sysroot -opensource -confirm-license -skip qtwayland -skip qtlocation -skip qtscript -make libs -prefix /usr/local/qt5pi -extprefix ~/raspi/qt5pi -hostprefix ~/raspi/qt5 -no-use-gold-linker -v -no-gbm
```

* Thông báo thành công trên Terminal:
```
Build options:
......
QPA backends:
  ......
  EGLFS .................................. yes
  EGLFS details:
    ......
    EGLFS Raspberry Pi ................... yes
    ......
```
## 7. Make và make install – [Co]

Bước này sẽ kéo dài ít nhất 2h đồng hồ, tùy vào cấu hình máy

* Máy cấu hình yếu: 
```sh
make
```
* Máy cấu hình mạnh (nếu dùng cho máy yếu sẽ gây treo hoặc tự tắt máy)
```sh
make -j 4
```
* Cài đặt
```sh
make install
cd ..
rsync -avz qt5pi pi@piboard.com:/usr/local
```

## 8. Thiết lập Qt Creator để biên dịch chéo cho Pi – [Co]

* Cài đặt QtCreator
```sh
sudo chmod +x qt-opensource-linux-x64-5.12.3.run
./qt-opensource-linux-x64-5.12.3.run
```
Bỏ các lựa chọn Android và Script

* SSH sang Pi để thiết lập SSH Key 
```sh
ssh-keygen
Enter
Enter
ls ~/.ssh
```
Nhìn thấy có 2 file **id_rsa** và **id_rsa.pub**
```sh
ssh-copy-id pi@piboard.com
```
* Dùng **FTP Client** (**FireZilla**), copy 2 file **id_rsa** và **id_rsa.pub** từ Pi sang Co

(Cài FireZilla, nhập ip, username và password của Pi vào để kết nối FTP)

* Cấu hình QtCreator
	* Tools -> Options -> Devices section -> Devices tab -> Add a new "Generic Linux Device".
	
		Name: Raspberry Pi

		Network: piboard.com

		Key: id_rsa

		Nhấn Test, nếu kết nối được là thành công

	* Kits section: 
	
		**Compilers tab:**
		
			Add GCC C: GCC (Raspberry Pi)
			
			Path:
			
			~/raspi/tools/arm-bcm2708/gcc-linaro-arm-linux-gnueabihf-raspbian-x64/arm-linux-gnueabihf/bin/gcc

			Add GCC C++: GCC (Raspberry Pi)
			
			Path:
			
			~/raspi/tools/arm-bcm2708/gcc-linaro-arm-linux-gnueabihf-raspbian-x64/arm-linux-gnueabihf/bin/g++

		**Debuggers tab**
		
			Add: GDB (Raspberry Pi)
			
			Path:
			
			~/raspi/tools/arm-bcm2708/arm-rpi-4.9.3-linux-gnueabihf/bin/arm-linux-gnueabihf-gdb

		**Qt Versions tab**
		
			Add a new version: Qt 5.12 (Raspberry Pi)
			
			Path:
			
			~/raspi/qt5/bin/qmake

		**Kits tab**
			Add a new kit
			
			Thiết lập tên và icon tùy ý (thường là Raspberry Pi)
			
			Các thiết lập quan trọng:			

				Device type: Generic Linux Device
				Device: Raspberry Pi (defaut for Generic Linux)
				Sysroot: ~/raspi/sysroot
				Compiler C: GCC (Raspberry Pi)
				Compiler C++: GCC (Raspberry Pi)
				Debugger: GDB (Raspberry Pi)
				Qt version: Qt 5.12 (Raspberry Pi)


## Phụ lục 1: Sửa lỗi [Pi]
```sh
sudo mv /usr/lib/arm-linux-gnueabihf/libGLESv2.so.2 /usr/lib/arm-linux-gnueabihf/libGLESv2.so.2.bak

sudo ln -s /opt/vc/lib/libbrcmGLESv2.so /usr/lib/arm-linux-gnueabihf/libGLESv2.so.2

sudo mv /usr/lib/arm-linux-gnueabihf/libEGL.so.1 /usr/lib/arm-linux-gnueabihf/libEGL.so.1.bak

sudo ln -s /opt/vc/lib/libEGL.so /usr/lib/arm-linux-gnueabihf/libEGL.so.1

sudo ln -s /opt/vc/lib/libbrcmEGL.so /opt/vc/lib/libEGL.so

sudo ln -s /opt/vc/lib/libbrcmGLESv2.so /opt/vc/lib/libGLESv2.so

sudo ln -s /opt/vc/lib/libEGL.so /opt/vc/lib/libEGL.so.1

sudo ln -s /opt/vc/lib/libGLESv2.so /opt/vc/lib/libGLESv2.so.2

sudo nano ~/.profile
```

```
# physical display properties in mm
export QT_QPA_EGLFS_PHYSICAL_WIDTH=156
export QT_QPA_EGLFS_PHYSICAL_HEIGHT=86
```
```sh
source ~/.profile
```

## Phụ lục 2: Tải Font chữ [Pi]

Từ bản Qt5, Qt không cung cấp font cho thiết bị nhúng

Do đó cần tải thủ công để bổ sung và đưa vào thư mục Font

```sh
mkdir /usr/local/qt5pi/lib/fonts
```
Tải font [ở đây](https://dejavu-fonts.github.io)

Giải nén và copy vào thư mục 
```sh
/usr/local/qt5pi/lib/fonts
```

## Phụ lục 3: Cài thêm thư viện phát triển[Co]
```sh
sudo apt-get install libgl-dev
sudo apt-get install mesa-common-dev zlib1g-dev
sudo apt-get install libglu1-mesa-dev freeglut3-dev mesa-common-dev libgl1-mesa-dev
sudo apt-get install libxi-dev build-essential libdbus-1-dev libfontconfig1-dev libfreetype6-dev libx11-dev
sudo apt-get install libqt4-dev libqt4-opengl-dev
sudo apt-get install libqt5opengl5-dev
sudo apt install libncurses5
```
Kết luận
-------
* Đã được kiểm chứng thành công trên 
	* Ubuntu 19.04
	* Qt5.12.3 
	* Raspberry Pi 3B+ 
	* Màn hình LCD 7 Inch IPS CapTouch
* Bản hướng dẫn là "As Is", miễn trừ mọi trách nhiệm 
