在默认的情况下,PHP会把全部的会话数据保存在服务器上的文本文件里面,这些文件通常都是保存在服务器上的临时目录里边。
那为什么我们要把session会话保存在数据库中呢?
1. 主要原因:提高系统的安全性。在共享服务器上，在没有进行特别的设置，所有的网站站点都会使用同一个临时目录，这意味着数十个程序都在同一个位置对文件进行读写操作。不仅速度下降了，而且别人也有可能窃取到我的站点的用户数据。
2. 把会话数据保存到数据库还可以更方便的搜索web站点会话的更多信息，我们可以查询活动会话的数量（同时在线的用户量），还可以对会话数据进行备份
3. 假如我的站点同时运行于多个服务器，那么某个用户在一个会话过程中，可能会对不同的服务器发送多个请求，但是会话数据如果保存在某一个服务器上，那么其他服务器就不能使用到这些会话数据。假如我的某一台服务器仅仅是数据库的角色，那你把会话数据全保存在数据库中，不是很方便么？

# 1、创建会话表
由于session数据是保存在服务器上的,而在客户端中保存的是一个索引(sessionID),这个索引对应于服务器上某一条session数据。因此该表必须包含的两个字段是id、data
还有就是会话会有过期时间，所以在这里还有个字段就是last_accessed，这里我把改表建在test数据库下：
```
CREATE TABLE sessions(
    id CHAR(32) NOT NULL,
    data TEXT,
    last_accessed TIMESTAMP NOT NULL,
    PRIMARY KEY(id)
);
```
PS:如果程序需要在会话中保存大量的数据，则data字段可能就需要定义为MEDIUMTEXT或LONGTEXT类型了。
# 2、定义会话函数
这里我们主要有两个步骤:
1. 定义与数据库交互的函数
2. 是php能使用这些自定义函数

在第二步中，是通过调用函数session_set_save_handler()来完成的，调用它需要6个参数，分别是open（启动会话）、close（关闭会话）、read（读取会话）、write（写入会话）、destroy（销毁会话）、clean（垃圾回收）。

我们新建php文件session.inc.php，代码如下:
```php
<?php

$sdbc = null;  //数据库连接句柄，在后面的函数里面让它成为全局变量

//启动会话
function open_session()
{
    global $sdbc;      //使用全局的$sdbc
    $sdbc = mysqli_connect('localhost', 'root', 'lsgogroup', 'test');     //数据库 test
    if (!$sdbc) {
        return false;
    }
    return true;
}

//关闭会话
function close_session()
{
    global $sdbc;
    return mysqli_close($sdbc);
}

//读取会话数据
function read_session($sid)
{
    global $sdbc;
    $sql = sprintf("SELECT data FROM sessions WHERE id='%s'", mysqli_real_escape_string($sdbc, $sid));
    $res = mysqli_query($sdbc, $sql);
    if (mysqli_num_rows($res) == 1) {
        list($data) = mysqli_fetch_array($res, MYSQLI_NUM);
        return $data;
    } else {
        return '';
    }
}

//写入会话数据
function write_session($sid, $data)
{
    global $sdbc;
    $sql = sprintf("INSERT INTO sessions(id,data,last_accessed) VALUES('%s','%s','%s')", mysqli_real_escape_string($sdbc, $sid), mysqli_real_escape_string($sdbc, $data), date("Y-m-d H:i:s", time()));
    $res = mysqli_query($sdbc, $sql);
    if (!$res) {
        return false;
    }
    return true;
}

//销毁会话数据
function destroy_session($sid)
{
    global $sdbc;
    $sql = sprintf("DELETE FROM sessions WHERE id='%s'", mysqli_real_escape_string($sdbc, $sid));
    $res = mysqli_query($sdbc, $sql);
    $_SESSION = array();
    if (!mysqli_affected_rows($sdbc) == 0) {
        return false;
    }
    return true;
}

//执行垃圾回收（删除旧的会话数据）
function clean_session($expire)
{
    global $sdbc;
    $sql = sprintf("DELETE FROM sessions WHERE DATE_ADD(last_accessed,INTERVAL %d SECOND)<NOW()", (int)$expire);
    $res = mysqli_query($sdbc, $sql);
    if (!$res) {
        return false;
    }
    return true;
}

//告诉PHP使用会话处理函数
session_set_save_handler('open_session', 'close_session', 'read_session', 'write_session', 'destroy_session', 'clean_session');

//启动会话，该函数必须在session_set_save_handler()函数后调用，不然我们所定义的函数就没法起作用了。
session_start();

//由于该文件被包含在需要使用会话的php文件里面，因此不会为其添加PHP结束标签
```
PS:
1. 处理“读取”函数外，其他函数必须返回一个布尔值，“读取”函数必须返回一个字符串。
2. 每次会话启动时，“打开”和“读取”函数将会立即被调用。当“读取”函数被调用的时候，可能会发生垃圾回收过程。
3. 当脚本结束时，“写入”函数就会被调用，然后就是“关闭”函数，除非会话被销毁了，而这种情况下，“写入”函数不会被调用。但是，在“关闭”函数之后，“销毁”函数将会被调用。
4. session_set_save_handler()函数参数顺序不能更改，因此它们一一对应open、close、read、。。。。。
5. 会话数据最后将会以数据序列化的方式保存在数据库中。
# 3、使用新会话处理程序
使用新会话处理程序只是调用session_set_save_handler()函数，使我们的自定义函数能够被自动调用而已。其他关于会话的操作都没有发生变化(以前怎么用现在怎么用，我们的函数与会在后台自动被调用),
包括在会话中存储数据，访问保存的会话数据以及销毁数据。

由于我们知道，php会在脚本执行完之后自动关闭数据库的所有连接，而同时会话函数会尝试向数据库写入数据并关闭连接。这样一来，会话数据没法写入数据库，并且出现一大堆错误，
例如write_session()、close_session()函数中都有用到数据库的连接。
为了避免以上说的问题，我们<font color=red size=6>脚本执行完之前调用session_write_close()函数，他就会调用"写入"函数和"关闭"函数</font>，而此时数据库连接还是存在的。
PS:在使用header()函数重定向浏览器之前也应该调用session_write_close()函数，假如有数据库的操作时。

<font color=red size=6>PS:经测试不调用session_write_close()函数，不会调用session的write_session()方法</font>