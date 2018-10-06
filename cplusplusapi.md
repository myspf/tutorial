# DolphinDB C++ API
本教程介绍在linux环境下，如何使用DolphinDB提供的C++ API进行应用开发。
### 1、环境需求
* linux 编程环境；  
* g++ 6.2编译器；（由于libDolphinDBAPI.so是由g++6.2编译的，为了保证ABI兼容，建议使用该版本的编译器）
 
### 2、编译工程
#### 2.1 下载so文件和头文件
从XXX下载api-cplusplus，包括bin和include，如下：
> bin (libDolphinDBAPI.so)
  include (DolphinDB.h  Exceptions.h  SmartPointer.h  SysIO.h  Types.h  Util.h)
#### 2.3 编写main.cpp文件
在与bin和include平级目录创建目录project，进入project并创建文件main.cpp，内容如下：
```
#include "DolphinDB.h"
#include "Util.h"
#include <iostream>
#include <string>
using namespace dolphindb;
using namespace std;

int main(int argc, char *argv[]){
    DBConnection conn;
    bool ret = conn.connect("localhost", 8080);
    if(!ret){
        cout<<"Failed to connect to the server"<<endl;
        return 0;
    }
    ConstantSP vector = conn.run("\`IBM\`GOOG\`YHOO");
    int size = vector->rows();
    for(int i=0; i<size; ++i)
        cout<<vector->getString(i)<<endl;
    return 0;
}
```
#### 2.3 编译
g++ 编译命令如下：
> g++ main.cpp -std=c++11 -DLINUX -DLOGGING_LEVEL_2 -O2 -I../include -lDolphinDBAPI -lssl  -lpthread -luuid -L../bin  -Wl,-rpath ../bin/ -o main

### 3、登录dolphindb


### 4、执行dolphindb script
#### 4.1 创建连接
#### 4.2 执行脚本
#### 4.3 不同类型的返回值
##### 4.3.1 vector
##### 4.3.2 set
##### 4.3.3 matrix
##### 4.3.4 dictionary
##### 4.3.5 table

### 6、调用dolphindb 内置函数











 
