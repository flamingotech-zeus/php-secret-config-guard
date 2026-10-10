---
name: php-secret-config-guard
description: 在 PHP（宝塔/Nginx）项目里安全存放含密钥的配置文件，并规避「注释里的 PHP 结束标记会截断整个文件」这个致命陷阱。当需要把 AccessKey / API Key / 短信凭证 / 数据库口令写进服务器网站目录，或调试「所有 PHP 接口同时返回源码片段」「Parse error: Unclosed '{'」时使用。
agent_created: true
---

# PHP 项目密钥配置安全守护

两个必须同时处理的问题：**配置被直接下载** 和 **注释里的 PHP 结束标记**。
两者都会造成"看起来像核心逻辑坏了、实际是一个字符"的排查事故。

## 一、问题一：含密钥的配置放在网站目录里会被直接下载

宝塔 / Nginx 默认把 `.json` 当静态文件返回。若配置叫 `auth.config.json` 放在 `api/` 下，任何人访问
`https://你的域名/api/auth.config.json` 就能完整下载到 AccessKeySecret / 大模型 Key / 七牛 AK。

**做法：配置一律用 `.php` 后缀，首行写 exit 保护。**

```
<?php exit; /* 本文件含密钥，禁止直接访问。不要删除本行，也不要改回 .json */ ?>
{ "access_key": "...", "api_key": "..." }
```

PHP 引擎一进来就退出 → 直接访问只得到空白页（HTTP 200，空 body）；
程序读文件时跳过首行 → 照常拿到 JSON。

### 读取函数（跳过保护头）

```php
function sec_json_path($base) {           // $base 不含扩展名，如 'auth.config'
    $php = __DIR__ . '/' . $base . '.php';
    if (is_file($php)) return $php;
    $jsn = __DIR__ . '/' . $base . '.json';   // 兼容旧文件
    if (is_file($jsn)) return $jsn;
    return $php;                              // 默认写更安全的 .php
}

function sec_json_read($base) {
    $p = sec_json_path($base);
    if (!is_file($p)) return null;
    $raw = (string)@file_get_contents($p);
    $i = strpos($raw, '{');                   // 关键：从第一个 { 开始
    if ($i === false) return null;
    $d = json_decode(substr($raw, $i), true);
    return is_array($d) ? $d : null;
}
```

### 写入函数（自动迁移，只留一份）

```php
function sec_json_write($base, $data) {
    $php = __DIR__ . '/' . $base . '.php';
    $jsn = __DIR__ . '/' . $base . '.json';
    $json = json_encode($data, JSON_UNESCAPED_UNICODE | JSON_UNESCAPED_SLASHES | JSON_PRETTY_PRINT);
    if ($json === false) return false;
    $body = "<?php exit; /* 本文件含密钥，禁止直接访问。不要删除本行，也不要改回 .json */ ?>\n" . $json . "\n";
    $tmp = $php . '.tmp';
    if (@file_put_contents($tmp, $body, LOCK_EX) === false) return false;
    if (!@rename($tmp, $php)) { @unlink($tmp); return false; }
    @chmod($php, 0600);
    if (is_file($jsn)) @unlink($jsn);         // 迁移：避免两份不一致
    return true;
}
```

> 上面字符串里的结束标记是**字符串内容**，安全。危险的是把它写进**注释**（见第二节）。

### Nginx 再加一道锁（可选，多一层保险）

```nginx
location ~* ^/api/.*\.(json|log|data)$ { deny all; }
location ~* ^/api/.*\.tmp$              { deny all; }
```

## 二、问题二（致命）：注释里的 PHP 结束标记会截断整个文件

PHP 的词法分析**在 `//` 和 `/* */` 注释里也会把结束标记当成代码块结束**，即使它出现在注释中。
于是注释后面的**所有函数定义都会被当作 HTML 原样输出**。

症状（极其容易被误判为逻辑改坏了）：

- 所有 PHP 接口同时失败，响应体里混着源码片段和中文注释
- `Parse error: Unclosed '{' on line N` —— N 就是那条注释所在行
- 接口返回的 JSON 前面多了一段文字，`JSON.parse` 失败 → 前端一片空白

**规避规则：注释里永远不要原样写出结束标记。** 需要举例时写成文字描述：

```php
// 做法：配置文件用 .php 后缀，首行在 PHP 开始标记后立刻写 exit 语句。
// ★ 注意：注释里绝对不能原样写出 PHP 结束标记（问号大于号）。
//    需要举例时请写成「PHP 结束标记」这种文字描述。
```

## 三、上线前自检（建议写成脚本常跑）

1. **扫描注释陷阱**：遍历项目 `*.php`，逐行检查 trim 后以 `//` 或 `*` 开头且包含结束标记的行 → 报错。
   （字符串内的结束标记是安全的，只查注释行即可。）
2. **验证配置不可下载**：`curl -s 站点/api/auth.config.php` → 输出必须为空。
3. **检查运行时残留**：`api/` 下不应出现 `*.devcode.log`（明文验证码）、`auth.data.json`（会话）、明文 `*.config.json`。
4. **文件权限**：配置 `0600`，目录 `755`、属主 `www`。
5. **验证清单**：直接访问配置文件为空白页；未登录访问烧额度的接口返回 401；正常功能不受影响。

## 四、适用与不适用

- 适用：宝塔 + Nginx + PHP-FPM 的普通站点，静态前端 + PHP 接口结构，无框架或轻框架。
- 不适用：把配置放在**网站根目录之外**（如 `/www/server/conf/`）的部署 —— 那种情况本身已安全，
  用普通 `.json` 即可，但仍需注意第二节的注释陷阱（它的危害与放哪无关）。
- 折中判断：如果配置必须在站点目录内，本方案比"移到根目录外"更省事且同样有效（无需改路径、无需处理权限）。
