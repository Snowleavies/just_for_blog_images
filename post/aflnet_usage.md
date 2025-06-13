使用aflnet對openssl 3.5 進行TLS協定模糊測試
===

前言
---
最近開始研究後量子加密的部分，旁通道是公司另一組負責的，所以我們只搞協定的部分了。) 哭  
剛好openssl跟新3.5.0版本，支援MLKEM金鑰演算法，因此打算使用aflnet進行TLS模糊測試，網路上的測試範例都是專案裡的RTSP伺服器，因此寫這一篇，剛好當作第一篇的練習。

---

什麼是aflnet
---
aflnet是基於afl開發針對協定的模糊測試框架，使用基於覆蓋率的分析，會分析回傳封包以及插樁函式，選擇更好的種子以及變異，方式但afl主要用於函式庫，aflnet用於協定，協定的測試作用於整個系統和狀態機，不太一樣。  

什麼是模糊測試
---
模糊測試是一種動態程式測試方式，透過不停變異輸入值，找出程式執行時的記憶體錯誤，像是bufferoverflow，可能導致crash以及hang(掛住)，自從tls 1.2 的heartbeat漏洞後，模糊測試就開始變得逐漸重要。

使用環境
---
被測試伺服器(System Under Test): ubuntu 22.04  
攻擊機: ubuntu 22.04

#### 如果你電腦CPU夠強，可以裝在同一台，但我是用筆電，所以沒辦法

第一步 - 安裝aflnet (攻擊機) 
===
參考[**教學文件**](https://blog.csdn.net/weixin_45100742/article/details/139895669)  

- 安裝依賴項
~~~sh
sudo apt-get update
sudo apt-get install net-tools wireshark clang-12 git make  gcc graphviz-dev libcap-dev
#跳出頁面選擇yes

sudo usermode -aG wireshark $USER
~~~

- 安裝aflnet
~~~sh
cd ~
git clone https://github.com/aflnet/aflnet.git
make clean all
cd llvm_mode
make 
~~~

- 加入環境變數
~~~sh
cd ~
nano ~/.bashrc

## 加在最底下
export AFLNET=$HOME/aflnet
export WORKDIR=$HOME
export PATH=$PATH:$AFLNET
export AFL_PATH=$AFLNET
# 完成後ctrl+x => y => enter

source ~/.bashrc

#驗證安裝
afl-fuzz --help
~~~
![](https://github.com/Snowleavies/just_for_blog_images/blob/main/images/1/7.png?raw=true)

第二步 - 插樁/編譯 openssl 3.5.0 (攻擊機) 
===

~~~sh
cd ~
wget https://github.com/openssl/openssl/releases/download/openssl-3.5.0/openssl-3.5.0.tar.gz
tar zxf openssl-3.5.0.tar.gz
cd openssl-3.5.0

###將編譯器換成afl，在編譯時可以插樁
./Configure --prefix=/opt '-Wl,-rpath,$(LIBRPATH)'
export CC=afl-gcc
export CXX=afl-g++
make 
sudo make install
cd apps
./openssl version

sudo mkdir /opt/openssl-3.5.0/certs 
cd /opt/openssl-3.5.0/certs 
openssl req -x509 -newkey rsa:2048 -keyout key.pem -out cert.pem -days 365 -nodes9 
~~~

第三步 - 編譯/安裝 openssl 3.5.0 (SUT) 
===
- 這裡就不多講，別人寫的教學文件超級詳細，請參照[**教學文件**](https://www.linode.com/docs/guides/post-quantum-encryption-nginx-ubuntu2404/)
- 但仍有部分需要修改的地方，在這裡說明一下  

- 原本的nginx conf會報錯 
![](https://github.com/Snowleavies/just_for_blog_images/blob/main/images/1/3.png?raw=true)
- 如果想要使用後量子簽章演算法
![](https://github.com/Snowleavies/just_for_blog_images/blob/main/images/1/2.png?raw=true)

第四步 - 收集語料庫種子
===

要收集語料庫，首先要收集封包
語料庫種子並不式越多越好，而是要能夠觸發新的函式路徑

- 確認SUT開啟nginx server
![]()
- 打開wireshark(SUT)
![]()

- openssl連線、收集封包(攻擊機)
![]()

- 製作語料庫
![](https://github.com/Snowleavies/just_for_blog_images/blob/main/images/1/1.png?raw=true)

第五步 - 進行模糊測試
===

- 執行aflnet (攻擊機)
~~~sh
cd /opt/openssl-3.5.0/apps/
afl-fuzz -d -i /<語料庫資料夾> -o out -N tcp://192.168.1.140/443(被測系統ip/port) -P TLS -q 3 -s 3 -E -K -R ./openssl s_server -key key.pem -cert cert.pem -accept 443 
~~~
- 可能發生的問題
![解決辦法](https://github.com/Snowleavies/just_for_blog_images/blob/main/images/1/5.png?raw=true)
- 執行成功
![執行成功](https://github.com/Snowleavies/just_for_blog_images/blob/main/images/1/4.png?raw=true)

- 接下來就要進行長時間等待，直到有hang或crash發生()

第六步 - 分析結果
===
由於尚未找到漏洞，因此無法做TLS的漏洞重現，但根據Readme文檔，可以使用afl-replay進行漏洞重現
![](https://github.com/Snowleavies/just_for_blog_images/blob/main/images/1/6.png?raw=true)

結語
===
第一篇 雖然好像很多都沒講清楚，但先這樣吧