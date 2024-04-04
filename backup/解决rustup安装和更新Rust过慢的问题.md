国内连接默认的镜像速度过慢，建议安装过的没安装过的都设置一次！！
步骤一：设置 Rustup 镜像， 修改配置 ~/.zshrc or ~/.bashrc
``` bash
export RUSTUP_DIST_SERVER="https://rsproxy.cn"
export RUSTUP_UPDATE_ROOT="https://rsproxy.cn/rustup"
```
步骤二：安装 Rust 或更新Rust
``` bash
#安装
curl --proto '=https' --tlsv1.2 -sSf https://rsproxy.cn/rustup-init.sh | sh
#更新
rustup self update
rustup update stable
```
步骤三：设置 crates.io 镜像， 修改配置 ~/.cargo/config，已支持git协议和sparse协议，>=1.68 版本建议使用 sparse-index，速度更快。
```
[source.crates-io]
replace-with = 'rsproxy-sparse'
[source.rsproxy]
registry = "https://rsproxy.cn/crates.io-index"
[source.rsproxy-sparse]
registry = "sparse+https://rsproxy.cn/index/"
[registries.rsproxy]
index = "https://rsproxy.cn/crates.io-index"
[net]
git-fetch-with-cli = true
```
参考地址：
[字节官方rust镜像](https://rsproxy.cn/#getStarted)

设置其他镜像地址步骤相同，设置成自己喜欢的就行