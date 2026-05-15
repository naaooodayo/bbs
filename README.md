#!/usr/local/bin/perl

use CGI;
use CGI::Carp qw/ fatalsToBrowser /;

require './init.cgi';

my $cgi = CGI->new;

my $NAME        = escapeText($cgi->param('NAME'));
my $ADDRESS     = escapeText($cgi->param('ADDRESS'));
my $SUBJECT     = escapeText($cgi->param('SUBJECT'));
my $MESSAGE     = escapeText($cgi->param('MESSAGE'));
my $CLIENT_TIME = $cgi->param('CLIENT_TIME');

if($NAME eq '' && $MESSAGE eq '' && $SUBJECT eq '' && $ADDRESS eq ''){
    printHTML();
}
else{
    if($NAME eq '' || $SUBJECT eq '' || $MESSAGE eq ''){
        printAlert("メールアドレス以外はすべて入力してください");
    }
    else{

        if(checkPost()){
            saveLog();
            printHTML();
        }
        else{
            printAlert("連続投稿はできません。10秒待ってください。");
        }
    }
}

sub escapeText {

    my $text = shift;

    $text =~ s/&/&amp;/g;
    $text =~ s/</&lt;/g;
    $text =~ s/>/&gt;/g;
    $text =~ s/,/&#44;/g;
    $text =~ s/\r\n/<br>/g;
    $text =~ s/\n/<br>/g;

    return $text;
}

sub checkPost {

    my $ip  = $ENV{'REMOTE_ADDR'};
    my $now = time;

    open(LOG, "$log_file") || return 1;

    my @lines = <LOG>;

    close(LOG);

    foreach my $line (reverse @lines){

        my ($post_time, $client_time, $log_ip) = split(/\t/, $line);

        if($ip eq $log_ip){

            if(($now - $post_time) < 10){
                return 0;
            }

            last;
        }
    }

    return 1;
}

sub saveLog {

    my $ip = $ENV{'REMOTE_ADDR'};

    open(LOG, ">> $log_file") || printError();

    printf LOG "%s\t%s\t%s\t%s\t%s\t%s\t%s\n", time, $CLIENT_TIME, $ip, $NAME, $ADDRESS, $SUBJECT, $MESSAGE;

    close(LOG);
}

sub getLog {

    my $ret = "";

    open(LOG, "$log_file") || printError();

    my @lines = <LOG>;

    close(LOG);

    foreach my $line (reverse @lines){

        chomp($line);

        my ($unix_time, $time_str, $ip, $name, $address, $subject, $message) = split(/\t/, $line);

        my $name_html;

        if($address eq ""){
            $name_html = "<b><font color=\"$nomail_color\">$name</font></b>";
        }
        else{
            $name_html = "<b><a href=\"mailto:$address\">$name</a></b>";
        }

        $ret .= <<"EOL";

<hr>

<font size="+1" color="$title_color">
<b>$subject</b>
</font>
<br>

名前：$name_html
$time_str
<br>

<p>$message</p>

EOL

    }

    return $ret;
}

sub getForm {

return <<"EOL";

<form method="post" action="./bbs.cgi">

<center>

<table border="3">

<tr>
<td valign="top" align="right">名前：</td>
<td align="left">
<textarea name="NAME" rows="1" cols="60" maxlength="50"></textarea>
</td>
</tr>

<tr>
<td valign="top" align="right">E-mail：</td>
<td align="left">
<textarea name="ADDRESS" rows="1" cols="60" maxlength="50"></textarea>
</td>
</tr>

<tr>
<td valign="top" align="right">タイトル：</td>
<td align="left">
<textarea name="SUBJECT" rows="3" cols="60" maxlength="50"></textarea>
</td>
</tr>

<tr>
<td valign="top" align="right">内容：</td>
<td align="left">
<textarea name="MESSAGE" rows="5" cols="60" maxlength="500"></textarea>
</td>
</tr>

</table>

<input type="hidden" name="CLIENT_TIME" id="CLIENT_TIME">

<br>

<input type="submit" value="　送信　">
<input type="reset" value="リセット">

</center>
</form>

EOL
}

sub printError {

print <<"EOL";
Content-type: text/html

<html>

<head>
<meta charset="UTF-8">
<title>ERROR</title>
</head>

<body>

<font size="6" color="FF0000">
CGIエラー
</font>
</body>
</html>
EOL

die "$!";
}

sub printAlert {

    my $message = shift;

print <<"EOL";
Content-type: text/html

<html>

<head>
<meta charset="UTF-8">
<title>WARNING</title>
</head>

<body>

<script>
alert("$message");
history.back();
</script>
</body>
</html>
EOL
}

sub printHTML {

print <<"EOL";
Content-type: text/html

<html>

<head>
<meta charset="UTF-8">
<title>$title</title>
</head>

<body bgcolor="$bg_color" link="$mail_color">

<font color="$font_color">

<center>
<h1>$title</h1>
</center>

</font>

<center>

<div id="clock"></div>

<script>

function showClock() {

    const now = new Date();
    const year = now.getFullYear();
    const month = String(now.getMonth() + 1).padStart(2, '0');
    const day = String(now.getDate()).padStart(2, '0');
    const hour = String(now.getHours()).padStart(2, '0');
    const min = String(now.getMinutes()).padStart(2, '0');
    const sec = String(now.getSeconds()).padStart(2, '0');

    const time = year + "年" + month + "月" + day + "日 " + hour + ":" + min + ":" + sec;

    document.getElementById("clock").innerHTML = time;
    document.getElementById("CLIENT_TIME").value = time;
}

setInterval(showClock, 1000);

showClock();

</script>
</center>

@{[getForm()]}
@{[getLog()]}

<hr>

<font size="2">

<div align="right">
$title
</div>
</font>
</body>
</html>

EOL
}
