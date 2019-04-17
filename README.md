# kaldi-asr-ali-version
- 阿里开源的语音识别，基于kaldi构建，本项目用于记录在复现过程中遇到的问题及细节
- 基于`Dockerfile`安装、编译；

## 1.Dockerfile 建立 kaldi的docker镜像

- 新建文件夹

```
$ mkdir kaldi-ali
$ cd kaldi-ali
$ touch Dockerfile
```

- 编写Dockerfile

```
$ vim Dockerfile
```
> i 切换到写状态
> 编写代码如下

```
FROM nvidia/cuda:9.1-devel-ubuntu16.04

MAINTAINER sih4sing5hong5

ENV CPU_CORE 4

RUN \
  apt-get update -qq && \
  apt-get install -y \
    git bzip2 wget sox \
    g++ make python python3 \
    zlib1g-dev automake autoconf libtool subversion \
    libatlas-base-dev unzip


WORKDIR /usr/local/
# Use the newest kaldi version
RUN git clone https://github.com/kaldi-asr/kaldi.git kaldi-trunk --origin golden 


WORKDIR /usr/local/kaldi-trunk/tools
RUN extras/install_mkl.sh
RUN extras/check_dependencies.sh
RUN make -j $CPU_CORE

WORKDIR /usr/local/kaldi-trunk/src
RUN ./configure && make depend -j $CPU_CORE && make -j $CPU_CORE
```
- build 镜像
```
$ touch Dockerfile
$ docker build -t docker.ailong.com/phecda/kaldi:ali-version
```
> 一直跑到docker镜像docker.ailong.com/phecda/kaldi:ali-version建立完成

## 2.使用阿里的patch

- 启docker

```
$ nvidia-docker run -it docker.ailongma.com/phecda/kaldi:ali-version /bin/bash
```

- 切换kaldi版本，加载阿里开源的patch，添加 Git 账户邮箱和用户名，应用阿里的patch

```
[root@ebdda6ff5aba:/usr/local/kaldi-trunk/src]$ cd ..
[root@ebdda6ff5aba:/usr/local/kaldi-trunk]$ git clone https://github.com/alibaba/Alibaba-MIT-Speech
[root@ebdda6ff5aba:/usr/local/kaldi-trunk]$ git checkout 04b1f7d6658bc035df93d53cb424edc127fab819
[root@ebdda6ff5aba:/usr/local/kaldi-trunk]$ git apply --check Alibaba-MIT-Speech/Alibaba_MIT_Speech_DFSMN.patch
[root@ebdda6ff5aba:/usr/local/kaldi-trunk]$ git config --global user.email "git_user_email"
[root@ebdda6ff5aba:/usr/local/kaldi-trunk]$ git config --global user.name "git_user_name"
[root@ebdda6ff5aba:/usr/local/kaldi-trunk]$ git am --signoff < Alibaba-MIT-Speech/Alibaba_MIT_Speech_DFSMN.patch
```
- 安装kaldi依赖的包

```
[root@ebdda6ff5aba:/usr/local/kaldi-trunk]$ cd tools
[root@ebdda6ff5aba:/usr/local/kaldi-trunk/tools]$ extras/check_dependencies.sh
```
> 出现`extras/check_dependencies.sh: all OK.`时表示安装完成

- 编译

```
$ make -j 6
```

> 6 表示使用的线程数，可根据实际情况设定
> 出现`All done OK.`表示编译成功

- 配置

```
[root@ebdda6ff5aba:/usr/local/kaldi-trunk/tools]$ cd ../src/
[root@ebdda6ff5aba:/usr/local/kaldi-trunk/src]$ ./configure --shared
```

> 出现`SUCCESS`表示配置完成
> 后边还会出现`To compile: make clean -j; make depend -j; make -j`为接下来要进行的操作

- 继续编译

```
[root@ebdda6ff5aba:/usr/local/kaldi-trunk/src]$ make clean -j 6
[root@ebdda6ff5aba:/usr/local/kaldi-trunk/src]$ make depend -j 6
[root@ebdda6ff5aba:/usr/local/kaldi-trunk/src]$ make -j 6
[root@ebdda6ff5aba:/usr/local/kaldi-trunk/src]$ make ext
```
> 前两个过程执行后不报错就继续，后两个过程要出现明显的完成字样，如下
> `Done`
> `Done`

- demo测试是否安装成功

```
[root@ebdda6ff5aba:/usr/local/kaldi-trunk/src]$ cd ../egs/yesno/s5/
[root@ebdda6ff5aba:/usr/local/kaldi-trunk/egs/yesno/s5]$ sh run.sh
```

> 出现`%WER 0.00 [ 0 / 232, 0 ins, 0 del, 0 sub ] exp/mono0a/decode_test_yesno/wer_10_0.0`表示成功