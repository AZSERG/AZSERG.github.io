---
title: "Masscanning the web: from bug bounty to profit"
author: rjones
date: 2021-11-07
categories:
  - blog
tags:
  - bug bounty
  - recon
  - network pentesting
  - masscan
---

Amass is cool, but what if we just need more endpoints to hack?

![](/assets/2021-11-01-scanning-the-web/bb.jpg)

## The idea

1. Use `masscan` to scan all common web ports on a subnet. 
2. Parse the results into a gowitness readable format.
3. Pass the results into gowitness to screenshot every IP address/port identified.
4. Obtain gowitness screenshots from identified web pages.
5. Verify additional web server information through shodan.io

## Finding Subnets

![](/assets/2021-11-01-scanning-the-web/heis.png)

The [Hurricane Electric BGP Toolkit](https://bgp.he.net/) will provide all subnets for any organization you search. 
For this reason, the Hurricane Electric BGP Toolkit is a highly used tool in bug bounty programs.

## Masscanning multiple subnets using Bash

```bash
#!/bin/bash

# Add subnets here
declare -a varname
subnet_array=( 
# 0.0.0.0/0
)

for val in ${subnet_array[@]}; do
   masscan $val -p80,443,8080,8443,81, 4444, 4443, 8888 >> massscan_open_ports
done
```

This bash script includes an array of subnets that will be used for scanning.
The script will loop through every subnet and port scan the endpoints.

![](/assets/2021-11-01-scanning-the-web/masscan_output.PNG)

## Parsing masscan output using Python

```python
unparsed_file = open('unparsed.txt', 'r') 
parsedfile = open('parsed.txt', 'w') 
Lines = unparsed_file.readlines() 

# Strips the newline character 
for line in Lines: 
    temp_split = line.split()
    
    unparsed_port = temp_split[3]
    port = unparsed_port.replace('/tcp', '')
    
    ip = temp_split[5]
    
    # For every port try both HTTP and HTTPS
    #HTTP
    if port == "443" or port == "80":
    	http_formatted_line = ("http://" + ip)
    else:
        http_formatted_line = ("http://" + ip + ":" + port)

    #HTTPS
    if port == "443" or port == "80":
    	https_formatted_line = ("https://" + ip)
    else:
        https_formatted_line = ("https://" + ip + ":" + port)

    #Output
    parsedfile.writelines(http_formatted_line)
    parsedfile.writelines("\n")
    parsedfile.writelines(https_formatted_line)
    parsedfile.writelines("\n")
```

Gowitness requires endpoints to be in the HTTP URI format. 
This python script parses through the masscan output and properly formats the output into an HTTP URI readable format.
Example output could include the following formats.
```
* http://0.0.0.0
* https://0.0.0.0
* http://0.0.0.0:8081
* https://0.0.0.0:8081
```

## Gowitness

![](/assets/2021-11-01-scanning-the-web/gowitness_output.PNG)

Using Gowitness the parsed file can be supplied using the `-f` argument. 
Gowitness will screenshot all endpoints found from masscan.
Happy hacking!

## Shodan

![](/assets/2021-11-01-scanning-the-web/shodan.PNG)

Shodan can be used to provide additional information on the identified web endpoints. It crawls the internet for publicly available endpoints, which can be used for verification purposes when performing this recon assessment.

## Takeaways

![](/assets/2021-11-01-scanning-the-web/sandwhich.png)


## FAQ

### Why Masscan and not NMAP?
Masscan is fast! From one machine, it has the potential to scan the entite internet in under 5 minutes at a rate of 10 million packets per second. Masscan can also be exported to an XML format, which can be read by NMAP for OS and version detection.

### Why Gowitness?
During testing I had eyewitness break multiple times. The gowitness executable had no issues, which is why I chose to go with it.

## Resources

* [Gowitness](https://github.com/sensepost/gowitness)
* [Masscan](https://github.com/robertdavidgraham/masscan)
* [Shodan.io](https://www.shodan.io)






