
git 使用 ss 代理
==================
git config --global http.proxy socks5://127.0.0.1:1080
git config --global https.proxy socks5://127.0.0.1:1080
git config --global http.sslVerify false

git clone git://g.csail.mit.edu/6.824-golabs-2020 6.824

取消的方法：
git config --global --unset http.proxy 
git config --global --unset https.proxy











